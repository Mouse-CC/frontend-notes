
## 起源

后台路由 => 实例导航

单页路由

- 1. 标签跳转实现单页内多个模块的切换 => 单页内多前端文件（实例）管理
- 2. 单页内管理多个模块之间的切换和传参 => 前端管理参数传递

面试题：

1. 原地刷新问题由来？
     后端将前端路由全部指向根文件 index.html
     前端做实例跳转
     => 是否未配置当前路径的默认指向？是否需要路径回显?
     解决方案：
     配置初始页面，统一刷新触达页面（默认指向） / active 标签（query）：地址栏显示当前实例的路径
