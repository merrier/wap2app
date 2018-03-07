# wap页调起app流程

### 背景

- 调起过程中可能会存在app未安装情况（<b>目前还没有有效的方法能检测到是否安装了APP</b>），所以作为fallback。在需要调起APP的地方，都是需要添加下载app逻辑的
- 现有浏览器五花八门，操作系统也有ios和android之分，所以从wap页调起app的具体方案会根据用户的手机情况进行调整
- 我们只会讨论逻辑，并不会有详细代码，同时需要你具备一定的概念基础，知道universal link和scheme这些的用途

#### 一些知识点

* <b>universal link</b>：https://blog.branch.io/how-to-setup-universal-links-to-deep-link-on-apple-ios/
* <b>scheme</b>：http://www.voidcn.com/article/p-ntfhprrk-wk.html

### 约定

根据我们上面的描述，wap页调起app需要在调起的同时添加下载逻辑作为fallback，所以作如下约定：

* wap2app：代表调起
* appDownload：代表下载
* nativeLink：等同于scheme

而调起和下载的逻辑又会由于操作系统和浏览器而有所不同，我们稍后会详细介绍这两部分的实现；同时，我们将根据操作系统对大体上的实现步骤分别进行描述

### IOS

#### 微信浏览器

微信中，下载短链会302到应用宝，同时universal link会302到snssdk143://xxx，而微信会屏蔽snssdk143://这种非http协议，由于两者时间很短，几乎同时，似乎对短链的302也屏蔽的（不太了解机制），所以两者时间上要有一定的间隔。
> 由于下载短链会在当前页面打开应用宝页面，从而将不会执行后面的代码，所以先进行跳转：

```js
wap2app(params);
setTimeout(function () {
    appDownload(params);
}, 1000)
```

#### 其他浏览器

wap中下载短链会呼出apple store, wap2app的时候如果用户没有安装，会alert一个错误，导致下载短链无法执行，因此必须先执行下载

```js
appDownload(params);
setTimeout(function () {
    wap2app(params);
}, 900);
```

> setTimeout的时间要小于等于1000ms，这样safari中才不会出现类似于"确定打开某APP"的弹窗

### Android

#### QQ浏览器

根据抓包显示，QQ浏览器会在发起调起请求之后再发起一次"包装"后的请求，导致一个bug的出现：用户在调起app之后会被强制返回到QQ浏览器，然后弹出下载弹窗；所以QQ浏览器需要将下载的逻辑延迟执行一定时间，具体方案为：首先尝试调起app，延迟执行下载，如果调起未成功，说明用户并没有安装，此时将执行下载逻辑：

```js
var loadDateTime = new Date();
    setTimeout(function () {
        var timeOutDateTime = new Date();
        if (!loadDateTime || timeOutDateTime - loadDateTime < 510) {
            appDownload(params);
        }
    }, 500);
    wap2app(params);
```

> 这里500ms是固定的，所以会造成一个弊端：如果用户没有安装app，下载的逻辑执行将有一定时间的延迟，也就会出现大概500ms的"假死"状态，曾经尝试通过pageshow和pagehide事件进行优化，无奈浏览器对于这两种方法的判定机制相差太多，同时有些浏览器已经不支持两种方法，所以目前没有完美的解决方案。。

#### 其他浏览器

和IOS不同，Android中的大部分浏览器都是很"规矩"的执行JS代码，所以我们可以避免上面QQ浏览器中的弊端，将调起的逻辑放在下载的前面：

```js
wap2app(params);
appDownload(params);
```

### 调起

大体上来讲，IOS调起采用universal link的方式，Android调起采用scheme的方式，但是根据浏览器的不同会有其他兼容处理

#### IOS

根据ios版本分为两种情况：ios9及以上，ios9以下；同时由于QQ浏览器自身存在问题，所以也需要区别对待

- <b>QQ浏览器</b>：历史总是惊人的相似，又是QQ浏览器，ios下的QQ浏览器跳转时会出现白屏，所以不采用universal link方式
- <b>ios9及以上</b>：尝试用universal link方式打开，如果有universal link，就用universal link打开
- <b>ios9以下</b>：在iframe中调起scheme

#### Android

安卓的兼容性问题比较少，统一在iframe中打开nativeLink

#### 总结一下

最后总结一下调起的逻辑（android的坑比较少，主要是ios和QQ浏览器）：

- Android:

```js
openAppInIframe(params.nativeLink)
```

- ios9及以上&&非QQ浏览器

```js
location.href = params.universalLink || deepLink
```

- QQ浏览器&&ios版本≥9

```js
location.href = params.nativeLink
```

- QQ浏览器&&ios版本<9

```js
openAppInIframe(params.nativeLink)
```

> openAppInIframe方法如下所示：

> ```js
> var openAppInIframe = function(src) {
>     $('#app_iframe').remove();
>     $("body").append("<iframe id='app_iframe' src='"+src+"' style='display:none'></iframe>");
> };
> ```

### 下载

下载都是通过页面重定向到短链来实现；但是由于ios中有app store，所以ios和android的下载逻辑也会有所不同；同时，微信对短链会进行强制处理，所以也需要区别对待：

- ios中通过短链下载，但是微信中会被302到微下载（应用宝），通过微下载调起app store；而其他浏览器会直接302到app store；但是这都是浏览器自己的处理，对于我们而言代码是一样的：

```js
location.href = params.downloadLink
```

- Android&&微信，短链会302到微下载（应用宝），我们可以在下载链接中附加scheme，从而用户下载完毕之后可以调起app，同时打开指定页面：

```js
var invokeUrl = appendQuery(params.downloadLink, 'android_scheme=' + encodeURIComponent(params.nativeLink))
location.href = invokeUrl
```

- Android&&非微信，尝试调起应用宝，否则跳转到短链，302到自己的包地址：

```js
setTimeout(function () {
    $('body').append('<iframe id=\'app_dl_iframe\' src=\'' + params.yybHref + '\' style=\'display:none\'></iframe>')
    setTimeout(function () {
        $('iframe#app_dl_iframe').remove()
        location.href = params.fallback
    }, 1500)
})
```

### 其他问题

到这里还没有结束[摊手]，还可能遇到其他零碎bug：

#### ios9以下有时候universal link不起作用

UniversalLink在跳转到app后，会在浏览器右上角有一个带有网页URL的标记，用户如果点击后，就会跳到safari浏览器中打开页面，而此时，ios会记录下来用户针对这种url的点击默认行为是采用safari打开。之后，在微信中点击，ios就不再尝试UniversalLink，而是采用safari，进而被微信阻止。
重置回允许用户UniversalLink调起的方法，是safari访问标记中的URL，并在列表页顶部下滑一下屏幕，会出现打开某APP的提示——这个功能是safari提供的功能，此时点击打开后，就能调起App，并恢复UniversalLink的调起方式。另外一种恢复方案见http://stackoverflow.com/questions/32729489/how-can-i-reset-ios-9-universal-linking-settings 。目前，还没有找到能程序恢复的手段。。。

#### universal link 与 当前网址 必须是不同的域名

官方给出的解释：

> When a user is browsing your website in Safari and they tap a universal link to a URL in the same domain as the current webpage, iOS respects the user’s most likely intent and opens the link in Safari. If the user taps a universal link to a URL in a different domain, iOS opens the link in your app.

### 扩展阅读

下面这些关于调起的文章写得很不错，推荐阅读：

* [唤醒 App 的那些事](http://www.siyuweb.com/javascript/2533.html)
* [Deep Linking：从浏览器调起 APP](http://harttle.land/2017/12/24/launch-app-from-browser.html)