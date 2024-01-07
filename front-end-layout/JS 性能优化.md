
性能优化的目的，就是为了提供给用户更好的体验，这些体验包含这几个方面：**展示更快**、**交互响应快**、**页面无卡顿情况**。

更详细的说，就是指，在用户输入 url 到站点完整把整个页面展示出来的过程中，通过各种优化策略和方法，让页面加载更快；在用户使用过程中，让用户的操作响应更及时，有更好的用户体验。

对于前端工程师来说，要做好性能优化，需要理解浏览器加载和渲染的本质。理解了本质原理，才能更好的去做优化。

## Yahoo 性能优化军规

雅虎军规是雅虎的开发人员在总结了网站的不合理部分后，提出的优化网站性能提高的一套方法规则，非常适合初学者绕过这些坎。

![[Pasted image 20231110155116.png]]

## 浏览器运行周期与优化相关部分

![[Drawing 2023-11-06 11.34.18.excalidraw |100%]]

### 面试题：基于 window. performance 衡量一个网络请求，如果请求慢，如何判断这个请求是前端慢还是服务慢还是其他原因？

前端慢的原因是发送请求的中间处理，比如 Axios 拦截器、AJAX 组装逻辑…当这些中间处理某一环的依赖崩了或者某一环花费的时间过长。服务慢的原因是服务器接收到请求，中间处理的过程花费太多时间，返回慢。当我们发现前端和后端都不慢，那么问题就可能出在建立通路上，TCP 在建立的过程中一直不成功，超时导致失败。（HTTP 缓存，导致本地与线上数据不一致）

Navigation Timing API
> 回调浏览器运行过程中的每个关键节点

1. navigationStart
    表示从上一个页面开始，==要进行跳转==的最初钩子

2. unloadEventStart  /  End
	表示上一个页面 unload 的时间点
   
3. redircetStart  /  End
	页面发生重定向事件，开始 / 结束的时间
   
4. fetchStart  /  End
	浏览器准备好使用 HTTP 请求抓取文档的时间（准备时间，并没有抓取）
   
5. domainLookupStart  /  End
	HTTP 开始 / 结束，重新建立连接的时间
   
6. connectStart  /  End
	开始 / 结束，使用连接的时间 => 纯数据连接时间
   
7. secureConnectionStart  /  End
    HTTPS 加密连接，开始 / 结束的时间

8. requestStart  /  End
    请求花费的时间

9. responseStart  /  End
    返回花费的时间

10. domLoading  |  domInteractive  |  domContentLoadedEventStart  /  End  |  domComplete
     开始渲染 DOM 树的时间、完成 DOM 解析的时间、网页内部内容开始 / 结束加载时间、完成 DOM 渲染

11. loadEventStart  /  End
     开始 / 结束，事件循环

使用回调：
```html
// index.html

<script>
javascript: (() => {
	var perfData = window.performance.timing
	var pageLoadTime = perfData.loadEventEnd - perfData.navigationStart
	// 结果为从上一个页面跳转到当前页面，当前页面所有资源加载完成的耗时，单位毫秒
	console.log('耗时：' + pageLoadTime + 'ms')
})()
</script>
```

## 性能优化指标

-  **`First Paint 首次绘制（FP）`**
    这个指标用于记录页面第一次绘制像素的时间，如显示页面背景色。
	| FP 不包含默认背景绘制，但包含非默认的背景绘制。

-  **`First contentful paint 首次内容绘制 (FCP)`**
    LCP 是指页面开始加载到最大文本块内容或图片显示在页面中的时间。如果 FP 及 FCP 两指标在 2 秒内完成的话我们的页面就算体验优秀。

-  **`Largest contentful paint 最大内容绘制 (LCP)`**
	用于记录视窗内最大的元素绘制的时间，该时间会随着页面渲染变化而变化，因为页面中的最大元素在渲染过程中可能会发生改变，另外该指标会在用户第一次交互后停止记录。官方推荐的时间区间，在 2.5 秒内表示体验优秀

-  **`First input delay 首次输入延迟 (FID)`**
	首次输入延迟，FID（First Input Delay），记录在 FCP 和 TTI 之间用户首次与页面交互时响应的延迟。

-  **`Time to Interactive 可交互时间 (TTI)`**
	首次可交互时间，TTI（Time to Interactive）。这个指标计算过程略微复杂，它需要满足以下几个条件：
	
	  1. 从 FCP 指标后开始计算
	  2. 持续 5 秒内无长任务（执行时间超过 50 ms）且无两个以上正在进行中的 GET 请求
	  3. 往前回溯至 5 秒前的最后一个长任务结束的时间

-   Total blocking time 总阻塞时间 (TBT)
	阻塞总时间，TBT（Total Blocking Time），记录在 FCP 到 TTI 之间所有长任务的阻塞时间总和。

-   Cumulative layout shift 累积布局偏移 (CLS)
	累计位移偏移，CLS（Cumulative Layout Shift），记录了页面上非预期的位移波动。页面渲染过程中突然插入一张巨大的图片或者说点击了某个按钮突然动态插入了一块内容等等相当影响用户体验的网站。这个指标就是为这种情况而生的，计算方式为：位移影响的面积 * 位移距离。

### 三大核心指标（Core Web Vitals）

Google 在 20 年五月提出：网站用户体验的三大核心指标

![[Pasted image 20231113155229.png]]

#### **`Largest Contentful Paint (LCP)`**

`LCP` 代表了页面的速度指标，虽然还存在其他的一些体现速度的指标，但是上文也说过 `LCP` 能体现的东西更多一些。一是指标实时更新，数据更精确，二是代表着页面最大元素的渲染时间，通常来说页面中最大元素的快速载入能让用户感觉性能挺好。

哪些元素可以被定义为最大元素？

- `<img>` 标签
- `<image>` 在 svg 中的 image 标签
- `video` video 标签
- CSS background url () 加载的图片
- 包含内联或文本的块级元素

![[Pasted image 20231113161909.png]]
**线上测量工具**

- [Chrome User Experience Report](https://developer.chrome.com/docs/crux/)
- [PageSpeed Insights](https://pagespeed.web.dev/?utm_source=psi&utm_medium=redirect)
- [Search Console（Core Web Vitals report）](https://support.google.com/webmasters/answer/9205520)
- [web-vitals JavaScript library](https://github.com/GoogleChrome/web-vitals)

**实验工具**

- [Chrome DevTools](https://developer.chrome.com/docs/devtools/)
- [Lighthouse](https://developer.chrome.com/docs/lighthouse/overview/)
- [WebPageTest](https://www.webpagetest.org/)

**原生的 JS API 测量**

`LCP` 还可以用 `JS API` 进行测量，主要是用 `PerformanceObserver` 接口，目前除了 `IE` 不支持，其他浏览器基本都支持。

```js
new PerformanceObserver((entryList) => {
	for (const entry of entryList.getEntries()) {
		console.log('LCP candidate:', entry.startTime, entry)
	}
}).observe({ type: 'largest-contentful-paint', buffered: true })
```

##### **如何优化 LCP**
LCP 可能被这四个因素影响：

- 服务端响应时间
- JavaScript 和 CSS 引起的渲染卡顿
- 资源加载时间
- 客户端渲染

#### **`First Input Delay (FID)`**

`FID` 代表了页面的交互体验指标，毕竟没有一个用户希望触发交互以后，页面的反馈很迟缓，交互响应的快会让用户觉得网页挺流畅。

![[Pasted image 20231113165052.png]]

这个指标挺容易理解，就是看用户交互事件触发到页面响应中间耗时多少，如果其中有长任务发生的话那么势必会造成响应时间变长。推荐响应用户交互在 ==100ms== 以内。

![[Pasted image 20231113165319.png]]

**线上测量工具**

- [Chrome User Experience Report](https://developer.chrome.com/docs/crux/)
- [PageSpeed Insights](https://pagespeed.web.dev/?utm_source=psi&utm_medium=redirect)
- [Search Console（Core Web Vitals report）](https://support.google.com/webmasters/answer/9205520)
- [web-vitals JavaScript library](https://github.com/GoogleChrome/web-vitals)

**原生的 JS API 测量**

```js
new PerformanceObserver((entryList) => {
	for (const entry of entryList.getEntries()) {
		const delay = entry.processingStart - entry.startTime
		console.log('FID candidate:', delay, entry)
	}
}).observe({ type: 'first-input', buffered: true })
```

##### **如何优化 FID**
FID 可能被这四个因素影响：

- 减少第三方代码的影响
- 减少Javascript的执行时间
- 最小化主线程工作
- 减小请求数量和请求文件大小

#### **`Cumulative Layout Shift (CLS)`**

`CLS` 代表了页面的稳定指标，它能衡量页面是否排版稳定。尤其在手机上这个指标更为重要，因为手机屏幕挺小，`CLS` 值一大的话会让用户觉得页面体验做的很差。`CLS` 的分数在0.1 或以下，则为 Good。

![[Pasted image 20231113170858.png]]

浏览器会监控两帧之间发生移动的不稳定元素。布局移动分数由 2 个元素决定：`impact fraction` 和 `distance fraction` 。

```js
layout shift score = impact fraction * distance fraction
```

下面例子中，竖向距离更大，该元素相对适口移动了 25% 的距离，所以 distance fraction 是 0.25。所以布局移动分数是 0.75 * 0.25 = 0.1875。

![[Pasted image 20231113172009.png]]

**但是要注意的是，并不是所有的布局移动都是不好的，很多 web 网站都会改变元素的开始位置。只有当布局移动是非用户预期的，才是不好的**。

换句话说，当用户点击了按钮，布局进行了改动，这是 ok 的，`CLS` 的 `JS API` 中有一个字段 `hadRecentInput`，用来标识 500ms 内是否有用户数据，视情况而定，可以忽略这个计算。

**线上测量工具**

- [Chrome User Experience Report](https://developer.chrome.com/docs/crux/)
- [PageSpeed Insights](https://pagespeed.web.dev/?utm_source=psi&utm_medium=redirect)
- [Search Console（Core Web Vitals report）](https://support.google.com/webmasters/answer/9205520)
- [web-vitals JavaScript library](https://github.com/GoogleChrome/web-vitals)

**实验工具**

- [Chrome DevTools](https://developer.chrome.com/docs/devtools/)
- [Lighthouse](https://developer.chrome.com/docs/lighthouse/overview/)
- [WebPageTest](https://www.webpagetest.org/)

**原生 JS API 测量**

```js
let cls = 0

new PerformanceObserver((entryList) => {
  for (const entry of entryList.getEntries()) {
    if (!entry.hadRecentInput) {
      cls += entry.value;
      console.log(‘Current CLS value:’, cls, entry);
    }
  }
}).observe({ type: 'layout-shift', buffered: true });
```

##### **如何优化 CLS**
我们可以根据这些原则来避免非预期布局移动：

- 图片或视屏元素有大小属性，或者给他们保留一个空间大小，设置 width、height，或者使用 [unsized-media feature policy](https://github.com/w3c/webappsec-permissions-policy/blob/main/policies/unsized-media.md) 。
- 不要在一个已存在的元素上面插入内容，除了相应用户输入。
- 使用 animation 或 transition 而不是直接触发布局改变。

## 性能工具

Google 开发的 [所有工具](https://web.dev/articles/vitals-tools?hl=zh-cn)都支持 Core Web Vitals 的测量。
工具如下：

- [Lighthouse](https://developer.chrome.com/docs/lighthouse/overview/)
- [PageSpeed Insights](https://pagespeed.web.dev/?utm_source=psi&utm_medium=redirect)
- [Chrome DevTools](https://developer.chrome.com/docs/devtools/)
- [web-vitals JavaScript library](https://github.com/GoogleChrome/web-vitals)
- [Chrome UX Report](https://developer.chrome.com/docs/crux/)

工具：思考与总结
我们该如何选择？如何使用好这些工具进行分析？

- **首先使用 Lighthouse，在本地进行测量，根据报告给出的一些建议进行优化；**
- **发布后，使用 PageSpeed Insights 去看线上的性能情况；**
- **接着，使用 Chrome User Experience Report API 去捞取线上过去 28 天的数据**；
- **发现数据有异常，使用 DevTools 工具进行具体代码定位分析；**
- **使用 Search Console’s Core Web Vitals report 查看网站功能整体情况；**
- **使用 Web Vitals 扩展方便的看页面核心指标情况**

