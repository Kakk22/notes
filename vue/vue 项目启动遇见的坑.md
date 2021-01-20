# vue 项目启动遇见的坑

在`npm install`构建前端时遇到很多环境问题

如报`python`无法在path中找到 则要自己下载`python`后在路径指定`python`安装目录

## node-sass 无法下载导致构建失败

由于node-sass的源使用的是Github上面的，经常无法访问，我们构建的时候需要单独设置node-sass的下载地址。

先删除`node_modules`目录，再配置镜像地址

```bash
# window
set SASS_BINARY_SITE=https://npm.taobao.org/mirrors/node-sass&& npm install node-sass
```

## 有些依赖无法下载导致构建失败

由于npm源访问慢的问题，有些源可能会无法下载，改用淘宝的npm源即可解决。

```bash
# 设置为淘宝的镜像源
npm config set registry https://registry.npm.taobao.org
# 设置为官方镜像源
npm config set registry https://registry.npmjs.org
```

设置完后执行`npm install`  `npm run dev`命令即可启动