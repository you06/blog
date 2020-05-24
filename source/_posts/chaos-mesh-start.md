title: Chaos Mesh・Kubernetesのカオス上の舞う
route: chaos-mesh-start
date: 2020-02-16T12:24:33.279Z
tags: 
    - 技术杂谈
    - 日本語
--------------------------
このブログを改善してくれた[genboku](https://twitter.com/SSSSSSSHHHHHH4)さんに感謝しました。

---

[`Chaos Mesh`](https://github.com/pingcap/chaos-mesh)はカオスエンジニアリングのためのテストツールです。

コンテナ化されたアプリケーションに対してカオスなテストを行うことができます。

そして、クラウドネイティブに準じたオープンソースなツールキットです。

<!-- more -->

## 基礎知識

### カオステストが必要な理由

カオスはプロダクション環境で頻繁に発生します、マシン障害、ネットワーク障害など。それらの障害=故障に対応するために、多くの試行錯誤を行って来ました。その結果生まれたのがカオステストです。

単体テストはソースコードのモジュールごとの安定性を保障するために行います。同様にカオステストは 故障時の障害対応を安定して行えることを保証するために 行います。

[マーフィーの法則](https://ja.wikipedia.org/wiki/%E3%83%9E%E3%83%BC%E3%83%95%E3%82%A3%E3%83%BC%E3%81%AE%E6%B3%95%E5%89%87)によれば、**「失敗する余地があるなら、失敗します」**。

システムがより複雑でより規模が大きくなるほど、潜在的なバグも多くなります。プロダクション運用環境と同じサイズの開発環境であったとしてもバグを見逃すことはまだあるでしょう。しかし、コストを節約するために、通常の開発環境は(プロダクション運用環境よりも)小さいです。バグはさらに見つけにくくなります。

例えばハードドライブが損傷する確率は1時間あたり0.0001％であると仮定しましょう。100台のハードドライブのクラスターが10,000時間実行してようやく1台のハードディスクが破損する可能性がでてきます。しかし、開発環境ではその可能性は非常に低いことはいうまでもありません。

カオステストはこの障害状況をシミュレートすることです。

### クラウドネイティブな理由

#### クラウドネイティブに至った理由

カオスの実装方法はたくさんあります。時間のカオスをシミュレートするためにシステム時間を調整したり、ネットワークのカオスをシミュレートするために`iptables`を使用したファイアウォールのセットアップを行ったり...その中で自分の経験からクラウドネイティブを選んだ理由を説明します。

`PingCAP`では、非常に早い段階でカオステストを開始しました。

最初は、`SSH`経由でマシンにカオスを設定していました。

![sshで構成されたカオス](https://user-images.githubusercontent.com/9587680/74592246-59144200-505a-11ea-8dd5-e6fd9b561e7a.png)

あの時、僕は`TiDB`用のカオステストツールを作成しました。

[`tidb-ansible`](github.com/pingcap/tidb-ansible/) を使用してTiDBをデプロイし、データベースに接続するロードプログラムを実行しながら、さらに同時にカオス操作を実行するツールです。

大事な問題は、このテストフレームワークにもバグがあることでした。例えば、`tidb-ansible`を使用した`TiDB`のデプロイに失敗したり、`iptables`のルール削除に失敗するなど（そしてそれは次のテストの環境汚染を引き起こします）。

他にも様々な問題がありました：

* リソース使用率が低い
* 問題が発生した場合、プログラムの実行状態を記述するために、次のテストは停止する必要がありました
* ログ収集を行う必要がありました
* 複数のクラスターを同時にテストできましたが、新しいクラスタを追加するたびに、テスト環境を手動で構成する必要がありました
* ...

さらに上記の問題に共通する事項として、このテストスイートを使用するためには、多くの場合、手動による介入が必要でした。

#### クラウドネイティブのアドバンテージ

* `Kubernetes`はリソースを適切に管理し、テストに標準環境を提供します
* [`TiDB Operator`](github.com/pingcap/tidb-operator/)は、`TiDB`クラスタを管理できます
* `Chaos Mesh`、`TiDB Operator`などのユニットに分割できます

#### クラウドネイティブの欠陥

* ある一部の障害のシミュレートできません。例えば、**電源切断**など。

### `Chaos Mesh`を試す

カオスエンジニアリングによるテストがなぜ必要か。

そしてChaos Meshがどうしてクラウドネイティブな作りをしているのか。

わかったところで公式ドキュメント：[https://github.com/pingcap/chaos-mesh#deploy-chaos-mesh](https://github.com/pingcap/chaos-mesh#deploy-chaos-mesh)にしたがって、実際に`Kubernetes`で`Chaos Mesh`を動かしてみましょう。

#### `Helm`使用して、`Chaos Mesh`展開

`Helm`チャートを取得する

```sh
git clone https://github.com/pingcap/chaos-mesh.git
cd chaos-mesh/
```

カスタムリソースをインストールする

```sh
kubectl apply -f manifests/
# CRD を確認する
kubectl get crd podchaos.pingcap.com
kubectl get crd networkchaos.pingcap.com
kubectl get crd iochaos.pingcap.com
kubectl get crd timechaos.pingcap.com
```

通常、コンテナランタイムは`Docker`。もし他のコンテナランタイムを使用している場合はドキュメントを参照してください。

```sh
# 名前空間を作成する
kubectl create ns chaos-testing
# helm 2.X
helm install helm/chaos-mesh --name=chaos-mesh --namespace=chaos-testing
# helm 3.X
helm install chaos-mesh helm/chaos-mesh --namespace=chaos-testing
# Chaos Mesh pods を確認する
kubectl get pods --namespace chaos-testing -l app.kubernetes.io/instance=chaos-mesh
```

#### テストしてみましょう

テスト対象のアプリケーションコンテナを実行します。

```sh
kubectl create ns hello-chaos
kubectl -n hello-chaos create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
```

カオスを設定します。

例：[https://github.com/pingcap/chaos-mesh/blob/master/examples/pod-kill-example.yaml](https://github.com/pingcap/chaos-mesh/blob/master/examples/pod-kill-example.yaml)

hello-chaosネームスペースの`app=kubernetes-bootcamp`というラベルがついたpodを1分ごとにランダムに一つ終了させます。

```yaml
apiVersion: pingcap.com/v1alpha1
kind: PodChaos
metadata:
  name: hello-pod-kill
  namespace: chaos-testing
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - hello-chaos
    labelSelectors:
      "app": "kubernetes-bootcamp"
  scheduler:
    cron: "@every 1m"
```

`pod kill`を適用して、動作しているか確認します。

```sh
# pod kill を適用する
kubectl apply -f hello-pod-kill.yaml
# pod 再起動を確認する
watch -n 1 kubectl -n hello-chaos get pods
```

## 今後の仕事

現在の`Chaos Mesh`にはいくつかの欠陥があります。

* カオスの追加は面倒です
* カオスイベントを視覚化できません

作業効率を向上し、より良い体験を得るために、`Chaos Mesh`はこれからも将来的に改善し続けます。
