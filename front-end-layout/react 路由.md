# 路由的来源

React-router, Vue-router
- SPA 出现之后，前端才开始接管路由

## 以前：

![[react 路由 2024-01-23 14.41.02.excalidraw]]

## 现在：

![[react 路由 2024-01-23 14.57.58.excalidraw]]

### 关于页面不刷新

- `history.pushState` 做到不刷新，本质就是这个浏览器的 `api` ，帮助修改了地址栏内的 `url`
- `localhost:3000/book` 直接输入，是要访问服务器的，为什么没访问到后台 server，因为 `nginx` 做了配置

# 路由是什么

- 路由变化，意味着界面变化？
- 界面变化，意味着加载的组件发生变化？组件内数据变化？
- Keyword：context、components、path

## 路由核心

>[!quote] 根据 / 监听 => path / url 的变化，联系 path 和 components 的对应关系，触发一些组件的 unmount 和 mount ，同时使用 context 注入上下文

![[react 路由 2024-01-23 17.16.04.excalidraw]]

# 路由分类

## History 路由

```js
history./\(go | back | forward | replaceState | pushState)/
```

## Hash 路由

```js
window.location.hash
```

## Memory 路由

## History 和 Hash 路由的区别

1. Hash 路由一般会携带 `#` ，不美观。History 路由不存在这个问题。
2. 默认 hash 路由，不会向浏览器发出请求，history => go | back 都是会发出请求。
3. Hash 模式一般不支持 SSR，history 支持。
4. History 路由部署的时候，需要用 nginx 处理一下。

```nginx
location / {
  try_files uri $uri /xx/xx/xx/index.html
}
```

# React 路由

v6

## `React-router`

提供一些核心的 api，如 Router，Route ... 但不提供和 DOM 相关的。

## `React-router-dom`

提供 BrowserRouter，HashRouter，Link 这些 api，可以通过 DOM 操作触发事件，控制路由

## `History`

模拟浏览器的 history 库，v6 版本，已经将其内置了，并导出成了 navigation

## v6 的特点：

1. 已经没有 component 了，全都是 element
   
```jsx
function App() {
  return (
    <BrowserRouter>
      <header className="w-full h-14 flex flex-row justify-start items-center bg-blue-200">
        <a
          className="mx-2 hover:text-blue-600 hover:font-semibold cursor-pointer"
          href="./"
        >
          首页
        </a>
        <a
          className="mx-2 hover:text-blue-600 hover:font-semibold cursor-pointer"
          href="./news"
        >
          新闻列表
        </a>
        <a
          className="mx-2 hover:text-blue-600 hover:font-semibold cursor-pointer"
          href="./about"
        >
          关于我们
        </a>
        <a
          className="mx-2 hover:text-blue-600 hover:font-semibold cursor-pointer"
          href="./hot"
        >
          热点事件
        </a>
      </header>
      <Routes>
        <Route path="/news" element={<div>新闻列表页--pages</div>} />
        <Route path="/about" element={<div>关于我们页--pages</div>} />
        <Route path="/hot" element={<div>热点事件页--pages</div>} />
      </Routes>
    </BrowserRouter>
  );
}

export default App;
```

2. 提供 Route, Routes, Link，Outlet ... 
   
```jsx
const NavMenu = ({to, children}) => (
  <div className="mx-2 hover:text-blue-600 hover:font-semibold cursor-pointer">
    <Link to={to} >{children}</Link>
  </div>
)

const Menu = () => (
  <div>
    <header className="w-full h-14 flex flex-row justify-start items-center bg-blue-200" >
      <NavMenu to="./">首页</NavMenu>
      <NavMenu to="./news">新闻列表</NavMenu>
      <NavMenu to="./about">关于我们</NavMenu>
      <NavMenu to="./hot">热点事件</NavMenu>
    </header>
    <Outlet />
  </div>
)

function App() {
  return <BrowserRouter>
    <Routes>
      <Route path="/" element={<Menu />}>
        <Route path="/news" element={<div>新闻列表页--pages</div>} />
        <Route path="/about" element={<div>关于我们页--pages</div>} />
        <Route path="/hot" element={<div>热点事件页--pages</div>} />
      </Route>
    </Routes>
  </BrowserRouter>
}
```
   
3. 提供了一些 api
   
   - UseRoutes
   - UseNavigate
   - UseParams
   - UseLocation

```jsx
const News = () => {
  const nav = useNavigate();
  return <div>
    this is the news Pages:
    <ul>
      <li>1. ygoiuygogoig</li>
    </ul>
    <button className="border-dark-100 border border-solid rounded-md p-2" onClick={() => nav('/post/1')}>去看新闻1</button>
  </div>
}

const Post = () => {
  const { id } = useParams();
  const location = useLocation();

  console.log(location);
  return <div>
    {id}: 新闻信息
  </div>
}

let routes = [
  {
    path: '/', element: <Menu />,
    children: [
      { path: '/news', element: <News />},
      { path: "/post/:id", element: <Post />},

      { path: '/about', element: <div>关于我们页--pages</div>},
      { path: '/hot', element: <div>热门新闻页--pages</div>}
    ]
  }
];

const MyRouting = () => useRoutes(routes);

const App = () => (
  <BrowserRouter>
    <MyRouting />
  </BrowserRouter>
)
```

懒加载
```jsx
const DynamicNews = lazy(() => import("./pages/hots"));

let routes = [
  {
    path: "/",
    element: <Menu />,
    children: [
      // ...
      {
        path: "/hot",
        element: (
          <div>
            <Suspense fallback={<div>loading..</div>}>
              <DynamicNews />
            </Suspense>
          </div>
        ),
      },
    ],
  },
];
```

# 手写

```js
import { createBrowserHistory, createHashHistory } from "history";
import React, {
  createContext,
  useContext,
  useLayoutEffect,
  useMemo,
  useRef,
  useState,
} from "react";

// 创建上下文
const NavigationContext = createContext({});
const LocationContext = createContext({});

export function HashRouter({ children }) {
  let historyRef = useRef();
  if (historyRef.current == null) {
    historyRef.current = createHashHistory();
  }

  let history = historyRef.current;

  let [state, setState] = useState({
    action: history.action,
    location: history.location,
  });

  useLayoutEffect(() => history.listen(setState), [history]);

  return (
    <Router
      children={children}
      location={state.location}
      navigator={history}
      navigationType={state.action}
    />
  );
}

export function BrowserRouter({ children }) {
  let historyRef = useRef();
  if (historyRef.current == null) {
    historyRef.current = createBrowserHistory();
  }

  let history = historyRef.current;

  let [state, setState] = useState({
    action: history.action,
    location: history.location,
  });

  // 我们需要监听这个 history 的变化
  // 当 history 变化时，（浏览器输入，a标签）
  // 派发更新，渲染整个 router 树
  // 同步执行 useLayoutEffect
  useLayoutEffect(() => history.listen(setState), [history]);

  return (
    <Router
      children={children}
      location={state.location}
      navigator={history}
      navigationType={state.action}
    />
  );
}

// Provider
function Router({ children, location: locationProp, navigator }) {
  const navigatorContext = useMemo(() => ({ navigator }), [navigator]);
  const locationContext = useMemo(
    () => ({ location: locationProp }),
    [locationProp]
  );

  return (
    <NavigationContext.Provider value={navigatorContext}>
      <LocationContext.Provider
        value={locationContext}
        children={children}
      ></LocationContext.Provider>
    </NavigationContext.Provider>
  );
}

function useLocation() {
  return useContext(LocationContext).location;
}

function useNavigate() {
  return useContext(NavigationContext).navigator;
}

// 我这里是 const Routing = () => useRoutes(routes); 这样的
// 所以我这个 Routing，在代码里，就是返回对应的组件
// 所以这里要做的就是，根据 url， 返回 对应的 element
export const useRoutes = (routes) => {
  let location = useLocation(); // 拿到当前路径
  let currentPath = location.pathname || "/";
  for (let i = 0; i < routes.length; i++) {
    let { path, element } = routes[i];
    let match = currentPath.match(new RegExp(`^${path}`));
    if (match) {
      return element;
    }
  }
  return null;
};

export const Route = () => {};

// Routes 这个东西
export const Routes = ({ children }) =>
  useRoutes(createRoutesFromChildren(children));

// 这个函数的作用，就是把嵌套的 <Route /> 组件，转成一棵树。
export const createRoutesFromChildren = (children) => {
  let routes = [];
  // children.forEach((node) => { ... })
  React.Children.forEach(children, (node) => {
    let route = {
      element: node.props.element,
      path: node.props.path,
    };
    if (node.props.children) {
      route.children = createRoutesFromChildren(node.props.children);
    }
    routes.push(route);
  });
  return routes;
};
```
