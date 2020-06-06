---
layout: post
title:  "iOS Universal Link"
date:   2020-06-03 07:02:48 +0800
tags: tech note
img: 
author: wving5
describe: 
---

突然丢过来的一件事，满打满算也花了小半天在上面，体力活，记之。

### #需求

wx 内 H5 直跳 native。 问题丢过来的时候，Leader 已经给了方向 `universal link`，体验舒肤佳

### #围观下前人踩过的坑
几个 18/19 年的帖子说 wx 开始着手封了 universal link，将信将疑。
SO 上有12.2 universal link 失效的提问
反正先搞起来再说...


**靠谱信息均来自官方文档，提炼要点如下**

### #前置条件
1. 合法 SSL 域名一只 (含机器)
2. iOS 9+ (含)

### #操作流程
1. Capabilities 里修改 entitlements, App 关联上域名。支持通配符`*`，~~即1对多~~ 匹配当前域名的所有子域名
2. 构造一个 manifest 名为 apple-app-site-association (json)，作用就好比是当前域名跳转 App 的路由表。具体格式见官网。 
	* 注意键值有两套: 有一套新的; 旧的兼容 iOS12 及以下。按需使用旧的，支持APL定义的通配符 `NOT ? *`
	* 每个关联 App 的域名(子域名)都要配置一次。 APL注: ````If your site uses different subdomains (such as example.com, www.example.com, and support.example.com) each requires its own entry in the Associated Domains Entitlement, and each must serve its own apple-app-site-association file.````
	* 该文件可使用 SSL 通过 GET 方法访问到。即`https://domain/apple-app-site-association`和`https://domain/.well-known/apple-app-site-association` APL注: `You must host the file using https:// with a valid certificate and without using any redirects.`
3. 验证 manifest
	* https://search.developer.apple.com/appsearch-validation-tool/ 较慢，未上线的应用 Link to Applicatino 会显示为 Error no apps with domain entitlements。APL 注: `The entitlement data used to verify deep link dual authentication is from the current released version of your app. This data may take 48 hours to update.`
	* 设备上安装好你的App，在 邮件/备忘录 中粘贴关联链接，长按有"在 \*\*\* APP中打开"的选项。或者相机扫描关联链接的二维码，会弹出"始终在 \*\*\* 中打开此二维码"的选项

### #App 内处理
对应 AppDelegate 中处理路由。雷同 URL scheme

### #自己踩的坑
1. 更改 manifest 需要删除 App 再装才能立即生效 (文档有写,实操时忘了)。APL 注: 
`After an app successfully associates with a domain, it remains associated until the app is **deleted** from the device. During development, delete your app from your testing device each time you update the association file to immediately see your changes.`

2. 到底什么时候才能"跳转"
	* 根据 APL 的描述，跳转的动作跟用户的"Tap" (符合用户认知的动作)是绑定的。贯彻了"不让用户感到懵x"的 APL 哲♂学 (颇有种解读 xxtv 新闻的感觉)
	* 用户点了之后，跳转 App 的优先级高于 web 容器，效果等同 `openURL`。 
	APL 注: 
	`When users **tap or click** a universal link, the system redirects the link directly to your app without routing through Safari or your website. `
	APL 又注: 
	`With universal links, users open your app when they **tap** links to your website within **Safari and WKWebView**, and links that result in a call to open(_:options:completionHandler:) in iOS and tvOS or a call to open(_:withApplicationAt:configuration:completionHandler:) in macOS, such as those that occur in **Mail, Messages, and other apps**.`

	* **在 Web 容器里**，跳转的另一个隐藏条件是"跨域"。
	APL 注: 
	`When a user browses your website in Safari and taps a universal link in the **same domain**, the system opens that link in Safari, respecting the user’s most likely intent to continue within the browser. If the user taps a universal link in a **different domain**, the system opens the link in your app.` 没错，还是那股熟悉的味道。
	同样的，APL 又又又注: `If your app uses open(_:options:completionHandler:) in iOS and tvOS or open(_:withApplicationAt:configuration:completionHandler:) in macOS to open a universal link to your website, the link does not open in your app.`
	说人话就是需要一个"跳板"，不管是一条 twitter，还是一个 blog，还是你自己的 html/js...这件事我也是看了文档没放心上，玩了下 moblink 的 demo 才意识到。。。~~APL 你丫重要的事情能不能加红加大加粗~~

好。打完收工。 

*（ 没想到写这么一篇水文花了我1个小时......一脸问号*

*（ 没错，好像通篇也没 wx 什么事儿*。我大胆~~(瞎JB)~~猜测 WKWebview 首先是独立进程，openURL 又比 web 容器的 navigation 优先级高，理论上APL不给你拦截的机会你就拦截不到啦，臭流氓哈哈哈(立了个旗)。

<br/>

----

### 2020-06-06 补充:

* Web容器里的“跨域”可以是跨**子域名**（实测至少二级域名没问题

* wx 1.8.6+ sdk 强行要求提供 universal link 了 （这旗子倒的有点快。。如有所忌惮，可以给它独立提供一个 domain。针对白名单当然是痴人妄想。

* 覆盖安装时，查看设备 log 可能会出现以下进程 MobileSMS,SpringBoard,mediaserverd,appstored,duetexpertd 打印相关的 request 
````
默认	17:07:47.916811+0800	SpringBoard	Requested application <private> has policy 0, associated category:DH1009 associated sites:(null) equivalent bundle identifiers:com.demo.testUniversalLink
````
* 上面 log 所谓的这个 `request`，在一定时间内会直接命中设备上的缓存。超出一定时间，安装 App 的行为会触发设备上所有 associated domain 缓存的刷新。(抓包会看到大量请求)
	* 如果覆盖安装时，Associated Domains 有修改过，则会重新去**修改后的域名**查询一次。即没修改的域名，不会再次去查询。
	* 如果覆盖安装时，Associated Domains 没有改变，则会看缓存是否失效，再去查询。( 具体缓存失效时间未知，貌似24小时也行，12小时也行，你猜...
	* 至于从 AppStore 安装更新，其行为就不得而知了。也许同 testflight? 懒得去试了。 ( 有知道的同学可以给本人提 issue 更正

综上所述，除非你预先**一次性关联了多个域名**。否则还是不太容易去控制 Associated Domains 的切换，因为刷新的时机/条件，都不是可以随意掌控的。强行要求用户删除重装 App / 或者明天再打开试试，有点太过硬核了。
