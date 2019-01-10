---
layout: post
title: 也谈使用NSUserDefaults处理默认值
date: 2019-1-10 18:30:00.000000000
tags: iOS NSUserDefaults UserAgent registerDefaults
---


![欢迎关注支持](https://apestalk.github.io/assets/BlogImages/buildings-clouds-dusk.jpg)


> 在Cocoa中，NSUserDefaults类的API用于存储和获取用户偏好设置。最近在了解设置webview的UserAgent时第一次用到了NSUserDefaults的``registerDefaults:``方法，顺带了解一下该方法。



## 始终使用``objectForKey:``方法



我一直建议大家使用``objectForKey:``方法来获取值，而不是直接使用诸如``boolForKey:``、``integerForKey:``、``stringForKey:``之类的方法。因为通过``-[NSUserDefaults objectForKey:]``方法来获取值并加上类型判断，可以增强我们程序的健壮性。看下面的例子：



```

BOOL showTutorial = [[NSUserDefaults standardUserDefaults] boolForKey:@"ShowTutorial"];
```



此时你没有办法判断到底存的值是NO还是没有存过值，如果想做类型区分就很难了，换成下面这样就好区分好理解了：



```

NSNumber *showTutorial = [[NSUserDefaults standardUserDefaults] objectForKey:@"ShowTutorial"];
BOOL shouldShowTutorial = NO;
if(!showTutorial){
//no set yet, put some Extra logic
shouldShowTutorial = YES;
}else{
shouldShowTutorial = [showTutorial boolValue];
}
```



## 关于``registerDefaults:``方法



``registerDefaults:``方法可以注册一个包含APP偏好设置默认值的字典，设置之后，在APP中通过``objectForKey:``方法获取值时，如果之前没有保存过值，则返回通过``registerDefaults:``方法注册的默认值。你必须记住的一点是：**``registerDefaults:``方法注册的默认值不会存储到硬盘，只在程序生命周期中有效，所以，你必须每次在程序启动时调用该方法去配置，通常在``application:didFinishLaunchingWithOptions:``方法中配置。**  



例子胜千言：



```


NSUserDefaults *ud = [NSUserDefaults standardUserDefaults];

//重新启动，之前通过registerDefaults设置的值不存在，说明没有做永久存储。
NSDictionary *rep = [ud dictionaryRepresentation];
NSLog(@"rep=%@", rep);    


[ud removeObjectForKey:@"name"];
[ud setObject:@"Apes" forKey:@"name"];
[ud registerDefaults:@{@"Apes Talk": @"male"}];
NSLog(@"name=%@",[ud objectForKey:@"name"]);//Apes
[ud removeObjectForKey:@"name"];
NSLog(@"name=%@",[ud objectForKey:@"name"]);//ApesTalk




[ud removeObjectForKey:@"twiceKey"];
[ud registerDefaults:@{@"twiceKey": @"Apes"}];
[ud registerDefaults:@{@"twiceKey": @"ApesTalk"}];
NSLog(@"twiceKey=%@", [ud objectForKey:@"twiceKey"]);//ApesTalk
[ud synchronize];


rep = [ud dictionaryRepresentation];
NSLog(@"rep=%@", rep);
```



一个常见的使用``registerDefaults:``方法的例子是设置webview的userAgent，客户端通过在userAgent中加入特殊标识，HTML页面通过这个标识来判断该页面是在APP中加载的还是在浏览器中加载的，以此可以做一些定制化的功能。这里以UIWebView为例，WKWebView在iOS9之后有一个新的设置userAgent的属性customUserAgent。



```


//获取webview的默认userAgent
UIWebView *tmpWebView = [[UIWebView alloc] initWithFrame:CGRectZero];
NSString *oldAgent = [tmpWebView stringByEvaluatingJavaScriptFromString:@"navigator.userAgent"];
//这里以简单的在旧的userAgent后面新增特殊标志为例
NSString * newAgent = [oldAgent stringByAppendingString:@" isInApp"];
//注册
NSDictionary * dictionnary = [[NSDictionary alloc] initWithObjectsAndKeys:newAgent, @"UserAgent", nil];
[[NSUserDefaults standardUserDefaults] registerDefaults:dictionnary];
[[NSUserDefaults standardUserDefaults] synchronize];
```



这个配置是怎么生效的呢？为什么通过NSUserDefaults设置的默认值为什么会影响到webview？webview在加载的时候是怎么读取userAgent的值的呢？



实际上，NSUserDefaults有个域domain（由不同级别的层次组成）的概念，每当你获取key对应的value时，NSUserDefaults会在这个域下由上到下查找对应的key，返回第一个找到的值，有点像响应链。域可以持久化存储在硬盘中或只是存储在内存中。



最重要的一个域是应用程序域（application domain），这里存储着调用``set...ForKey:``保存的APP设置信息。相对的，通过调用``registerDefaults:``方法保存的信息存储在低优先级的内存中的registration domain中，作为在应用程序域中找不到的任何值的一个后备（fallback）。



还有一个全局的域，存储系统范围的设置。特定语言的域存储地区偏好，如每个地区的月名称或日期格式。



最后但并非不重要的是，苹果使用相同的技术使我们能够通过命令行参数覆盖用户默认值。每个Cocoa应用程序自动检查命令行参数的键/值对，并将这些添加到适当命名的参数域（argument domain）下。由于参数域（argument domain）的优先级最高，我们可以使用它来临时覆盖任何偏好值。



看到这里我们再回过头想想设置UserAgent的问题，实际上，我们通过高优先级的参数域（argument domain）来临时修改了APP的偏好设置。然后我们可以大胆的猜测一下，UIWebView在加载的时候内部会自动从APP的偏好设置中读取UserAgent的值，所以我们设置的UserAgent才会生效。



完整的NSUserDefaults的域搜索顺序如下：



|Domain|State|

|---|---|

|NSArgumentDomain|volatile|

|Application|volatile|

|NSGlobalDomain|persistent|

|Languages|persistent|

|NSRegistrationDomain|volatile|



另外，通过Settings.bunlde设置的偏好设置信息也需要在程序启动时通过``registerDefaults:``方法注册才有效的。Settings.bunlde是个有意思的话题，不过目前使用该技术的APP较少，感兴趣的朋友可以看下这篇文章：[iOS开发中Settings.bundle的使用](https://blog.devzeng.com/blog/ios-settings-bundle.html)。





参考：



[Handling Default Values With NSUserDefaults](https://oleb.net/blog/2014/02/nsuserdefaults-handling-default-values/)

[iOS开发中Settings.bundle的使用](https://blog.devzeng.com/blog/ios-settings-bundle.html)





![欢迎关注支持](https://apestalk.github.io/assets/BlogImages/wx.jpeg)
<center>感谢关注支持</center>
