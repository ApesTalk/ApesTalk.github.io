---
layout: post
title: CocoaPods使用备忘
date: 2018-08-21 22:30:00.000000000
tags: Xcode CocoaPods 私有Pod Podfile pod-install pod-update
---

> 使用CocoaPods也有很长一段时间了，最近几个月的时间里也主导了公司私有Pods的创建和使用。在此期间踩过了不少坑，在踩坑的过程中也收获了不少经验，更加熟练地掌握了CocoaPods的一些指令的使用。本篇作为这段时间收获的备忘。



## 一、CocoaPods简介

CocoaPods是专门为iOS工程提供第三方依赖库的管理工具，通过CocoaPods，我们可以更方便地管理每个第三方库的版本，而且不需要我们做太多的配置，就可以直观、集中和自动化地管理我们项目的第三方库。


CocoaPods将所有依赖的库都放在一个名为Pods的项目下，然后让主项目依赖Pods项目。然后，我们编码工作都从主项目转移到Pods项目。Pods项目最终会编译为一个libPod-项目名.a静态库，主项目依赖于这个静态库。

对于资源文件，CocoaPods 提供了一个名为 Pods-resources.sh 的 bash 脚本，该脚本在每次项目编译的时候都会执行，将第三方库的各种资源文件复制到目标目录中。

CocoaPods 通过一个名为 Pods.xcconfig 的文件来在编译时设置所有的依赖和参数。


CocoaPods是用 Ruby 写的，并由若干个 Ruby 包 (gems) 构成的。在解析整合过程中，最重要的几个 gems 分别是： [CocoaPods/CocoaPods](https://github.com/CocoaPods/CocoaPods/), [CocoaPods/Core](https://github.com/CocoaPods/Core), 和 [CocoaPods/Xcodeproj](https://github.com/CocoaPods/Xcodeproj)。



### CocoaPod的核心组件


- CocoaPods/CocoaPod
这是是一个面向用户的组件，每当执行一个 pod 命令时，这个组件都将被激活。该组件包括了所有使用 CocoaPods 涉及到的功能，并且还能通过调用所有其它的 gems 来执行任务。

- CocoaPods/Core
Core 组件提供支持与 CocoaPods 相关文件的处理，文件主要是 Podfile 和 podspecs。

- Podfile
Podfile 是一个文件，用于定义项目所需要使用的第三方库。该文件支持高度定制，你可以根据个人喜好对其做出定制。更多相关信息，请查阅 Podfile 指南。

- Podspec
.podspec 也是一个文件，该文件描述了一个库是怎样被添加到工程中的。它支持的功能有：列出源文件、framework、编译选项和某个库所需要的依赖等。

- CocoaPods/Xcodeproj
这个 gem 组件负责所有工程文件的整合。它能够对创建并修改 .xcodeproj 和 .xcworkspace 文件。它也可以作为单独的一个 gem 包使用。如果你想要写一个脚本来方便的修改工程文件，那么可以使用这个 gem。

CocoaPod的安装和配置，以及Podfile中第三方库引用的语法规则（特别是版本号的语法格式）这里就不赘述了，下面挑重点讲一讲。


## 二、多target时Podfile该如何写？

我的建议是使用Ruby语法，定义不同的分组，然后不同的target可以自由选择依赖哪些分组，这种方式看起来更简洁，对于多target的项目来说也更友好：


```
platform :ios, '8.0'

def commonPods #通用pods集
    pod 'AFNetworking', '~> 2.0'
    pod 'Masonry'
end

def appOnlyPods #app专用pods集
    pod 'MBProgressHUD'
end

def extensionPods #扩展专用pods集
    pod 'GTSDKExtension'
end

target :TestCocoaPods do
    commonPods
    appOnlyPods

    target :TestCocoaPodsTests do
    inherit! :search_paths
    # Pods for testing
    end

    target :TestCocoaPodsUITests do
        inherit! :search_paths
        # Pods for testing
    end
end

target :SecondTarget do
    commonPods
end

```

## 三、如何忽略Pods警告？

有些第三方Pod集成进来会有一大堆警告信息，如果你看着比较难受想把它忽略的话，在Podfile中对应的target或分组下加上关键字``inhibit_all_warnings``即可。

## 四、如何直接引用第三方库中的头文件？

在用CocoaPods集成第三方库之后，默认情况下，我们需要使用类似``#import <XXX/YYY.h>``的方式引入第三方库的头文件。我们可以在Build Settings -> User Header Search Paths中添加``${SRCROOT}``并设置成recursive，这样我们就可以直接使用``#impot "YYY.h"``这种方式了。


## 五、``pod install`` or ``pod update``?


如官方文档所说，``pod install`` 和 ``pod update``确实是大家最容易搞混的两条指令，很多人还没搞清楚这两条指令的区别，反正不管三七二十一上来就是一个``pod update``，大家一定要搞清楚这两条指令的区别。


按照官方文档所说，``pod install``在第一次检索集成第三方以及每一次在Podfile中新增、更改或删除pod的时候使用。每一次执行``pod install``命令，它都会下载安装新的pod，并且会把每一个安装的pod的版本信息写入Podfile.lock文件。Podfile.lock文件跟踪每一个安装的pod的版本并且上锁。每一次执行``pod install``命令，只解决还没有在Podfile.lock中列出的依赖：对于已在Podfile.lock中列出的pod，会下载指定的版本，不会检查是否有新版本。对于没有在Podfile.lock中列出的pod，它会搜索并安装Podfile中指定的版本。


直接执行``pod update``命令会检查安装Podfile中列出的所有pod的新版本（往往比较慢）。

执行``pod update PODNAME``命令会检查PODNAME的新版本（不考虑Podfile.lock中记录的版本信息），它会把PODNAME更新为最新版本，只要跟Podfile中指定的版本匹配。也就是说，``pod update PODNAME``将PODNAME更新到Podfile中指定的版本，可以是更新到老版本也可以是更新到新版本，取决于Podfile。（比如：如果此时Podfile中指定了``pod 'AFNetworking', '~> 2.0'``，此时执行``pod update AFNetworking``并不会把AFNetworking更新到最新版本（因为此时的版本满足大于等于2.0版），必须先修改Podfile中的版本信息才会更新到指定版本）。

两者的区别：

- 用``pod install``命令来安装新的pod，每次在Podfile中新增和删除pod都使用``pod install``命令。

- 在Podfile中添加新的pod后应该用``pod install``命令，而不是``pod update``命令。通过``pod install``命令安装新的pod而不用担心在同一进程中修改已有的pod。

- ``pod update``命令仅用在更新指定pod到指定版本或者更新所有pod。


我的建议是：该用``pod install``的时候不要用``pod update PODNAME``。另外，尽量不要用``pod update``，因为它是全部检查一遍，不仅慢有时候还会出现坑。比如有一个依赖的第三方库本来是2.0版本的用的好好的，因为它是国外的资源，下载起来非常慢，我们在没有bug的情况下是不希望轻易去更新它的，那么如果你上来就是一个``pod update``指令，OK， 如果你Podfile中指定了每次使用最新版本（不指定版本号），那么CocoaPods就会去下载最新的这个第三方库，那在下载完成之前你还要不要做其他事情了？这还是情况好的，如果这个最新的版本一直下载失败，所以一直集成失败怎么办？


## 六、如何创建私有Pod？

要创建私有Pod，首先我们需要两个私有仓库，一个放私有Pod源码，一个放私有Pod的说明书（类似公有Pod的[CocoaPods/Specs](https://github.com/CocoaPods/Specs)）。

### 1、添加私有Spec仓库到本地

```
pod repo add privateSpecs your_privateSpecs.git

```

如果执行成功，之后便可以通过``pod repo list``命令查看本地Spec仓库列表，正常情况下会有一个公有的CocoaPod官方的master repo 和你的 privateSpecs repo，并可以看到它们在本地的存放路径（其实在``~/.cocoapods/repos``目录下）。

### 2、创建私有Pod

在私有Pod代码所在文件夹下执行``pod spec create your_podName``在该目录下创建一个your_podName.podspec说明书文件。之后的工作就是编辑这个说明书文件了，这里简单注明一下规则：

```
Pod::Spec.new do |s|

  s.name         = "ATCategory"
  s.version      = "0.0.1"
  s.summary      = "共用扩展类集合"
  s.description  = <<-DESC
  大家如果需要用到扩展，都使用这里已有的扩展啦。
                   DESC
  s.homepage     = "your_privatePodGit_address/ATCategory"
  s.author       = { "ApesTalk" => "lqcjdx@163.com" }
  s.platform     = :ios
  s.platform     = :ios, "8.0"
  s.source       = { :git => "your_privatePodGit_address", :tag => "#{s.version}"
  # 如果你有多个私有Pod放在一个仓库里，你可以修改tag像下面这样，对应打tag的时候的规则就对应需要变成PodName-v0.0.1这样子了
  # s.source       = { :git => "your_privatePodGit_address", :tag => s.name + "-v"+"#{s.version}"
}
  s.source_files  = 'ATCategory/**/*'
  s.public_header_files = 'ATCategory/Category/*.h'
  s.requires_arc = true
  s.frameworks = 'UIKit','Foundation'

# 依赖的系统library，这里是指系统的类似libz.tbd、libxml2.tbd这类的系统库
# s.library = 'z' // 单个
# s.libraries = 'z','xml2' // 多个

# 第三方.a
# s.vendored_libraries =
# 第三方frameworks文件
# s.vendored_frameworks =
# 依赖关系，该项目所依赖的其他库，如果有多个需要填写多个s.dependency
# s.dependency 'AFNetworking', '~> 2.3'
# 资源文件地址
# s.resource_bundles = {
#   'ATCategory' => ['ATCategory/Images/*.png']
# }
end
```

### 3、提交源代码并打tag

注意这里tag必须跟podspec文件中的tag保持一致，因为CocoaPod是通过podspec文件中的tag去找源文件的，如果tag对应不起来就会验证失败。打好tag提交到远端。


### 4、验证podspec文件合法性和可选参数

有两种验证方式，一种是本地验证``pod lib lint your_podName.podspec``和联网验证``od spec lint your_podName.podspec``。建议大家都用联网验证。

这里可选参数有:

- ``--allow-warnings``：允许警告
- ``--sources=‘master,privateSpecs``：指定源，比如你的私有pod同时依赖了公有库和私有库，你必须指定源才行，因为默认只会去在公有源中查找对应的依赖
- ``--use-libraries``：如果使用了静态库，记得加上它

### 5、提交说明书文件到私有说明书库

``pod repo push privateSpecs your_podName.podspec``，同样的加上上面验证时使用到的可选参数。

### 6、如果使用素材？

官方建议pod中的素材用bundle的形式避免和主项目中的文件名发生冲突，那么集成后我们如何使用bundle中的素材呢？

```
NSBundle *bundle = [NSBundle mainBundle];
//NSBundle *bundle = [NSBundle bundleForClass:[ClassFromPodspec class]];//对于静态库，拿到的是mainBundle，如果是动态库，拿到的是类所在的bundle。使用动态库需要在Podfile中开启use_frameworks!
NSURL *wttpodBundleURL = [bundle URLForResource:@"WTTPod" withExtension:@"bundle"];
NSBundle *wttpodBundle = [NSBundle bundleWithURL: wttpodBundleURL];
UIImage *img = [UIImage imageNamed:@"Chat_checkin_empty_stu" inBundle:wttpodBundle compatibleWithTraitCollection:nil];
```


### 7、关于subspec

如果我们的pod中文件比较多，而我们又希望能像AFNetworking那样集成后分几个物理文件夹（默认会把所有文件都放在一个物理文件夹下，文件太多会显得很乱），那么就要用到subspec来把我们的pod分成几个独立的子模块。

```

s.subspec '子模块名称' do |别名，不能和子模块名称相同，比如ss|

ss.source_files = ''

end

```

具体怎么用，大家可以参考AFNetworking.podspec文件中的写法。


## 七、如何使用私有库

如果我们同时使用了公有库和私有库，我们只需要在Podfile的头部同时把公有库和私有库的source加上即可。


## 八、``pod search``搜不到私有库？或者搜得到``pod install``失败？

在提交私有库说明书之后，先执行一下``pod repo update privateSpecs``，然后再集成。


## 九、集成某一个pod速度过慢，比如MobileVLCKit总是下载失败

把对应版本的MobileVLCKit下载下来放在访问速度更快的地方（比如内网服务器或者本机用Python开启一个FTP服务）在本机master repo源中搜索找到对应版本的MobileVLCKit.podspec.json文件，把其中的source改成我们存放MobileVLCKit.tar.xz文件地址，之后再执行相关指令集成。



如何利用Python开启一个本地FTP服务：

cd到要共享的目录下，执行``python -m SimpleHTTPServer 8000``，之后同一个局域网内就可以通过``本机ip:8000``访问到该共享文件夹了。

```

"http":"http://192.168.210.111:8000/MobileVLCKit-3.1.2-bf58e19-37855b857a.tar.xz"

```

获取本机IP地址的方法：按住Option的同时点下Mac菜单栏的无线网Icon，在下拉列表中即可看到IP地址。也可以在终端中输入``ifconfig en0``命令查看。


## 十、验证podspec时，报错 symbol(s) not found for architecture i386

检查一下是否私有Pod中使用到的什么文件不支持i386架构，比如什么.a文件。


解决办法：在podspec文件中指定支持的架构


```
valid_archs = ['armv7s','arm64',]
s.xcconfig = {
  'VALID_ARCHS' =>  valid_archs.join(' '),
}
s.pod_target_xcconfig = {
    'ARCHS[sdk=iphonesimulator*]' => '$(ARCHS_STANDARD_64_BIT)'
}
```

同样的，如果在验证过程中遇到xcodebuild: Returned an unsuccessful exit code.却不报具体的错，去看看NOTE类型的信息，如果看到missing required architecture i386 in file此类消息，说明某个.a或者frawork不支持i386架构，需要在podspec文件中写明改pod支持哪些架构。


## 十一、``pod init``失败？


用Xcode9.4.1新建一个项目，然后执行``pod init``（Cocoapods1.4.0版本）时提示失败，错误提示如下：



```

――― MARKDOWN TEMPLATE ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――

### Command

/usr/local/Cellar/cocoapods/1.4.0/libexec/bin/pod init


### Report

* What did you do?

* What did you expect to happen?

* What happened instead?


### Stack

   CocoaPods : 1.4.0
        Ruby : ruby 2.3.3p222 (2016-11-21 revision 56859) [universal.x86_64-darwin17]
    RubyGems : 2.5.2
        Host : Mac OS X 10.13.3 (17D47)
       Xcode : 9.4.1 (9F2000)
         Git : git version 2.15.2 (Apple Git-101.1)
Ruby lib dir : /System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib
Repositories : ios-cocoapodsSpecs - http://git.zhihuishu.com/coder/ios-cocoapodsSpecs.git @ 4c4f38416c28d58d4dda2a706879956c5861a55e
               master - https://github.com/CocoaPods/Specs.git @ 1eab292c36f39ecb104af54e474b00f142e57d0f


### Plugins


cocoapods-deintegrate : 1.0.2
cocoapods-plugins     : 1.0.0
cocoapods-search      : 1.0.0
cocoapods-stats       : 1.0.0
cocoapods-trunk       : 1.3.0
cocoapods-try         : 1.1.0


### Error


RuntimeError - [Xcodeproj] Unknown object version.
/usr/local/Cellar/cocoapods/1.4.0/libexec/gems/xcodeproj-1.5.4/lib/xcodeproj/project.rb:217:in `initialize_from_file'
/usr/local/Cellar/cocoapods/1.4.0/libexec/gems/xcodeproj-1.5.4/lib/xcodeproj/project.rb:102:in `open'
/usr/local/Cellar/cocoapods/1.4.0/libexec/gems/cocoapods-1.4.0/lib/cocoapods/command/init.rb:41:in `validate!'
/Library/Ruby/Gems/2.3.0/gems/claide-1.0.2/lib/claide/command.rb:333:in `run'
/usr/local/Cellar/cocoapods/1.4.0/libexec/gems/cocoapods-1.4.0/lib/cocoapods/command.rb:52:in `run'
/usr/local/Cellar/cocoapods/1.4.0/libexec/gems/cocoapods-1.4.0/bin/pod:55:in `<top (required)>'
/usr/local/Cellar/cocoapods/1.4.0/libexec/bin/pod:22:in `load'
/usr/local/Cellar/cocoapods/1.4.0/libexec/bin/pod:22:in `<main>'


――― TEMPLATE END ――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――

[!] Oh no, an error occurred.

Search for existing GitHub issues similar to yours:
https://github.com/CocoaPods/CocoaPods/search?q=%5BXcodeproj%5D+Unknown+object+version.&type=Issues

If none exists, create a ticket, with the template displayed above, on:
https://github.com/CocoaPods/CocoaPods/issues/new

Be sure to first read the contributing guide for details on how to properly submit a ticket:
https://github.com/CocoaPods/CocoaPods/blob/master/CONTRIBUTING.md

Don't forget to anonymize any private data!

Looking for related issues on cocoapods/cocoapods...
 - Pod Update: RuntimeError - [Xcodeproj] Unknown object version. Xcode Beta 5
   https://github.com/CocoaPods/CocoaPods/issues/8003 [closed] [17 comments]
   a day ago

 - RuntimeError - [Xcodeproj] Unknown object version.
   https://github.com/CocoaPods/CocoaPods/issues/7697 [closed] [28 comments]
   3 weeks ago

 - Pod init. Unknown object version
   https://github.com/CocoaPods/CocoaPods/issues/7907 [closed] [2 comments]
   03 Jul 2018

and 42 more at:
https://github.com/cocoapods/cocoapods/search?q=[Xcodeproj]%20Unknown%20object%20version.&type=Issues&utf8=✓

```

这种失败原因是Cocoapods和xcodeproj版本兼容问题。


尝试了网上的解决办法[Run gem install xcodeproj:1.4.1](https://link.jianshu.com/?t=https://github.com/CocoaPods/CocoaPods/issues/6168)，依然失败。

解决办法：打开项目，在Project Document下将Project Format从Xcode 9.3-compatible修改为Xcode 8.0-compatible即可。

![Xocde_project_format](https://apestalk.github.io/assets/BlogImages/Xocde_project_format.png)


## 十二、在pod中引入项目文件报错（file not found）

在开发中有时候需要在pod中import项目中的文件进行调试或测试（当然这种情况比较少见），但是当你输入import的时候会发现系统根本没法联想到你想用的项目中的文件，即使你手动写入，也会报file not found的错误。

此时，可以在我们的Pods项目中的Build settings下找到 User Header Search Paths，添加一行，并设置``$(SRCROOT)/..``为recursive。注意有``/..``，意思是到当前目录的上级目录。意思是在上级目录下递归查找文件。


![欢迎关注支持](https://apestalk.github.io/assets/BlogImages/wx.jpeg)
<center>感谢关注支持</center>

