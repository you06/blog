title: 关于 TiDB 日文文档
route: about-tidb-jp-doc
date: 2019-10-15T17:58:47.164Z
tags: 
    - 技术杂谈
--------------------------
`TiDB`日文文档这个坑其实早就有想法了，但原本也没打算这么早开。

- [doc](https://you06.github.io/tidb-docs-jp/)
- [repo](https://github.com/you06/tidb-docs-jp)

<!-- more -->

# 关于开坑

周末的时候在看日语语法，看了之后觉得需要做些题，我一个萌新也不知道做什么好，就跑去问大手子。大手子先是说“我不做题的”，然后给我指了条明路——翻译ben子，据说多翻翻就能过 N1 了。

好吧我这么正经的人怎么会去翻ben子，然后就想起了这个日文文档的坑，那就开吧。然后当天就用[`gitbook`](https://www.gitbook.com/)搞了一份出来，写了个序，摸了。

# 文档的一些选择

但是过了几天之后我就放弃了`gitbook`，采用了[`mdbook`](https://github.com/rust-lang-nursery/mdBook)这个工具来写文档，这里稍作解释。主要原因是我不愿意在这件事情想引入一个第三方平台，和许多静态博客一样，源码放在`GitHub`，构建后的网站放在`GitHub Pages`上是最简洁的流程。其二是因为我司的环境比较融入`rust`生态，所以`mdbook`作为一个`cargo`包，个人会比较想要用它。

构建和文本规范检查用的是[`GitHub Actions`](https://github.com/features/actions)，作为`GitHub`家自产的`CI`，可以说上手体验是很不错了，如果用第三方`CI`，部署到`gh-pages`分支会很头疼，可以参考 [rust-lang/simpleinfra](https://github.com/rust-lang/simpleinfra/blob/master/travis-configs/static-websites.yml)里的`travis`配置。显然我不是一个`travis`或是`CircleCI`专家，所以在这种地方就想要尽可能的简单，所以借助[github-pages-deploy-action](https://github.com/JamesIves/github-pages-deploy-action)这个工具完成了自动部署的工作。

# 计划

`TiDB`目前开发中的版本是`4.0.0-alpha`，稳定的版本为`release-3.0`和`release-3.1`，`release-3.1`是从`release-3.0`切出来的，文档还是使用的 3.0 版本。因此，翻译工作也将从 3.0 开始做起。估计等我 3.0 做的差不多了就该 4.0 发布了，所以 2.0 的文档没有填坑计划。

这个翻译项目在很长一段时间内会作为个人项目进行，在有一定完成度之后会有进入到官方仓库的意向（反正我现在不好意思提到官方仓库去）。

当然也希望多多收到 PR，有大手来 review 我的翻译就更好了。

# 一些插曲

做这个翻译的过程中逼着自己写日文，也翻了很多日文技术相关的文档，参考措词。俺日本语本当苦手，然而正如大手所言，的确是个学习的好方式。

此前这个文档项目我只在微博和推上提到过，然后不知道怎么就被公司的大佬发现了，导致坑还没开（刚写完序，实际还没开始翻），就收获了一些`star`。并且推还被大佬点了个赞，我推上天天转色图应该没翻车吧？