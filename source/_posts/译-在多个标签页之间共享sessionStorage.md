title: 译-在多个标签页之间共享sessionStorage
date: 2016-09-09 09:37:12
tags:
- localStorage
- sessionStorage
- 共享
- memoryStorage
- 认证token
categories: 前端
---

原文：[Sharing sessionStorage between tabs for secure multi-tab authentication](https://blog.guya.net/2015/06/12/sharing-sessionstorage-between-tabs-for-secure-multi-tab-authentication/)

## 译者得er瑟
---

昨天，就在昨天，前端一同事提了一个问题：**我们的系统，用户重新开一个标签页，就要重新登录**。我当时觉得这怎么可能？结果现场一测，还真是，好尴尬！

今天抽了点时间网上查了查，才发现原来一直以为很简单的sessionStorage，还真埋了这么一颗雷。不过国外前辈也提出了一个解决方案，不仅如此，文章还把浏览器端保存数据的场景分析的很透彻，所以斗胆翻译了一下。

<!--more-->

## 原文翻译
---

#### tl;dr;

我实现了一种机制可以利用浏览器提供的sessionStorage或memoryStorageStorage的固有的安全性来实现用户身份认证，并且可以保证用户不需要每次新开一个标签页都重新登录。

#### 现有的浏览器存储机制

- **localStorage**：~5MB，数据永久保存直到用户手动删除
- **sessionStorage**：~5MB，数据只在当前标签页有效
- **cookie**：~4KB，可以设置成永久有效
- **session cookie**：~4KB，当用户关闭浏览器时删除（并非总能立即删除）

#### 安全的认证token保存

一些重要的系统会要求当用户关闭标签页时会话立刻到期。

为了达到这个目的，不仅绝对不应该使用cookies来保存任何敏感信息（例如认证token）。甚至session-cookies也无法满足要求，它在标签页关闭（甚至浏览器完全关闭）后还会持续存活一定时间。

（任何时刻我们都不应该只使用cookies，它还有其他很多问题需要讨论，例如CSRF）

这些问题就使得我们在保存认证token时应使用内存或sessionStorage。sessionStorage的好处是它允许跨多个页面保存数据，并且也支持浏览器刷新操作。这样用户就可以在多个页面之间跳转或刷新页面而保持登录状态。

Good。我们将token保存在sessionStorage，并在每次请求服务器时将token放在请求头中来完成用户的身份认证。当用户关闭标签页，token会立即过期。

#### 但多标签页怎么办？

即便是在单页面应用中也有一个很常见的情况，用户经常希望打开多个标签页。而此场景下将token保存在sessionStorage中将会带来很差的用户体验，每次开启一个标签页都会要求用户重新登录。没错，sessionStorage不支持跨标签页共享数据。

#### 利用localStorage事件来跨标签页共享sessionStorage

我利用localStorage事件提出了一种解决方案。

当用户新开一个标签页时，我们先来询问其它已经打开的标签页是不是有需要给我们共享的sessionStorage数据。如果有，现有的标签页会通过localStorage事件来传递数据到新打开的标签页中，我们只需要复制一份到本地sessionStorage即可。

传递过来的sessionStorage绝对不会保存在localStorage，从localStorage事件将数据中复制并保存到sessionStorage，这个流程是在同一个调用中完成，没有中间状态。而且数据是对应事件携带的，并不在localStorage中。（译者注：作者意图解释这个方案的安全性）

[在线例子](https://blog.guya.net/security/browser_session/sessionStorage.html)

点击“Set the sessionStorage”，然后打开多个标签页，你会发现sessionStorage共享了。

```javascript

// 为了简单明了删除了对IE的支持
(function() {

	if (!sessionStorage.length) {
		// 这个调用能触发目标事件，从而达到共享数据的目的
		localStorage.setItem('getSessionStorage', Date.now());
	};

	// 该事件是核心
	window.addEventListener('storage', function(event) {
		if (event.key == 'getSessionStorage') {
			// 已存在的标签页会收到这个事件
			localStorage.setItem('sessionStorage', JSON.stringify(sessionStorage));
			localStorage.removeItem('sessionStorage');

		} else if (event.key == 'sessionStorage' && !sessionStorage.length) {
			// 新开启的标签页会收到这个事件
			var data = JSON.parse(event.newValue),
					value;

			for (key in data) {
				sessionStorage.setItem(key, data[key]);
			}
		}
	});
})();

```
（译者注：上面的代码是我从在线demo中截取的，原文中并无提到）

#### 接近完美

我们现在拥有了一个几乎非常安全的方案来保存会话token在浏览器里，并支持良好的多标签页用户体验。现在当用户关闭标签页后能确保会话立即过期。难道不是么？

chrome和firefox都支持当用户进行“重新打开关闭的标签页”或“撤销关闭标签页”时恢复sessionStorage。F**k！（译者注：作者原文用的是“Damn it!”，注意到那个叹号了吗？）

safari在这个问题上处理是正确的，它并不会恢复sessionStorag（只测试了上述这三个浏览器）。

对用户而言，能够确定sessionStorag已经过期的方法是直接重新打开网站，而不是选择“重新打开关闭的标签页”。

除非chrome和firefox能够解决这个bug。（但我预感开发组会称其为“特性”）

即便存在这样的bug，使用sessionStorag依然要比session-cookies方案或其他方案要安全。如果我们希望得到一个更加完美的方案，我们就需要自己来实现一个内存的方案来代替sessionStorag。(onbeforeunload也能做到，但不是太可靠且每次刷新页面也会被清空。window.name也不错，但它太老了且也不支持跨域保护)

#### 跨标签页共享memoryStorage

这应该是唯一一个真正安全的实现浏览器端保存认证token的方法了，并且要保证用户打开多个标签页不需要重新登录。

关闭标签页，会话立即过期--这次是真真儿的。

这个方案的缺点是，**当只有一个标签页时**，浏览器刷新会导致用户重新登录。安全总是要付出点代价的，很明显这个缺点可能是致命的。

[在线例子](https://blog.guya.net/security/browser_session/memoryStorage.html)

设置一个memoryStorage，然后打开多个标签页，你会发现数据共享了。关闭所有标签页token会立即永久过期（memoryStorage其实就是一个javascript对象而已）。


```javascript

(function() {

	window.memoryStorage = {};

	function isEmpty(o) {
		for (var i in o) {
	  		return false;
	 	}
	 	return true;
	};

	if (isEmpty(memoryStorage)) {
		localStorage.setItem('getSessionStorage', Date.now());
	};

	window.addEventListener('storage', function(event) {
		if (event.key == 'getSessionStorage') {
			localStorage.setItem('sessionStorage', JSON.stringify(memoryStorage));
			localStorage.removeItem('sessionStorage');

		} else if (event.key == 'sessionStorage' && isEmpty(memoryStorage)) {
			var data = JSON.parse(event.newValue),
						value;

			for (key in data) {
				memoryStorage[key] = data[key];
			}
		}
	});
})();

```
