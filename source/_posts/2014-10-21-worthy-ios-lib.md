title: 值得关注的iOS开源库
date: 2014-10-21 14:40:35
categories: iOS
tags: [iOS,Library]
---
无论你是否喜欢，iOS开发在近几年得到了很大的关注和发展，在开发中，我们其中很大一部分工作是关于如何写好代码，复用代码，已经依赖开源的源代码。

因此我想推荐一些好的开源库和框架来帮助开发，使开发过程变得更容易。
<!-- more -->

### Reusing code

那些需要开发者重复一遍又一遍的造轮子的岁月应该一去不复返了，我们在开始做任何事情之前都要首先在网上寻找一些对应的主题。

为什么？因为我们可能找到一个很好的开源项目，而那正是我们想要的。即使不是这样，我们也可能找到一些人在做相同或者类似的事情，他们可能遇到跟我们相同的问题，可能已经有了好的解决办法。

### The libraries and frameworks

接下来将列出一些最常用的开源库和框架，每一个都应该去尝试学习。

#### 1.Development tools
A. **Functional Reactive Programming**
	(1).[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) – implementation of a paradigm that makes development easier.  

B. **Toolkit**
	(1).[NimbusKit](https://github.com/jverkoey/nimbus)
	(2).[SSToolKit](https://github.com/soffes/sstoolkit)
	(3).[BlocksKit](https://github.com/zwaldowski/BlocksKit)
	(4).[APUtils](https://github.com/andrei512/APUtils)
	
C. **Debugging**
	(1).[DCIntrospect](https://github.com/domesticcatsoftware/DCIntrospect) – for UI debugging	
	(2).[PonyDebugger](https://github.com/square/PonyDebugger)
	(3).[SimulatorRemoteNotifications](https://github.com/acoomans/SimulatorRemoteNotifications/)– allows testing notifications on the simulator (faking them)

D. **Logging**
	(1)[CocoaLumberjack](https://github.com/CocoaLumberjack/CocoaLumberjack) – easy to use, super extensible and great performance logging component

E. **Formatting**
	(1)[FormatterKit](https://github.com/mattt/FormatterKit) – data formatting out of the box (addresses, distance between 2 points, time intervals, …)
	(2)[TransformerKit](https://github.com/mattt/TransformerKit) - A block-based API for NSValueTransformer, with a growing collection of useful examples.

F. **Unit Testing (read more about iOS Unit Testing [here](http://iosunittesting.com/))**
	(1)[Kiwi](https://github.com/allending/Kiwi) – good BDD library (specs, expectations, mocks, stubs, async tests). I did have to drop it due to async tests failing to work as expected
	(2)[Specta](https://github.com/specta/specta) – another TDD/BDD library. Very similar to Kiwi. Works great together with Expecta and OCMock. 
	(3)[Expecta](https://github.com/specta/expecta) – expectations for unit tests
	(4)[OCMock](https://github.com/erikdoe/ocmock)
	(5)[xctool](https://github.com/facebook/xctool) – very useful replacement for xcodebuild tool. It allows running Application Test targets from the command line and has a beautiful output. [Read more](http://nshipster.com/xctool/)

G. **Code Documentation**
	(1)[Cocoadocs](http://cocoadocs.org/) – documentation for all Cocoapods projects
	(2)[appledoc](https://github.com/tomaz/appledoc) – command line tool for generating “Apple like” code documentation from comments

H. **Acknowledgements**
	(1)Listing the licenses of all the open source libraries used is a recommendation (might become a must in the near future).(VTAcknowledgementsViewController)[https://github.com/vtourraine/VTAcknowledgementsViewController]

I. **Xcode plugins**
	(1)[Alcatraz](https://github.com/mneorr/Alcatraz) – Xcode package manager
	(2)[KFCocoaPodsPlugin](https://github.com/ricobeck/KFCocoaPodsPlugin) – CocoaPods plugin, requires Xcode 5
	(3)[VVDocumenter-Xcode](https://github.com/onevcat/VVDocumenter-Xcode) – great plugin that eases code documentation. You just need to press the shortcut (by default ///) and the code section below to the cursor gets documented (template). Then fill in the missing info – it’s very easy and it looks great.
	(4)[XcodeColors](https://github.com/robbiehanson/XcodeColors) – used with CocoaLumberjack to highlight errors and warnings
	(5)[HOStringSense-for-Xcode](https://github.com/holtwick/HOStringSense-for-Xcode) – takes care of string escaping
	(6)[KSImageNamed-Xcode](https://github.com/ksuther/KSImageNamed-Xcode) – autocomplete for image names
	(7)[Snippets](https://github.com/mattt/Xcode-Snippets) – Mattt Thompson gives away some of the snippets he uses. More about [Xcode Snippets](http://nshipster.com/xcode-snippets/)		

#### 2.Communication and data handling
	
A. **Networking**
	(1)[AFNetworking](https://github.com/AFNetworking/AFNetworking) – the best networking library for iOS and more than that, a true model on how an open source project should be run. It’s Github repo is a great place to learn.
	(2)[RestKit](https://github.com/RestKit/RestKit) – easy integration with REST webservices based on AFNetworking
	(3)[Reachability](https://github.com/tonymillion/Reachability)
	(4)[CocoaAsyncSocket](https://github.com/robbiehanson/CocoaAsyncSocket) – for low level networking (TCP, UDP)

B. **Browser** 
	(1)[TSMiniWebBrowser](https://github.com/tonisalae/TSMiniWebBrowser)

C. **Image caching**
	(1)[SDWebImage](https://github.com/rs/SDWebImage) – great extensible image caching, good performance
	(2)[Fast Image Cache](https://github.com/path/FastImageCache) – new competitor for SDWebImage, claims to get the most out of image caching performance

D. **Data Persistence**
	(1)[CoreData](https://developer.apple.com/library/mac/documentation/cocoa/conceptual/coredata/cdProgrammingGuide.html)
	(2)[MagicalRecord](https://github.com/magicalpanda/MagicalRecord) – great wrapper for CoreData, makes it easy for everybody to use it
	(3)[fmdb](https://github.com/ccgus/fmdb) –  for people that want just a SQLite wrapper

E. **Keychain**
	(1)[SSKeychain](https://github.com/soffes/sskeychain)

F. **Social**
	(1)[Facebook SDK + documentation](https://github.com/facebook/facebook-ios-sdk)
	(2)[ShareKit](https://github.com/ideashower/ShareKit)


#### 3.Media
A. **Video**
	(1)[VLCKit](https://wiki.videolan.org/VLCKit/) – ObjC wrapper for libvlc, the famous tool for video playback, playlists, streaming and transcoding from VLC
	(2)[iOS VideoKit](http://www.binpress.com/app/ios-videokit/1828) – video playing and streaming framework
	
......

[原文地址](http://bpoplauschi.wordpress.com/2013/11/06/worthy-ios-libraries/)
