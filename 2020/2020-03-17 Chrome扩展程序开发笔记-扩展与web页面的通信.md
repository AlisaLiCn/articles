# 可参考的资料
- [官方开发文档](https://developer.chrome.com/extensions) ：目前全英文，需要翻**墙。
- [Chrome插件开发全攻略](https://github.com/sxei/chrome-plugin-demo)：github上star挺多的一个repo，开发前可先过一下。

搜索到的资料可以作为参考，实际开发应以官方最新文档为准。

# 文件结构
- `manifest.json`配置文件：**必须**，[详细说明文档](https://developer.chrome.com/extensions/manifest)
- 其他`JavaScript`、`HTML`、`CSS`文件：根据功能和个人喜好，进行增减以及结构调整。

开发扩展程序与WEB开发类似，主要应用的技术为`JavaScript`、`HTML`、`CSS`等，再根据功能，调用chrome提供的各类API。

最终的文件结构可以类似这样：
```
my-extension
  - manifest.json  // required
  pages
    - background.html
    - popup.html
  icons
    - logo_16.png
    - logo_48.png
    - logo_128.png
  js
    - background.js
    - content.js
    - inject.js
    - popup.js
  css
    - popup.css
```


开发时为防止后续功能繁多，我使用了Vue-Cli初始化项目，并使用插件[vue-cli-plugin-chrome-ext](https://github.com/superoo7/vue-cli-plugin-chrome-ext)修改文件结构，再根据自己需要修改vue.config.js以修改webpack的一些配置。


# web页面与扩展程序之间通信
## 官方相关的文档
- 消息通信 [从web页面发送消息](https://developer.chrome.com/extensions/messaging#external-webpage)
- 外部连接相关 [externally_connectable](https://developer.chrome.com/extensions/manifest/externally_connectable)
- 监听事件 [onMessageExternal用法](https://developer.chrome.com/extensions/runtime#event-onMessageExternal)

## 权限声明
首先，需要在`manifest.json`中，配置`externally_connectable`字段，来声明哪些web应用可以通过后面的方式，与扩展程序建立连接。在matches字段中，注明需要通信的web网站清单。
```
"externally_connectable": {
  "matches": ["*://*.example.com/*"]
}
```
matches数组的每一项为URL字符串。且URL值必须要包含到二级域名级别。

`externally_connectable`字段，还可以声明`ids`字段，来指定需要通信的其他Chrome应用或者其他扩展程序。

## 通信方式
从Web页面向扩展程序发送消息，可以使用`chrome.runtime.sendMessage`,需要传入扩展程序的ID，回调函数可以接收Chrome扩展响应的消息：
```
const extensionId = 'deakpiepsidfpdfdioffidjfifjtest' // 想要与之通信的扩展ID

// 向Chrome扩展发送请求
chrome.runtime.sendMessage(extensionId, {
  event: 'requestEvent',
  data: requestData,
}, (response) => {
    console.log('res data', response)
}
```

在Chrome扩展内使用`chrome.runtime.onMessageExternal.addListener`监听消息，接收后用`sendResponse`发送响应消息：

```
chrome.runtime.onMessageExternal.addListener(async (request, sender, sendResponse) => {
  if (request.event == 'requestEvent') {
    const res = 'res message'
    sendResponse(res)
  }
})

```

# 监听和修改Web请求
文档位置：[webRequest](https://developer.chrome.com/extensions/webRequest)

依然是要先进行权限声明：
```
  {
      "name": "My extension",
      ...
      "permissions": [
        "webRequest",
        "*://*.google.com/"
      ],
      ...
  }
```
Chrome提供了一系列的事件来监听Web请求生命周期的各个阶段，其中`onBeforeSendHeaders`事件被触发的阶段，可以用来增、删、改HTTP请求的headers。也可以在这个事件中取消请求。

用法， 修改之后return新的配置：
```
chrome.webRequest.onBeforeSendHeaders.addListener((details) => {
  const { requestHeaders } = details
  
  requestHeaders.push({
      name: 'x-request-xx',
      value: '...'
  })
  
  return { requestHeaders: requestHeaders }
})
```


# cookies操作
使用cookie相关的API，需要先在`manifest.json`中的`permissions`字段进行权限声明，要访问cookies的host也要一并声明：

```
{
  "name": "My extension",
  ...
  "permissions": [
    "cookies",
    "*://*.google.com/"
  ],
  ...
}
      
```

Chrome共提供了5个方法和1个监听事件，用于cookie的操作和对cookie变化的监听：

**方法**：
- get: chrome.cookies.get(object details, function callback)
- getAll
- set
- remove
- getAllCookieStores

**事件**：
- onChanged

## 获取指定名称的cookie

```
chrome.cookies.get({
  url: 'https://example.com',
  name: 'token',
}, (cookie) => {
  console.log(‘token: ’, cookie.value) // cookie为获取到的cookie对象
})
```

## 设置cookie
```
 chrome.cookies.set({
    url: 'http://example.com',
    name: 'x-request-xx',
    value: 'value...',
  }, (cookie) => {
    console.log('set cookie', cookie)
  })
```

# 总结
Chrome扩展与web之间通信，还有一些其他方式，但个人认为按照官方文档中推荐的方式比较容易使用。

在实际开发时，如果担心扩展的id值发生变化，可以在`background.js`中利用`chrome.runtime.id`拿到当前扩展的id，通过`content.js`发送请求事件进行获取，最后可以以元素属性的值嵌入到HTML中，这样在WEB中就可以拿到啦~