---
layout: post
title: Anyway，我被UIWindow打脸了
date: 2019-1-27 19:40:00.000000000
tags: UIWindow makeKeyAndVisible hidden=NO
---


![](https://apestalk.github.io/assets/BlogImages/window.png)


> 需求好轮回，bug绕过谁。

近日捕获一bug，经排查定位后发现，该bug属于通过``[UIApplication sharedApplication].keyWindow.rootViewController``方法拿到了错误的控制器，导致后面执行的代码异常。下面请听我细细为你讲述该bug的前世今生。

因为是线上bug，奔溃在了广告业务加载完了之后，考虑到会影响大量用户，内心甚是恐慌。因为奔溃的地方是最近才改的代码，所以起初我一直在最近的两个业务需求里排查问题，因为最近另外一个需求也添加了一个window。事实上，我被狠狠的打脸了，脸好疼。。。


![](https://apestalk.github.io/assets/BlogImages/ohshit.jpg)

经过测试和分析，最近新增的这个window并不会导致这个问题。确定启动的时候也没有类似UIAlertView这种弹窗会弹出来，那为什么通过``[UIApplication sharedApplication].keyWindow``拿到了错误的window呢？不找到问题点实在是难以饶恕自己啊！

到后来，再看线上更新最近这个版本的用户很多，但是这个崩溃只有一例，再加上确认不是因为启动时和其他业务互相影响导致的启动就崩的严重bug。我的内心稍微平复了一些，没那么慌了，能够静下心来理性地分析问题了。

回归到问题源头，首先，我们知道当我们调用UIWindow的``makeKeyAndVisible``时，系统会调用它的``hidden = NO``让它显示出来，另外会帮我们把它加入``[UIApplication sharedApplication].windows``数组中。当我们调用``[UIApplication sharedApplication].keyWindow``时，系统给我们返回一个最近调用了``makeKeyAndVisible``方法的window。一般正常情况下我们一个APP只有一个window，通过``[UIApplication sharedApplication].keyWindow``拿我们的主window是没有问题的，只有当我们程序中出现多个window的时候才可能会出现问题。那么第二个问题来了，什么时候回出现多个window呢，也就是``[UIApplication sharedApplication].windows``数组长度不为1，那必须是显式或隐式地调用了window的``makeKeyAndVisible``方法才会出现这种情况。

然后，我在项目中搜``makeKeyAndVisible``方法，找到了一个很早之前的业务需求处用到了一个window并调用了``makeKeyAndVisible``方法，当时是为了解决其他问题。实在是没有想到会是跟这里的业务相互影响了。一把辛酸泪啊，突然想说一句“不是你的代码没有bug，而是你的bug需要点时间。”

另外需要注意的是，项目中有历史遗留的``UIAlertView``和``UIActionSheet``的，要尽快把它们改掉了，因为随着业务的不断增长，在有这些弹窗的时候如果去调用``[UIApplication sharedApplication].keyWindow``拿到的会是一个``_UIAlertControllerShimPresenterWindow``，它的rootViewController是一个``UIApplicationRotationFollowingController``。很明显，在它们弹出来的时候，系统会新建一个UIWindow，并且为了在不管当前视图层级结构是怎样的，都将弹窗置为最上层让用户能看到，所以不得不调用了window的``makeKeyAndVisible``方法。这是一个隐患，我猜测苹果新推出``UIAlertController``来替代``UIAlertView``和``UIActionSheet``有这方面的考量。

所以，这里有三个建议：

1、建议是尽快把项目中的``UIAlertView``和``UIActionSheet``换成``UIAlertController``。

2、在拿主window的时候通过``[UIApplication sharedApplication].delegate.window``方式拿。

3、在使用除主UIWindow外的其他window时，如果你只是想把它展示出来，直接调用``window.hidden = NO``即可，如果没有强制要将该window置为最上层，就不要去调用``makeKeyAndVisible``。


这次被UIWinow打脸呢，有点意外，幸好没有酿成重大必陷bug。这件事又给我了一个大教训，那就是遇事不要慌张，不要一上来就凭借自己的经验随便下结论，一定要静下心来理性地思考，先仔细分析bug，先摸清bug的来龙去脉，再顺藤摸瓜就很简单了。







![欢迎关注支持](https://apestalk.github.io/assets/BlogImages/wx.jpeg)
<center>感谢关注支持</center>
