---
title: Github + Travis CI + PM2实现 Next.js 项目的（其他 Node.js 项目同理）持续集成和自动化部署
date: 2019-12-14 13:15:00
categories:
- Deployment
- Automated deployment
---

# 前言

本文将详细介绍如何实现一个 Next.js 项目的持续集成和自动化部署，鉴于自建 gitlab 和 gitlab-runner 有一定的服务器硬件要求，并且需要一定的时间成本，不是本文的重点，所以我们使用现成的 Github 作为本次教程代码管理工具，Travis CI 来自动化构建， 使用 PM2 作服务器的进程管理工具来完成本次教程。下面我列出本次教程所需的物理材料：

- Linux 远程服务器（笔者使用的版本：Ubuntu 18.04）
- 本地个人开发主机

后文笔者将分别用 remote 和 local 简称上述材料

# 部署策略简述

我先把整个部署的策略按照步骤大致列一下，好让跟学的同学心里有个谱，判定一下符不符合自己的学习诉求

1. 使用 Github 托管源码
2. Travis CI 构建生产包，并将构建包提交到 Github 的部署分支（命名如：DEPLOY-PROD）上
3. Travis CI 完成构建包的推送后，调用 PM2 的 deploy 命令， 触发 remote 服务器拉取最新的构建包，然后自动重启服务

<!-- more -->

# 环境准备

remote 安装node版本管理工具 nvm [https://github.com/nvm-sh/nvm] 不赘述，我信你有这个动手能力

```bash
nvm install 12.13.1 // 安装最新稳定版
nvm use 12.13.1 // 切换node版本
node -v
```



全局安装 yarn (个人偏好)

```bash
npm install -g yarn
yarn -v
```

全局安装 pm2 

```bash
yarn global add pm2
pm2 -v
```



# 源码准备

因为是教程，就一切从简了，这里以一个最小的 Next.js 应用为例

```bash
npx create-next-app
```

安装完成后进入项目根目录试一下能不能在本地跑起来

```bash
cd my-app
yarn dev
```

验证没问题以后，把代码推到你的 Github 上: 

DO IT YOURSELF

# 账号准备

1. 打开 Travis CI 官网 [https://travis-ci.com/]

2. 使用 Github 账号登录 Travis CI，授权

3. 打开 Travis CI 个人页 https://travis-ci.com/account/repositories]

4. 点击Activate

5. 选择点击Approve & Install （可选择所有项目也可以只选择选中的项目，这个以后能在Github - Settings - Applications 里改的， 不重要）

6. 这时 Travis CI 就可以读取到你选中的需要构建的 Github 项目了

7. 配置 Github token

   由于 Travis CI 完成构建以后要把构建包推到 Github 仓库上，所以需要生成一个 Token 以环境变量的方式给 Travis CI ，以获取 push 权限

   Github -> Settings -> Developer settings -> Personal access tokens -> Generate new token -> 填写 Note 备注这个 token 的用途 -> Select scopes 勾选 repo 一项 -> Generate token 确认生成 -> 复制生成的 token (注意这个 token 只会展示一次， 以后不会再次展现) -> 回到 Travis CI 个人页, 找到你想要部署的项目，点击 Settings -> 在 Environment Variables 新增一个名为 `GH_TOKEN` 的环境变量，value 就是刚才复制的 token 然后选择分支 master （根据实际情况选择）， 安全起见，**不要勾选后面的 DISPLAY VALUE IN BUILD LOG** 

# 构建

一切就绪后，现在我们先来解决构建的问题，在项目的根目录下编写构建脚本 *.travis.yml*

```bash
vim .travis.yml
```

```yaml
#.travis.yml
sudo: false
language: node_js
node_js:
  - 12
cache: yarn
#指定构建分支
branches:
  only:
    - master
#安装依赖
install:
  - yarn install
script:
  - yarn build
#构建成功后，把部署所需的文件都拷贝到 dist 文件夹下
after_success:
  - mkdir dist
  - cp -r .next/ dist/.next
  - cp package.json dist
#把 dist 文件推送到 github DEPLOY-PROD 分支下
deploy:
  provider: pages
  skip_cleanup: true
  github_token: $GH_TOKEN
  keep_history: true
  target_branch: DEPLOY-PROD
  committer_from_gh: true
  on:
    branch: master
  local_dir: dist
```

[^注]: 搭建过 Github Page 的朋友对` provider: pages`应该不陌生，这里顺便说明一下，笔者的这个部署策略其实就是借鉴了 Github Page 部署的思路，对这块还不熟悉的朋友，建议也可以自己动手利用 Github Page 搭一个个人博客试试

完成构建脚本的编写后，我们提交一下代码触发一下构建试试

等待 Travis CI 构建完成后，查看一下仓库下是否多了一个 DEPLOY-PROD 分支，并且分支下文件无误就说明构建这一步已经完成了。本文的重点在**自动化部署**，所以下文才是真正的核心部分

# 自动化部署

现在，我们整理一下思路，我们现状是：

- 完成了源码托管
- 完成了自动化构建
- 完成了部署分支 DEPLOY-PROD

我们需要完成的任务是：

- remote 拉取 DEPLOY-PROD 分支下的文件
- 执行启动脚本 `yarn start` 启动服务

笔者以往管理 Node 服务进程常用的一个工具为 

[PM2]: https://pm2.keymetrics.io/

近期在鼓捣自己的服务器经常查阅 PM2 文档时无意间发现了 `pm2 deploy` 的API，如获至宝 

[DEPLOYMENT]: https://pm2.keymetrics.io/docs/usage/deployment/

废话不多说， 列举下利用 `pm2 deploy`的构建机器需要哪些前提条件:

1. 装有 pm2 （还说不废话）
2. 有权登录你的服务器

1好办，2有点头疼

从安全的角度上来说，我们云服务器一般是不允许 root 通过密码登录的，都是密钥对登录，那么我们就需要把我们个人主机（默认 local 已经是通过密钥对连接 remote 了）的私钥传给 Travis CI，怎么传呢，通过上传到 Github 来传？

## Travis CI 构建机器连接 remote 策略

**是的，通过 Github，但当然不是把私钥裸露上传，而是通过 Travis CI 提供的一个命令行工具 travis 对私钥进行加密，travis加密会生成一个加密过的文件并且会自动在项目构建配置里生成两个用于解密的环境变量，我们把加密过的文件上传到 Github 项目仓库下，构建时用两个解密的环境变量把文件解密，这样一来，Travis CI 构建机器就有权限连接 remote了，而且安全问题得以保障**，下面是具体操作步骤：

1. Local 安装 travis 

   ```bash
   gem install travis
   ```

2. 用 github 账号登录 travis

   ```bash
   travis login --com
   ```

3. 进入项目的根目录加密私钥

    ```bash
   travis encrypt-file ~/.ssh/id_rsa --add --com
   ```

   [^注]: 这里默认你的私钥在默认路径上哈，如果有异，自行调整路径即可

    travis会自动识别本地仓库对应的是远程的哪个仓库，执行完命令后，你的本地项目根目录会生成一个 `id_rsa.enc` 文件，你的 `.travis.yml` 脚本会多出一段 `before-install` 的解密脚本（需要调整一下细节，详情见下文完整文件），打开 Travis CI 的项目设置页查看 Environment Variables 一栏多了一个 以 `_iv` 和一个以 `_key` 结尾的环境变量

4. 第3步的结果确认无误后，我们再给 Travis CI 配置多一个环境变量PROD_SERVER_IP来存放 remote IP，添加方法上文讲过，不再赘述。

5.  我们修改构建脚本文件来验证一下是否能够登陆成功

   ```yaml
   sudo: false
   language: node_js
   node_js:
   - 12
   cache: yarn
   branches:
     only:
     - master
   before_install:
     - openssl aes-256-cbc -K $encrypted_db8d9b5ea11f_key -iv $encrypted_db8d9b5ea11f_iv -in id_rsa.enc -out ~/.ssh/id_rsa -d
     #降低id_rsa文件的权限，否则ssh处于安全方面的原因会拒绝读取秘钥
     - chmod 600 ~/.ssh/id_rsa
     #将生产服务器地址加入到构建机的信任列表中，否则连接时会询问是否信任服务器
     - echo -e "Host $PROD_SERVER_IP\n\tStrictHostKeyChecking no\n" >> ~/.ssh/config
   install:
   - yarn install
   script:
   - yarn build
   after_success:
   - mkdir dist
   - cp -r .next/ dist/.next
   - cp package.json dist
   deploy:
     provider: pages
     skip_cleanup: true
     github_token: "$GH_TOKEN"
     keep_history: true
     target_branch: DEPLOY-PROD
     committer_from_gh: true
     on:
       branch: master
     local_dir: dist
   after_deploy:
     ssh root@$PROD_SERVER_IP 'pwd'
   ```

   提交代码，注意要把 id_rsa.enc 文件带上，千万别错把你的真正的私钥提交了。

   等待构建完成，确认构建日志是否打印出 `/root`  字样验证连接成功与否

## 远程服务器获取github仓库pull权限

使用 pm2 部署还有个前提，直接引用官方原话

> Verify that your remote server has the permission to git clone the repository

需要确保 remote 能够使用 ssh 的方式 clone 代码仓库

操作步骤： 

1. remote 生成密钥对（ubuntu为例，已有密钥对的话忽略此步）

   ```bash
   ssh-keygen -t rsa // 一路回车，不要设置使用短语
   cat .ssh/id_rsa.pub // 把打印出来的公钥内容复制
   ```

2. Github -> Settings -> SSH and GPG keys -> New SSH key -> Title 一栏填入备注信息，Key 一栏粘贴第1步复制的公钥 -> Add SSH key

3. 测试 remote 拉取仓库，注意使用 SSH 连接

   ```bash
   git clone $YOUR_SSH_GIT_URL
   ```

4. 拉取权限没问题后，我们创建一个空目录以备后文部署使用

   ```
   mkdir /data/www/demo-for-automated-deployment
   ```

   

## 配置 pm2

解决了仓库拉取的问题后，我们正式进入部署配置的阶段

[DEPLOYMENT]: https://pm2.keymetrics.io/docs/usage/deployment/

参考 pm2 deployment 的相关文档，根据我们实际的需求来编写 pm2 的配置文件，项目根目录下

```json
vim ecosystem.config.js
```

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'da_prod',
      script: 'yarn',
      args: 'run start'
    }
  ],
  deploy: {
    // "prod" is the environment name
    prod: {
      user: 'root',
      key: '~/.ssh/id_rsa',
      host: ['$YOUR_REMOTE_IP'],
      ssh_options: 'StrictHostKeyChecking=no',
      // 拉取部署分支
      ref: 'origin/DEPLOY-PROD',
      // 仓库地址
      repo: 'git@github.com:Echo-Lynn/demo-for-automated-deployment.git',
      // 部署 remote 路径
      path: '/data/www/demo-for-automated-deployment',
      'post-deploy': '.  /root/.nvm/nvm.sh && yarn install && pm2 reload ecosystem.config.js'
    },
  }
}

```

其中 `apps` 选项是启动服务使用的配置，也就是说执行环境在 remote；`deploy` 是部署命令，执行环境在 Travis CI 的构建机器

[^注]: 有个坑点坑了笔者很多时间，我们平时使用终端连上服务器后，系统会开始自动准备好环境变量了，所以我们能轻松自如地在终端里使用 `npm`、`yarn`、`node`等非 Ubuntu 自带的命令。pm2 部署的时候远程调用 `yarn` 和 `pm2` ，是在非交互式的情况下调用的，也就是说环境变量并没有立刻就绪，所以在 `post-deploy` 里需要手动调用一下 `/root/.nvm/nvm.sh` 这个环境变量的脚本

修改 `.travis.yml` 文件的 `install` 和 `after_deploy`

```yaml
install:
	- yarn add global pm2
  - yarn install
after_success:
  - mkdir dist
  - cp -r .next/ dist/.next
  - cp package.json ecosystem.config.js dist
after_deploy:
	- pm2 deploy pm2.config.js prod setup --force
```

第一次使用 `setup` 建立部署环境

提交代码到 Github，等待构建完成后，查看构建日志看到 pm2 deploy后有 `suucess` 字样就说明 setup 成功了

接着，修改 `.travis.yml` 文件的 `after_deploy`

```yaml
after_deploy:
	- pm2 deploy pm2.config.js prod update --force
```

提交代码到 Github，等待构建完成

最后，在远程服务器上

```bash
pm2 ls
curl http://localhost:3000
```

验证进程是否成功启动，服务是否能够访问得到

# 结语

本次分享的内容就到此结束了。由于构建和部署本身就是比较综合性的应用，涉及的知识范围比较广，对动手能力和调试排除能力都有较高的要求，实际操作起来可能由于各种环境的不尽相同也会产生一些不顺利的体验，但这些其实都不重要，重要的是供大家参考的这个部署思路，条条大路通罗马，具体怎么设计策略要看手上的资源和场景需求。

# DEMO地址

[仓库链接]: https://github.com/Echo-Lynn/demo-for-automated-deployment
[笔者博客地址]: https://blog.echo-lynn.com/

