[TOC]

## 前言
目前，Chrome商店中大部分扩展程序都是基于MV2版本开发的。

Chrome浏览器从88版本开始支持MV3啦（即Manifest Version 3），目前查看官方文档时默认已经是MV3版本，Chrome商店在2021年1月也已经开始接收MV3的扩展提交。

本文是一份Chrome扩展开发从**MV2**向**MV3**迁移的备忘清单。

## 为什么需要迁移
两个原因：
- 使用MV3的扩展，在安全性、隐私性和性能方面将会得到增强;
- 和MV1一样，MV2会被逐步废弃。

不过到目前为止，对于MV2的废弃官方还没有给出具体的时间线，但会在MV3稳定支持后，留给开发者至少一年的时间来进行迁移。原文如下：
>While there is not an exact date for removing support for Manifest V2 extensions, developers can expect the migration period to last at least a year from when Manifest V3 lands in the stable channel. 

MV3虽然还没有完全稳定，但我们可以先了解两个版本之间的差异，在开发时尽量避免使用MV3不再支持的特性，减少未来迁移时的工作量。

## 修改项·checklist
以下是迁移至MV3时的一些修改项，目前开发接触到的部分中，**background向service worker的转变**以及**使用webRequest对web请求进行监听修改**这两个部分，相对变更较大，需要调整一定量的代码来完成迁移，其它的部分大多是配置上的变更，配合少量的代码修改即可。


### manifest版本号
升级第一步，把配置文件中的版本号设置为`3`。
```
// Manifest V3
"manifest_version": 3 // 版本号由2修改为3

```

### 从background pages到service workers
MV3中使用Service Worker替换了原来的背景页，这也是升级时变化比较大的部分。

关于Service Worker的详细内容可以参考[Service Worker简介](https://developers.google.com/web/fundamentals/primers/service-workers/)，这里只讨论扩展升级到MV3时需要做的变更。

**首先，在manifest.json中的变化：**
- 使用`background.service_worker`替换原来的`background.page`或`background.scripts`
- 文件路径配置由数组变为单字符串；
- service worker文件需置于扩展程序文件夹的根目录下，如果仍使用MV2中的相对路径就会报错，service worker无法成功注册。（留心官方文档提示：Service workers must be registered at root level: they cannot be in a nested directory.）
- 移除`background.persistent`字段

```
  // Manifest V2
  "background": {
   	"scripts": ["js/pages/background.js"]
   },



  // Manifest V3
  "background": {
    // Required
    "service_worker": 'background.js'
  },

```

**其次，要调整代码以保证能够在service worker中正常运行：**

两者最大的区别：MV2中的background scrips被成功注册后是一直运行着的，而MV3中的Service Worker 在不用时会被中止，并在下次有需要时重启。还需要注意的是Service Worker没有直接访问DOM的能力。

#### 用缓存替换全局变量
因为Service Worker并不是长期存在的，而是在浏览器会话中重复“启动-执行某些操作-终止”这样的过程，只有在需要的时候才会处于可用的状态，因此如果仍然使用全局变量存储某些数据，程序在重新启动时就会丢失。

在MV3中，可以利用storage API将需要的数据存储在缓存中，需要时从缓存中获取。

```
// MV2
let name = undefined;

chrome.runtime.onMessage.addListener(({ type, name }) => {
  if (msg.type === "set-name") {
    name = msg.name;
  }
});

chrome.browserAction.onClicked.addListener((tab) => {
  chrome.tabs.sendMessage(tab.id, { name });
});



// MV3
chrome.runtime.onMessage.addListener(({ type, name }) => {
  if (type === "set-name") {
    chrome.storage.local.set({ name });
  }
});

chrome.action.onClicked.addListener((tab) => {
  chrome.storage.local.get(["name"], ({ name }) => {
    chrome.tabs.sendMessage(tab.id, { name });
  });
});
```



#### 用alarms替换timers
setTimeout和setInterval在service worker中可能会失效，因为调度程序在service worker处于终止状态时会取消定时器的执行。

可以用扩展程序中的alarms API来代替定时器，以避免出现定时器失效的问题。

```
// MV2 timers
const TIMEOUT = 3 * 60 * 1000; // 3 minutes
window.setTimeout(() => {
  chrome.action.setIcon({
    path: getRandomIconPath(),
  });
}, TIMEOUT);



// MV3 => alarms
chrome.alarms.create({ delayInMinutes: 3.0 });

chrome.alarms.onAlarm.addListener(() => {
  chrome.action.setIcon({
    path: getRandomIconPath(),
  });
});
```


#### 绘制canvas
如果在背景页中有绘制canvas的场景（例如：用于展示或缓存资源），不要忘记service worker没有直接访问DOM的能力，但可以使用` OffscreenCanvas API`来做替换。

写法上使用`new OffscreenCanvas(width, height)`来代替 `document.createElement('canvas')`

```
// MV2
function buildCanvas(width, height) {
  const canvas = document.createElement("canvas");
  canvas.width = width;
  canvas.height = height;
  return canvas;
}



// MV3
function buildCanvas(width, height) {
  const canvas = new OffscreenCanvas(width, height);
  return canvas;
}
```

### 修改网络请求
在MV2中，我们使用`chrome.webRequest`相关的API来拦截和修改web请求；

在MV3中，需要使用新的`chrome.declarativeNetRequest`来代替。

新的API有以下区别：
- 由Chrome浏览器本身去计算和修改请求，而非像MV2中一样是javascript程序上的拦截和修改。
- 需要通过指定rules来实现请求的修改，浏览器会在匹配到符合规则的请求和操作时，按照rules中定义好的规则进行修改，程序中不再能直接查看请求的实际内容。

以修改web请求中的headers为例，

在MV2中我们可能会在请求发出之前，在监听事件中对headers做一些修改：
```
// MV2

chrome.webRequest.onBeforeSendHeaders.addListener(
details => {
  
  // some modify logic...
  
  return { requestHeaders: details.requestHeaders };
},
{ urls: ['<all_urls>'] },
['blocking', 'requestHeaders', 'extraHeaders']
);

```

在MV3中，首先要在manifest.json中指定rules文件，声明`declarativeNetRequest`权限：
```
// MV3

{
  "name": "My extension",
  ...

  "declarative_net_request" : {
    "rule_resources" : [{
      "id": "ruleset_1",
      "enabled": true,
      "path": "rules.json"
    }]
  },
  "permissions": [
    "declarativeNetRequest",
    "declarativeNetRequestFeedback", // 只在需要时声明即可
  ],
  "host_permissions": [ 
    "http://www.blogger.com/",
    "http://*.headers.com/" // 需要重定向或修改headers时需要声明对应的hosts，否则不需要
  ],
  ...
}
```
在rules.json文件中，定义修改规则，示例文件：

```
  {
    "id": 1,
    "priority": 1,
    "action": {
      "type": "modifyHeaders",
      "requestHeaders": [
        { "header": "token", "operation": "set", "value": "token value" },
      ]
      "responseHeaders": [
        { "header": "h1", "operation": "remove" },
        { "header": "h2", "operation": "set", "value": "v2" },
        { "header": "h3", "operation": "append", "value": "v3" }
      ]
    },
    "condition": { "urlFilter": "headers.com/123", "resourceTypes": ["main_frame"] }
  },
```

可以看到，一条规则由以下四个部分组成：
- id：规则的唯一id
- priority：规则的优先级
- condition：规则被触发的条件
- action：匹配到该条规则时进行的操作

`action`指定修改web请求的操作类型：
- block：阻塞请求
- redirect：重定向请求
- upgradeScheme
- allow
- allowAllRequests
- modifyHeaders：修改请求头

在处理请求时会根据rules里指定的优先级来，具体可以参考[Matching algorithm](https://developer.chrome.com/docs/extensions/reference/declarativeNetRequest/#matching-algorithm)这一节。

`rules.json`静态文件中已经启用(enabled)的规则可以通过`updateEnabledRulesets`API来更新，但rules在数量上有一定的限制，更新时需要注意，限制相关内容可以参考文档 [Global Static Rule Limit](https://developer.chrome.com/docs/extensions/reference/declarativeNetRequest/#global-static-rule-limit) 一节。

总的来说，新的API这种形式感觉相比MV2来说会更加复杂一些，这里只列出了我开发中用到的一部分内容，在迁移时根据业务场景，可以再仔细阅读下官方文档。

### Host permissions
MV3对权限声明进行了拆分，新增`host_permissions`字段单独声明host访问权限，其他权限的声明仍然在`permissions`字段下。
```
// Manifest V2
"permissions": [
  "tabs",
  "bookmarks",
  "http://www.blogger.com/",
],



// Manifest V3
"permissions": [
  "tabs",
  "bookmarks"
],
"host_permissions": [
  "http://www.blogger.com/",
  "*://*/*"
],
```


### 合并Action API
MV2中的`browser_action`和`page_action`，到MV3中被合并到了单独的`action` API中，涉及到两个场景的修改：

一、manifest.json中配置的修改

```
// manifest.json

// Manifest V2
{
  "browser_action": { … },
  "page_action": { … }
}


// Manifest V3
{
  "action": { … }
}

```
二、在javascript中使用API时的修改：
```
// background.js

// Manifest V2
chrome.browserAction.onClicked.addListener(tab => { … });
chrome.pageAction.onClicked.addListener(tab => { … });

// Manifest V3
chrome.action.onClicked.addListener(tab => { … });
```


### Content security policy
扩展程序的内容安全策略(CSP)声明方式变更，由字符串改为对象。

```
// Manifest v2
"content_security_policy": "..."


// Manifest v3
"content_security_policy": {
  "extension_pages": "...",
  "sandbox": "..."
}
```

## 其它变更
- 远程托管的代码(Remotely hosted code)：MV3中不再支持非扩展包内的js代码的执行，包括从远程服务器获取的js文件，和程序运行时传给`eval`的代码字符串，因此所有代码都需要打包在程序包中。
- 剩下的我这里没太常用，可以参考文档过一下，升级的时候如果有问题，在开发者工具中也会有对应的错误提示，再对照文档修改即可。


## 参考资料
- [Chromium Blog](https://blog.chromium.org/2020/12/manifest-v3-now-available-on-m88-beta.html)
- [Migrating to Manifest V3](https://developer.chrome.com/docs/extensions/mv3/intro/mv3-migration/)
- [Manifest V3 migration checklist](https://developer.chrome.com/docs/extensions/mv3/mv3-migration-checklist/)
- [Migrating from background pages to service workers](https://developer.chrome.com/docs/extensions/mv3/migrating_to_service_workers/)
- [chrome.declarativeNetRequest API](https://developer.chrome.com/docs/extensions/reference/declarativeNetRequest/#method-setExtensionActionOptions)

