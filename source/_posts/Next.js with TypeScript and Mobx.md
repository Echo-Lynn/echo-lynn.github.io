---
title: Next.js with TypeScript and Mobx
date: 2019-10-16 14:36:40
categories:
- Programming
- JavaScript
---

# 前言

有关Next.js、同构直出、SEO、SPA等相关介绍将不再赘述，本文主要针对Next.js配合typescript和mobx搭建一个完整的生产部署的前端工程进行核心代码的分析以及主要坑点的讲解，非Next.js入门课程，下面我将会列出本教程所需要的前置预备知识和能力：

- nodejs服务端编程基础
- 已至少阅读一遍Next.js官方文档
- 熟练使用reactjs
- 熟练使用webpack
- 理解同构直出的概念和它解决了什么样的痛点
- 有一定的前端工程化、自动化部署的经验

正文开始时，也就默认了有缘阅读到此文的同学均具备上述能力

# 创建基于typescript的项目
Zeit在2019/07发布了Next.js 9 该版本最吸人眼球的两个Feature分别是 **Built-in Zero-Config TypeScript Support** 和 **File system-Based Dynamic Routing** 即**零配置内置TypeScript支持**和**基于文件系统的动态路由支持**，这里主要提及一下关于TypeScript的支持。在9.0之前的版本，Next.js从6.0开始通过一个名为 *@zeit/next-typescript* 提供了基础版本的TypeScript支持，但并没有整合类型检查，Next.js核心代码本身也不提供types类型所以这个版本提供的TypeScript支持并不友好。Zeit本次发布的Next.js 9 核心代码使用TypeScript重构，因此给开发体验带来了极致的提升。以下将使用官方提供的Demo *with-typescript* 作为种子项目，后面内容将在这个项目上进行集成

## 安装

```bash
npx create-next-app --example with-typescript with-typescript-app
# or
yarn create next-app --example with-typescript with-typescript-app
```
## 启动

```bash
cd with-typescript-app
yarn dev
```

得到以下目录结构：

```bash
with-typescript-app
├─ .gitignore
├─ README.md
├─ components
│  ├─ Layout.tsx
│  ├─ List.tsx
│  ├─ ListDetail.tsx
│  └─ ListItem.tsx
├─ interfaces
│  └─ index.ts
├─ next-env.d.ts
├─ package.json
├─ pages
│  ├─ about.tsx
│  ├─ detail.tsx
│  ├─ index.tsx
│  └─ initial-props.tsx
├─ tsconfig.json
├─ utils
│  └─ sample-api.ts
└─ yarn.lock
```

# 使用mobx作为app状态管理方案

有关Mobx的介绍请自行官网查阅：[https://mobx.js.org/]
## 安装依赖
安装mobx、mobx-react模块：

```bash
yarn add mobx mobx-react
// or
npm install --save mobx mobx-react
```

安装babel plugin对装饰器提供编译支持：

```bash
yarn add -D @babel/plugin-proposal-class-properties @babel/plugin-proposal-decorators
// or
npm install --save-dev @babel/plugin-proposal-class-properties @babel/plugin-proposal-decorators
```
## 配置
创建一个.babelrc的文件在工程的根目录

```bash
touch .babelrc
vi .babelrc
```

写入

```json
{
  "presets": [
    "next/babel"
  ],
  "plugins": [
    ["@babel/plugin-proposal-decorators", { "legacy": true }],
    ["@babel/plugin-proposal-class-properties", { "loose": true }]
  ]
}
```
并在tsconfig.json中加入一行配置来使ts支持装饰器语法：
```json
{
  "compilerOptions": {
    "experimentalDecorators": true
  }
}
```
## store子模块代码实现
创建stores文件夹并创建user.ts：
```bash
mkdir stores
touch stores/user.ts
```
写入:
```typescript
// user.ts
import {action, observable} from 'mobx'

export default class UserStore {

  @observable name: string = 'Clint'

  constructor (initialState: any = {}) {
    this.name = initialState.name;
  }

  @action setName(name: string) {
    this.name = name
  }
}
```
UserStore类中的构造函数的意义是：**接受初始化数据来对该store下的状态进行初始化或者将在服务端渲染首屏时已经产生的状态同步到客户端（这里是同构直出中状态同步一个非常关键的环节，只有理解得足够透彻，Next.js才能用得得心应手**
由于每次创建一个这样的store子模块都需要实现一样的构造函数来对模块中的状态初始化或同步，我们可以通过编写一个基类，让所有store子模块继承这个基类来优化一下代码：
创建stores/base.ts，写入：
```typescript
// base.ts
export default class Base {
  [key: string]: any

  constructor(initState: { [key: string]: any } = {}) {
    for (const k in initState) {
      if (initState.hasOwnProperty(k)) {
        this[k] = initState[k]
      }
    }
  }
}
```
修改user.ts:
```typescript
// user.ts
import {action, observable} from 'mobx'
import Base from './base'

export default class UserStore extends Base {

  @observable name: string = 'Clint'

  @action setName(name: string) {
    this.name = name
  }
}
```
创建stores/config.ts，当有新的store子模块需要创建时候，只要通过这个配置文件引入子模块即可自动集成到根store中：
```bash
touch stores/config.ts
```
写入：
```typescript
import userStore from './user'
import Base from './base'

const config: { [key: string]: typeof Base } = {
  userStore
}

export default config
```
## mobx主体逻辑
优化了store子模块的代码以后，接下来实现store的主体逻辑，创建stores/index.ts：
```bash
touch stores/index.ts
```
写入:
```typescript
import {useStaticRendering} from 'mobx-react'
import config from './config'

const isServer = typeof window === 'undefined'
// Comment 1
useStaticRendering(isServer)

export class Store {
  [key: string]: any
  // Comment 2
  constructor(initialState: any = {}) {
    for (const k in config) {
      if (config.hasOwnProperty(k)) {
        this[k] = new config[k](initialState[k])
      }
    }
  }
}

let store: any = null
// Comment 3
export function initializeStore(initialState = {}) {
  if (isServer) {
    return new Store(initialState)
  }
  if (store === null) {
    store = new Store(initialState)
  }

  return store
}

```
代码注释：
  1. 由于Next.js首屏渲染是在服务端执行的，Mobx所创建的状态是可观察的对象，使用Mobx创建的可观察对象会在内存中使用listener来监听对象的变化，但实际上在服务端是没有必要监听变化的，因为首屏渲染完成得到html文件后，后续的工作都由客户端接手，所以如果在服务端的对象是可观察的，将有可能造成内存泄漏，所以我们使用useStaticRendering方法，当该文件在服务端执行时，让Mobx创建静态的普通js对象即可
  2. 构造函数将在Mobx的根store下挂载上文创建的子模块，并将接收到的初始状态/服务端透传的状态一一赋值给子模块，**当赋值过程是服务端状态同步时，由于执行环境是客户端，子模块中的状态将重新获得可观察的属性，能够让使用了该状态值的react组件响应变化当赋值过程是服务端状态同步时，由于执行环境是客户端，子模块中的状态将重新获得可观察的属性，能够让使用了该状态值的react组件响应变化**
  3. *initializeStore* 方法，服务端渲染时，每个独立的请求都将创建一个新的store，以此来隔离请求之间的状态混淆，当客户端渲染时，只需要引用之前已经创建过的store即可，因为同一个应用程序（SPA）应该共享一颗状态树
以上即Mobx状态管理的主逻辑实现，接下来将讲述Mobx如何配合Next.js和react实现状态管理
## mobx-react

Mobx配合react实现状态管理可以引用mobx-react来实现，写代码之前我们先来分析一下需求，即希望达Mobx具备什么样能力。

前文我们设计Mobx代码结构的时候，实现了一个store的子模块概念，那么第一个问题来了，**能通过注入的方式，给页面按需加载我们所需要的store子模块吗？**

另外，我们都已经知道，Next.js是通过一个实现一个名为*getInitialProps*的静态方法来做到当页面被首屏请求的时候，在服务端执行*getInitialProps*从而获取页面渲染所需的数据来做服务端渲染的，那么第二个问题：**如何在 *getInitialProps* 中获取store对象？**

第三，上文同样提到了，我们服务端首屏渲染的时候会产生一些初始状态存在store的某个或者某些子模块中，那么**Next.js是通过什么手段将这些状态带给客户端的** 而 **我们又怎样才能让这些状态同步到客户端的store对象里来保持服务端客户端状态一致呢？**这是第三和第四个问题。归纳一下需要解决的事务：

1. 向react组件注入store子模块
2. 在*getInitialProps*方法中使用store对象填充数据
3. 分析Next.js数据从服务端向客户端同步的机制
4. 同步服务端和客户端的store状态

解决第一个问题我们需要重写Next.js的*_app.tsx*文件：

```bash
touch pages/_app.tsx
```

写入：

```tsx
// pages/_app.tsx
import App, {AppContext} from 'next/app'
import React from 'react'
import {initializeStore, Store} from '../stores'
import {Provider} from 'mobx-react'

class MyMobxApp extends App {

  mobxStore: Store

  // Fetching serialized(JSON) store state
  static async getInitialProps(appContext: AppContext): Promise<any> {
    const ctx: any = appContext.ctx
    // Comment 1
    ctx.mobxStore = initializeStore()
    const appProps = await App.getInitialProps(appContext)

    return {
      ...appProps,
      initialMobxState: ctx.mobxStore
    }
  }

  constructor(props: any) {
    super(props)
    // Comment 2
    const isServer = typeof window === 'undefined'
    this.mobxStore = isServer ? props.initialMobxState : initializeStore(props.initialMobxState)
  }

  render() {
    const {Component, pageProps}: any = this.props
    return (
      // Comment 3
      <Provider {...this.mobxStore}>
        <Component {...pageProps} />
      </Provider>
    )
  }
}

export default MyMobxApp
```

代码注释：

1. 创建（服务端）或获取（客户端）store对象命名为mobxStore，将mobxStore挂载到appContext.ctx对象上，这个对象会在页面的*getInitialProps*方法中作为入参传入，这就解决了上述的第二个问题

2. 这里其实需要先解释一下Next.js同构直出的原理：**当首屏被请求时，Next.js在服务端利用react渲染页面的机制（服务端渲染生命周期只会执行到render）渲染出html文件后，来满足SEO的需求和首屏页面的展示，然后返回给客户端（通常是浏览器），到了浏览器，Next.js则会跑一遍完整React的生命周期渲染，所以只要渲染结果一致，react内置的*diff*算法结果没有任何差异，你将不会看到页面有任何可察觉的变化**
  Next.js通过什么方式来保证第二点提到的**渲染结果一致**呢？这就是我们要解决的第三个事务。Next.js服务端渲染html文件的同时，将本次请求产生的有关数据通过写入script标签的方式插在html文件一并返回。起一下本地服务，我们使用Chrome控制台看一下实际数据

  ```bash
  yarn dev
  ```

  ```javascript
  <script id="__NEXT_DATA__" type="application/json">
    {"dataManager":"[]","props":{"pageProps":{},"initialMobxState":{"userStore":{}}},"page":"/","query":{},"buildId":"development"}
  </script>
  ```

  就是以这种方式，**Next.js运行在客户端时会依据服务端带回的*__NEXT_DATA__*构建React SPA**，这就是同构直出的核心原理。

  从上面得到的数据，我们不难发现*initialMobxState*被带回，这时，回过头来看下*pages/_app.tsx*中的一段代码：

  ```typescript
  constructor(props: any) {
      super(props)
      const isServer = typeof window === 'undefined'
      this.mobxStore = isServer ? props.initialMobxState : initializeStore(props.initialMobxState)
    }
  ```

  在构造函数的执行环境为客户端时，store对象会依据*__NEXT_DATA__*中的*props.initialMobxState*被创建，这就完成了服务端store的状态向客户端同步，这就解决了事务4



3. 将store使用拓展运算符将子模块通过props注入到*provider*组件，配合mobx-react提供的*inject*方法来达到按需获取store模块的功能，下面给出一种用法代码示例，更多使用方式请移步mobx-react[https://github.com/mobxjs/mobx-react] 了解更多

   ```tsx
   // pages/detail.tsx
   import * as React from 'react'
   import Layout from '../components/Layout'
   import {User} from '../interfaces'
   import {findData} from '../utils/sample-api'
   import ListDetail from '../components/ListDetail'
   import {inject, observer} from 'mobx-react'
   import UserStore from '../stores/user'

   type Props = {
     item?: User
     userStore: UserStore
     errors?: string
   }

   @inject('userStore')
   @observer
   class InitialPropsDetail extends React.Component<Props> {
     static getInitialProps = async ({query, mobxStore}: any) => {
       mobxStore.userStore.setName('set by server')
       try {
         const {id} = query
         const item = await findData(Array.isArray(id) ? id[0] : id)
         return {item}
       } catch (err) {
         return {errors: err.message}
       }
     }

     render() {
       const {item, errors} = this.props

       if (errors) {
         return (
           <Layout title={`Error | Next.js + TypeScript Example`}>
             <p>
               <span style={{color: 'red'}}>Error:</span> {errors}
             </p>
           </Layout>
         )
       }

       return (
         <Layout
           title={`${item ? item.name : 'Detail'} | Next.js + TypeScript Example`}
         >
           {item && <ListDetail item={item}/>}
           <p>
             Name: {this.props.userStore.name}
           </p>
           <button onClick={() => {
             this.props.userStore.setName('set by client')
           }}>click to set name
           </button>
         </Layout>
       )
     }
   }

   export default InitialPropsDetail

   ```

   访问: [http://localhost:3000/detail?id=101] 查看效果

以上，就是基于Next.js开发的几个比较核心的思想和库的使用，下面开始介绍在构建和部署方面的内容

# 构建编译

Next.js使用webpack来构建打包项目，当项目不需要特殊的定制化构建的时候，执行以下命令即可构建项目包

```bash
next build
```

在前言里也提到，本文着重讲部署Next.js的完整实例，那么只以默认方式构建项目显然是满足不了我们的实际的生产诉求了，我会在这里讲一些平常我们构建项目所需要的几个比较通用的需求点，当然覆盖不了所有，不过也可以提供一些思路。

在这里，也顺便一提，当我们使用一个框架来搭建应用的时候，**能使用框架本身提供的API实现功能请尽量使用**，这样做的好处有哪些：

1. 避免重复造轮子
2. 自然形成一套规范和标准，团队开发减少学习成本
3. 文档现成，使用起来水到渠成
4. 项目里越少带有主观偏好的代码越好

## 环境分割

一个生产项目避免不了环境这个问题，比较常见的项目环境分为dev test production，即开发、测试、生产，下面我们以这类环境划分为例，多几种或者少几种同理可推

通常我们将项目内引用到的环境变量抽离出来，用配置文件把变量存起来，根据程序运行的环境来索引对应的配置文件，取出变量使用

在根目录下创建*/config*目录，分别创建*dev.js，test.js，prod.js*（提一下，这里为什么不是.ts文件呢，因为这个配置文件，构建时候被引用的文件，是不经过ts编译的）*index.js*项目根目录下执行：

```bash
mkdir config
touch config/dev.js config/test.js config/prod.js config/index.js
```

分别写入：

```javascript
// config/dev.js
module.exports = {
  env: 'dev'
}
```
```javascript
// config/test.js
module.exports = {
  env: 'test'
}
```
```javascript
// config/prod.js
module.exports = {
  env: 'prod'
}
```

```javascript
// config/index.js
const dev = require('./dev')
const test = require('./test')
const prod = require('./prod')

module.exports = {
  dev,
  test,
  prod
}
```



Next.js构建(`next build`)和启动应用(`next`、`next start`)通过在根目录下*next.config.js*文件读取定制化的配置选项，当文件不存在时，使用默认配置构建

创建*next.config.js*

```bash
touch next.config.js
```

```javascript
// next.config.js
const config = require('./config')
// Get process DEPLOY_ENV value
const DEPLOY_ENV = process.env.DEPLOY_ENV || 'dev'

module.exports = {
  serverRuntimeConfig: {
    // Will only be available on the server side
    secret: 'secret',
  },
  // Use which config file according to DEPLOY_ENV
  publicRuntimeConfig: config[DEPLOY_ENV]
}

```

修改*pages/index.tsx*文件：

```tsx
// pages/index.tsx
import * as React from 'react'
import Link from 'next/link'
import Layout from '../components/Layout'
import { NextPage } from 'next'
import getConfig from 'next/config'

const {publicRuntimeConfig, serverRuntimeConfig} = getConfig()

const IndexPage: NextPage = () => {
  return (
    <Layout title="Home | Next.js + TypeScript Example">
      <h1>Hello Next.js 👋</h1>
      <p>Public config JSON string: {JSON.stringify(publicRuntimeConfig)}</p>
      <p>Server side config JSON string: {JSON.stringify(serverRuntimeConfig)}</p>
      <p>
        <Link href="/about">
          <a>About</a>
        </Link>
      </p>
    </Layout>
  )
}

export default IndexPage
```

Next.js配置文件中，有两个配置选项*serverRuntimeConfig*，*publicRuntimeConfig*，*serverRuntimeConfig*只允许程序运行在服务端时使用，*publicRuntimeConfig*选项同时允许服务端和客户端获取，我用*publicRuntimeConfig*讲解思路

完成以上代码编写后，执行命令

```bash
next
```

使用浏览器打开 [http://localhost:3000]查看效果

可以注意到浏览器显示了*publicRuntimeConfig*是*config/dev.js*的内容，而*serverRuntimeConfig*为空对象，细心的朋友会注意到，当你快速不断刷新页面的时候，是可以看到*serverRuntimeConfig*是由`{"secret":  "secret"}`变为`{}`的，为什么会这样，结合上文提到的Next.js同构直出的核心思想和关于*serverRuntimeConfig*的特性就可以理解该现象了。

那么，现在我们要解决的问题就是，让程序构建后跑在test/prod环境时候，页面显示*config/test.js*或者*config/prod.js*的内容了

以test为例，Next.js的构建命令为`next build`，启动命令为`next start`，运行和构建都会根据*next.config.js*来决定应用构建和启动的定制化配置，从代码里可以看到，我们是根据一个叫*DEPLOY_ENV*的环境变量来索引配置文件的，那么我们只需要在运行`next build`和`next start`的时候给*DEPLOY_ENV*赋值即可

```bash
DEPLOY_ENV=test next build
DEPLOY_ENV=test next start
```

[^温馨提示]: windows平台的同学可能需要借助*cross-env*来改变node.js的运行环境变量，具体用法自行查阅*

执行完上述命令，打开[http://localhost:3000]查看页面是否已显示*config/test.js*的内容，有关环境分割的内容就讲到这里，更多有关环境的拓展可以依据这样的思路来实现

## CSS预编译

这个就更加简单了，官方提供插件的，我就不费口舌讲一遍了，直接上链接

[https://github.com/zeit/next-plugins/tree/master/packages/next-sass]: @zeit/next-sass
[https://github.com/zeit/next-plugins/tree/master/packages/next-less]: @zeit/next-less
[https://github.com/zeit/next-plugins/tree/master/packages/next-stylus]: @zeit/next-stylus

值得一提是的，Next.js在CSS方面有一点不足：**所有的样式文件最终会被打包为一个*style.chunk.css*文件随着首屏加载一并返回**。这会带来一点小小的缺陷就是当你的app工程庞大时，这个文件的体积会对首屏的加载带来一点影响，虽然在gzip压缩后这种影响微乎其微，不过终归是需要优化，另外一个问题就是，类名冲突了，你可能需要利用像Less、Sass这样的嵌套样式写法把不相关的页面样式包裹在一个命名空间里，或者是通过配置`{cssModules: true}`来为你的类名打上hash后缀。

关于CSS文件切割的问题笔者已经给Next.js作者提了issue了，期待后续版本的解决方案。

动手能力强webpack原理够硬的同学也可以尝试自己实现一下这个功能。笔者后面空下来有幸实现了的话，会再分享出来。

# 服务部署

Next.js不同于普通的静态web项目，当然，Next.js也可以搭建一个普通的静态项目，不过同构直出才是它的最大亮点，所以本文所有篇幅都是基于这个点出发的，不讨论其他小众方式运行Next.js

那么想部署同构直出，就需要有web服务器，前端领域目前比较热门的还是Node.js，Next.js的服务端也正是运行在Node.js上，下面介绍一下Next.js简单的部署方案，然后继续针对一些我认为出现频繁的一些场景讲解一下部署思路。

部署项目可以有两种方式：

一是把整个项目目录除了*node_modules*（当然你也可以把这个目录带上去，如果你连接服务端传输网速够快的话）以外的源文件一并上传到服务器，安装项目依赖

```bash
yarn
// or
npm install
```

构建

```bash
DEPLOY_ENV=$YOUR_SERVER_ENV_TYPE next build
```

启动服务

```bash
DEPLOY_ENV=$YOUR_SERVER_ENV_TYPE next start
```

二是你在本地或者使用 [Docs Gitlab Com Runner](https://docs.gitlab.com/runner/) (推荐使用，具体操作自行查阅文档)

构建后把所需要的资源上传到服务器，列一下所需要的目录清单

```bash
app
├─ .next // required
├─ pages // just empty dir, for safe
├─ next.config.js // if have
├─ server.js // if have
├─ static // if have
├─ config // Mentioned above, if have
├─ package.json // required
├─ package-lock.json // optional
└─ yarn.lock // optional
```

- `.next`：`next build`执行后编译完成的文件目录
- `pages`：建议传一个空目录。按理来说不需要，因为里面的源文件已经被打包到*.next*目录去了，但由于最近在部署的时候遇到一个报错提示说找不到pages，弄了一个空目录就正常运行了。emm...晚点去提个issue
- `next.config.js`：如果你有定制化配置的话
- `server.js`：如果你有定制化node服务的话
- `static`: 静态资源目录，由自己创建，Next.js编译会忽略这个目录，如果你app有引用这个目录的静态资源，需要带上
- `config`：前文提到的，如果你按照本文做的环境分割的话
- `package.json`：在服务器需要Next.js等的npm模块来启动服务，所以需要这个文件来安装依赖
- `package-lock.json`：不解释了
- `yarn.lock`：不解释了

完成传输后，运行

```bash
yarn
// or
npm install
// no build command needed
DEPLOY_ENV=$YOUR_SERVER_ENV_TYPE next start
```

## 部署路径

众所周知，Next.js默认是通过文件系统路由的（*file-system routing*）。假设你项目部署的域名是 www.myapp.com ，你要访问*/pages*目录下的*home.tsx*，则访问的url为 http://www.myapp.com/home ，通常这样是能够满足大部分的业务场景的，这一章我想要讲的，就是比较可能出现的另外一种业务场景，即单个域名下部署多个项目，不仅仅是Next.js项目，也有可能是Vue、React、Angular、JQuery等其他类型的web项目
// TODO 未完待续
