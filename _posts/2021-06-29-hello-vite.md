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

## 实践

本人一直信奉实践为主，学而致用，所以接下来我们使用 `Vite` + `Tailwindcss` 来创建一个简单前端项目应用。

先来看设计图

![login_page](https://www.uidesigndaily.com/uploads/444/day_444.png)

开始动手吧！ 

通过 npm 安装 `tailwind` 并且初始化
```
npm install -D tailwindcss@npm:@tailwindcss/postcss7-compat @tailwindcss/postcss7-compat postcss@^7 autoprefixer@^9

npx tailwindcss init -p
```

初始化之后，我们得到两个文件 `tailwind.config.js` 和 `postcss.config.js`

tailwind.config.js 来用增加自定义样式       
postcss.config.js 来用增加插件      

在 src/index.css 写入以下内容, 没有这个文件就创建
```
/* ./src/index.css */

/*! @import */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

最后，将 css 文件添加到  `src/main.js` 里
```
// src/main.js
import { createApp } from 'vue'
import App from './App.vue'
import './index.css'

createApp(App).mount('#app')
```

现在就可以开始 `tailwindcss` 之旅了! 

首先，我们创建一个蓝色的背景，很简单，在最外一层设置 `w-screen` `h-screen` `bg-blue-100` 就可以了。

接着，要有一张带不透明度的背景图片，tailwindcss 背景色不透明度比较好处理，只要加上`bg-opacity-50`就可以，但是图片不透明度的处理，需要在图片上层增加一个灰色的半透明层，两个图层叠加在一起就可以实现模糊的效果，看代码：

```
<div class="relative md:w-80 md:h-4/5 w-screen h-full bg-center shadow-2xl">
    <img class="absolute w-full h-full" src="/src/assets/bg.jpg" />
    <div class="absolute inset-0 w-full h-full bg-indigo-100 bg-opacity-25"></div>
</div>
```
然后就是头像，圆角和圆框都非常好处理，只要加个这两个类名  `border-b-2` `border-white-500` , 居中处理 `top-1/4` `left-1/2` `transform` `-translate-x-1/2` `-translate-y-1/2` `flex` `items-center` `justify-center`, 代码如下

```
 <img class="relative h-32 w-32 rounded-full h-24 w-24 top-1/4 left-1/2 transform -translate-x-1/2 -translate-y-1/2 flex items-center justify-center border-8 border-grey-600" src="/src/assets/avatar.png" />
```


最终效果：
![login](https://github.com/frankffenn/hello-vue/blob/master/screenshot/login.png?raw=true)

没有写一行 css 代码就能实现这样的效果，真的是太棒了!

完整代码 :https://github.com/frankffenn/hello-vue