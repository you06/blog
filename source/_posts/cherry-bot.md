title: Cherry Bot
route: cherry-bot
date: 2020-04-23T19:12:29.516Z
tags: 
    - 技术杂谈
    - English
--------------------------
![bot logo](https://user-images.githubusercontent.com/9587680/60788142-95abc100-a18e-11e9-9a42-fbf21a023449.jpg)

## How I cames from

I will talk about this bot from my story. Things began from was July 2019, I joined PingCAP as an intern. Holy I was a noob of golang and knew nothing about rust that time. The best way learning a language is by practising, I started my first golang project named cherry-picker. As the name shows, this is a tool which helps PingCAP's engineers cherry pick PRs from master branch to release branch. The project soon ran out of my expectation, we started in TiDB, then expanded to TiKV and PD. I noticed that this bot is such awesome, in the following months, the bot growed quickly, some features were deprecated and some were optimized. Nowadays, the bot is widely used in PingCAP, and we think it's time to share the bot with you!

<!-- more -->

## About my name

For a long time, I name the bot "cherry-picker", because it's the first feature of the bot. This time, I give it a new name, "Cherry Bot", so the logo of bot will not change lol! By now, there are over 10 providers in this bot 🍻.

## The mostly used features

### Cherry Pick

Bug fix pull requests in repositories with multi branches like `release-xx` and `master` are usually required to be cherry picked to release branches, manually do this would be tedious. It would be worse if we forget cherry picking bug-fix pull requests. Then the bot comes, it cherry picks every merged PR with specific labels, like pull request labeled with `needs-cherry-pick-2.0` will be cherry picked to `release-2.0` branch. What about conflicts? The conflicts are common between branches, the bot can be configured to generate conflicted pull requests either report a conflict failure. The conflict pull requests would be like the following which can be specified by IDEs.

```
<<<<<<< HEAD
branch contents
=======
bug fix contents
>>>>>>> 1234567890abcdef
```

### Auto Merge

If you are using some languages which have a heavy cost in compile or there are some tests that would last for a long time. After the reviewing of the pull request is finished, the CI may still running. Sometimes we would require the pull request tested with up to date changes. GitHub offers `Update Branch` to merge the latest changes from base branch, but after this action, the CI should be restarted. Think about this, a pull request is ready to merge and waiting for CI finished, then the base branch is updated, which makes a force update of this pull request and the CI restarts. A thousands years later, the CI is finally about to success, a pull request merged into the base branch again which makes the CI should restarted again. The same thing may be caused by you AFK while waiting for CI done. This makes pull requests under the state of competition, and the most of the CI's job would be waste. The bot offer a mechanism to merge pull requests in a queue. When maitainers commented with `/merge`, the bot will labeled the pull request with `status/can merge` which stands for the pull request will be auto merged. If the pull request need to be waited for others, bot will comment with all the waited pull request numbers. This auto merge job can be canceled by remove the `status/can merge` label easily.

### Redeliver Command

PingCAP uses Jenkins for CI trigger and scheduler, some commands in Jenkins are only available for organization members. Besides, we also want some commands can be used by pull request owner, that's why redeliver command comes. Comment with `@bot /run-integration-test`, then the bot will trigger this for you by comment with `/run-integration-test`. Do not try call `@bot /merge`, this would never work 😕.

### Auto Update

Some large project will have multiple repos. For example `TiKV` have the dependency `rust-rocksdb` and `rust-rocksdb` is based on `RocksDB` and `Titan`. This means any updates from `RocksDB` or `Titan` will need to be updated to `rust-rocksdb` then `TiKV`. We are always not to do this manually, that's the bot's time, give me the chore. The bot will auto file up a pull request to update the sub modules.

### More Providers

There are still many providers I've not mentioned in this blog, check it out if you're interested https://github.com/pingcap-incubator/cherry-bot/tree/master/pkg/providers. For those features added into bot, I try to make them common instead of dedicated. And I'm expecting to get some feedbacks from community users.

## About the Code

When I code this bot, I'm new to Golang. There are some unnacessary interfaces, redundant expressions and unsatisfied error handlings. I've made some efforts to make clean up the code, but it's so so so a big work, it's just like code the bot again 😭.

## Grumbles

I've written a lot of Golang code in the past year. This language is like a quad bike which makes everyone easy to drive. When use Golang in some projects, I often code many high repeatable expressions, like `if err != nil` which makes me feel pain. Also the feeling of controlling everything in CPP is missing. It's not smart in both grammar and extreme performance. However, it's still a preferred language for building some apps, where the performance is good enough and you don't need to worry about whether memory is in stack or heap(I'm not sure whether it's an advantage). Anyway, longtime writing Golang is tired, more features are always wanted like iterator, general types etc. The last complain, when doing some worthless jobs for fun, I'm very willing to try languages with more complexity. A further thinking, will you like a language which brings the ability of low level programming without knowledge of computer system?
