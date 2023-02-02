---
title: qiankun构成与原理
date: 2023-02-02T10:12:49Z
url: https://github.com/gwuhaolin/blog/issues/36
tags:

---

## 功能介绍&模块拆解

在聊原理之前先了解下qiankun提供的能力，一句话介绍qiankun功能：
**能根据路由自动调度子应用并实现沙箱（主子、子子应用之间的JS和CSS）隔离。**

举个例子：在主应用里注册两个子应用A&B，

```jsx
import { registerMicroApps } from 'qiankun';

registerMicroApps([
  {
    name: 'A',
    entry: 'https://baidu.com',
    container: '#yourContainer',
    activeRule: '/baidu',
  },
  {
    name: 'B',
    entry: 'https://google.com',
    container: '#yourContainer2',
    activeRule: '/google',
  },
]);
```

在pathname是_/baidu_时加载和运行子应用A，如果路由pushState到了_/google_时会unmount子应用A再加载和运行子应用B。

从上面的例子可以看出qiankun提供的能力可以划分为3大部分：
- [single-spa](https://single-spa.js.org/): 绑定路由的子应用调度，监听路由变化根据当前路由去调度对应的子应用
- [import-html-entry](https://github.com/kuitos/import-html-entry): 加载和运行子应用的HTML entry
- [sanbox](about:blank):子应用的隔离，又分为JS的隔离和CSS的隔离


<img width="681" alt="image" src="https://user-images.githubusercontent.com/5773264/216295142-ca8af020-a4c1-4944-8956-37f09cde5b63.png">

下面分别拆解来分析原理。

## Single-spa

一句话介绍single-spa：**根据路由变化做子应用调度（子应用生命周期管理）**

<img width="712" alt="image" src="https://user-images.githubusercontent.com/5773264/216295303-62b3d1d1-4c6c-4acd-ba5e-ae8218591b35.png">


以单个子应用的生命周期来看流程如下：

<img width="611" alt="image" src="https://user-images.githubusercontent.com/5773264/216295373-d1f8affc-b2dc-4616-a68b-59b97991ab56.png">


### 注册子应用

```js
singleSpa.registerApplication({ // 注册一个子应用,注册其他子应用同理
    name: 'taobao', // 子应用名,需要唯一
    app: () => System.import('taobao'), // 如何拿到子应用的生命周期，这里demo用的System，qiankun实际上不是基于System
    activeWhen: '/appName', // url 匹配规则，表示啥时候开始走这个子应用的生命周期
    customProps: {} // 自定义 props，从子应用的 bootstrap, mount, unmount 回调可以拿到
})
```


### 子应用调度

子应用在自己的入口 js导出了生命周期函数钩子，那在切换路由时子应用A的unmount会执行，子应用B的mount会执行。针对React体系的子应用通常在各生命周期中做如下事情：

```js
/**
* bootstrap 只会在微应用初始化的时候调用一次，下次微应用重新进入时会直接调用 mount 钩子，不会再重复触发 bootstrap。
* 通常我们可以在这里做一些全局变量的初始化，比如不会在 unmount 阶段被销毁的应用级别的缓存等。
*/
export async function bootstrap() {
  console.log('react app bootstraped');
}

/**
* 应用每次进入都会调用 mount 方法，通常我们在这里触发应用的渲染方法
*/
export async function mount(props) {
  ReactDOM.render(<App />, props.container ? props.container.querySelector('#root') : document.getElementById('root'));
}

/**
* 应用每次 切出/卸载 会调用的方法，通常在这里我们会卸载微应用的应用实例
*/
export async function unmount(props) {
  ReactDOM.unmountComponentAtNode(
    props.container ? props.container.querySelector('#root') : document.getElementById('root'),
  );
}
```


### Html-entry-loader

一句话介绍html-entry-loader：**把HTML当作子应用的manifest，加载和执行其中的JS拿到JS导出的模块，加载拿到其中的CSS**。

<img width="971" alt="image" src="https://user-images.githubusercontent.com/5773264/216295684-4de11506-b1bc-4c1c-bc2f-e60ffa075096.png">


特别说明：
- 外链的JS、CSS会被fetch下来以方便执行或提取
- 默认不带沙箱隔离

```js
importHTML('https://xxx.com/subApp.html')
    .then(res => {
        res.execScripts(proxy = window /*传入沙箱，默认全局window*/).then(exports => {
            console.log(exports); // 导出的JS模块，子应用生命周期从这里拿
        });
        res.getExternalStyleSheets() // Promise<string[]> 获取导出的CSS内容
});
```

### 为啥选择HTML作为子应用manifest的描述载体

主应用需要拿到运行一个子应用所需要的信息，包括：JS、CSS、mountId。
除能用HTML作为载体之外，还有一种方式是通过一个JSON来描述，类似这样：

```json
{
  "version": "1.3.1",
  "js": ["main.js", "common.js"],
  "css": ["main.css"],
  "publicPath": "https://cdn.cn/appXXX",
  "mountId": "root"
}
```

这两种方式各有千秋：

优点 | 缺点
-- | --
完全兼容原来就是以HTML方式输出的网页更灵活，支持HTML中内敛的JS、CSS等复用已有的公知的HTML规范作为协议，而不是新创造一种协议 | 存在信息冗余，传输体积大于JSON解析HTML耗时大于解析JSON
协议更简单，传输和解析更快 | 操作了一种新协议、规范，需要接入方按此新规范去改造适配，推广成本上升不够灵活


选择HTML最主要的原因是 “复用已有的公知的HTML规范作为协议，而不是新创造一种协议”，因为一个子应用可能需要在多个站点投放、或者需要独立运行。


### 如何提取HTML中的JS、CSS

通过正则表达式提取，源码里枚举了各种可能的情况下的正则提取式：

```js
const ALL_SCRIPT_REGEX = /(<script[\s\S]*?>)[\s\S]*?<\/script>/gi;
const SCRIPT_TAG_REGEX = /<(script)\s+((?!type=('|")text\/ng-template\3).)*?>.*?<\/\1>/is;
const SCRIPT_SRC_REGEX = /.*\ssrc=('|")?([^>'"\s]+)/;
const SCRIPT_TYPE_REGEX = /.*\stype=('|")?([^>'"\s]+)/;
const SCRIPT_ENTRY_REGEX = /.*\sentry\s*.*/;
const SCRIPT_ASYNC_REGEX = /.*\sasync\s*.*/;
const SCRIPT_NO_MODULE_REGEX = /.*\snomodule\s*.*/;
const SCRIPT_MODULE_REGEX = /.*\stype=('|")?module('|")?\s*.*/;
const LINK_TAG_REGEX = /<(link)\s+.*?>/isg;
const STYLE_TAG_REGEX = /<style[^>]*>[\s\S]*?<\/style>/gi;
const STYLE_TYPE_REGEX = /\s+rel=('|")?stylesheet\1.*/;
const STYLE_HREF_REGEX = /.*\shref=('|")?([^>'"\s]+)/;
```

[JS、CSS提取逻辑完整源码链接](https://github.com/kuitos/import-html-entry/blob/master/src/process-tpl.js)


### 如何执行子应用

```js
function getExecutableScript(scriptSrc, scriptText, proxy, strictGlobal) {
	const sourceUrl = isInlineCode(scriptSrc) ? '' : `//# sourceURL=${scriptSrc}\n`;

	// 通过这种方式获取全局 window，因为 script 也是在全局作用域下运行的，所以我们通过 window.proxy 绑定时也必须确保绑定到全局 window 上
	// 否则在嵌套场景下， window.proxy 设置的是内层应用的 window，而代码其实是在全局作用域运行的，会导致闭包里的 window.proxy 取的是最外层的微应用的 proxy
	const globalWindow = (0, eval)('window');
	globalWindow.proxy = proxy;
	// TODO 通过 strictGlobal 方式切换 with 闭包，待 with 方式坑趟平后再合并
	return strictGlobal
		? `;(function(window, self, globalThis){with(window){;${scriptText}\n${sourceUrl}}}).bind(window.proxy)(window.proxy, window.proxy, window.proxy);`
		: `;(function(window, self, globalThis){;${scriptText}\n${sourceUrl}}).bind(window.proxy)(window.proxy, window.proxy, window.proxy);`;
}
```


#### with语句

with的初衷是为了避免冗余的对象调用：

```js
foo.bar.baz.x = 1;
foo.bar.baz.y = 2;
foo.bar.baz.z = 3;

with(foo.bar.baz){
    x = 1;
    y = 2;
    z = 3;
}
```

[使用with语句有很多问题，详情见](https://swordair.com/javascript-with-statement-in-depth/)


#### 如何执行子应用的JS

```js
const evalCache = {};

export function evalCode(scriptSrc, code) {
	const key = scriptSrc;
	if (!evalCache[key]) {
		const functionWrappedCode = `window.__TEMP_EVAL_FUNC__ = function(){${code}}`;
		(0, eval)(functionWrappedCode);
		evalCache[key] = window.__TEMP_EVAL_FUNC__;
		delete window.__TEMP_EVAL_FUNC__;
	}
	const evalFunc = evalCache[key];
	evalFunc.call(window);
}
```

- 用eval去执行以字符串形式保存的子应用JS
- 借助window.__TEMP_EVAL_FUNC__记下子应用JS的执行结果，并缓存到evalCache中避免下次重复eval