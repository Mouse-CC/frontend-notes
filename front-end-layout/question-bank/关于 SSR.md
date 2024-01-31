# 目录结构

```html
|- ...
|- public
|- server
|- src
|- ...
|- vue.config.js
```

![[关于 SSR 2024-01-19 10.26.02.excalidraw]]

# 建立 server 服务

>[!tip] 提示
>服务端所有文件存放在 📁 `server` 内部

## 服务
`server/index.js`
```js
const express = require('express')
const Vue = require('vue')
const app = express() // 服务
const renderer = require('vue-server-renderer').createRenderer() // 服务端渲染辅助 

const page = new Vue({
  template: "<div>hello ssr</div>"
})

// static 内部为客户端入口的绝对路径
app.use(express.static('../dist/client'))

// 客户端请求服务端，返回模板
app.get('*', async(req, res) => {
  try {
    const html = await renderer.renderToString(page)
    res.send(html) // 返回渲染完成的 html
  } catch(err) {
    res.status(500).send('SERVER INNER ERROR')
  }
})

app.listen(3000, () => {
  // 在 3000 端口启动服务端
})
```

## vue 两个版本：`runtime` 和 `compiler` 的区别？

>[!note] 划分
 **`compiler`** 版本的工作主要是做模板编译
 **`runtime`** 版本不带编译，需要结合 **`vue-template-compiler`** 包一起使用，包生成模板，**`runtime`** 处理 vue 的逻辑

# 配置路由服务

>[!tip]
 应用维度：单页应用，通过路由去切换 vue 实例
 请求维度：现在不是每个页面都复用 **`main`** 中的 **`createApp`**  函数去创建实例，以请求做划分，一次请求为所有路由的生命周期。

## 路由
`src/router/index.js`
```js
import Vue from 'vue'
import Router from 'vue-router'
import HelloWorld from '@/components/HelloWorld'

export default function createRouter() {
  return new Router({
    router: [{
      path: '/',
      name: 'HelloWorld'
      component: HelloWorld
    }]
  })
}
```
改造后每次请求路由都会生成一个新的路由关系

## 修改 `main.js`
```js
// The Vue build version to load with the `import` command
// (runtime-only or standalone) has been set in webpack.base.conf with an alias.
import Vue from "vue";
import App from "./App";
import createRouter from "./router";

Vue.config.productionTip = false;

export default function createApp() {
  const router = createRouter();
  const app = new Vue({
    router,
    render: (h) => h(App),
  });

  return { app, router };
}
```
每次请求都是要走服务端，服务端不存在状态链接 (无状态)，所以每次进入都应该是生成一套新的 app 体系，路由也生成一套新的 ...

## 客户端入口
`src/entry-client.js`
```js
import createApp from "./main";

const { app, router } = createApp();

// router.onReady 可以有效确保服务端渲染时服务端和客户端输出的一致
router.onReady(() => {
  app.$mount("#app");
});
```

## 服务端入口
`src/entry-server.js`
```js
import createApp from "./main";

// 每次请求内容不同，路由递增
export default (context) => {
  return new Promise((resolve, reject) => {
    // 获取基础路由
    const { app, router } = createApp();

    // 增加当前路由
    router.push(context.url);
    // 同步客户端和服务端渲染，给到路由更新反馈
    router.onReady(() => {
      resolve(app);
    }, reject);
  });
};
```
