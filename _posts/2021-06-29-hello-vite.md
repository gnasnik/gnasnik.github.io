---
layout:     post
title:      "尝试尤大新打包玩具 Vite"
subtitle:   ""
date:       2021-06-29 15:02:26
author:     "frank"
header-style: text
#header-img: img/home-bg-o.jpg
tags:
    - vue
    - 前端
---
 
>Vite 需要 Node.js 版本 >= 12.0.0

## node.js 升级

查看 `node` 版本    
```
node -v
```

升级 `node` 
```
sudo npm i -g n --force 
n stable 
```

 > 升级之后重新打开 shell 生效
 
 
 ## 开始使用
 
使用 `Vite` 构建一个 `Vite` + `Vue` 项目
```
# npm 6.x
npm init @vitejs/app my-vue-app --template vue

# npm 7+, 需要额外的双横线：
npm init @vitejs/app my-vue-app -- --template vue

# yarn
yarn create @vitejs/app my-vue-app --template vue
```

创建完成之后，启动 web 服务器
```
cd my-project

npm install
npm run dev
```

启动比 `webpack` 快多了有木有！！ it's fast！