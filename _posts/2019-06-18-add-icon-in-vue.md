---
layout:     post
title:      "在 vue 中使用自定义 icon 笔记"
subtitle:   ""
date:       2019-06-18 13:49:50
author:     "frank"
# header-style: text
header-img: "img/post-bg-halcssting.jpg"
tags:
    - vue
    - 前端
---

## Usage

1. 登陆阿里巴巴 icon 库 [iconfont+](https://www.iconfont.cn/)， 找到合适的 icon     
2. 加入购物车，添加至我的项目   
3. 在我的项目页，点『更多操作』，『编辑项目』，编辑`FontClass/Symbol 前缀 ` 选项，修改为 `el-icon-xx-` ,命名统一规范     
4. 下载至本地，解压缩，之后把以下字体文件复制到你的项目中。
```
    iconfont.css
    iconfont.eot
    iconfont.svg
    iconfont.ttf
    iconfont.woff
```
编辑 `iconfont.css` 文件，修改 `font-family` 字体名称，如 `xx-iconfont`，避免与默认的字体库发生冲突，
然后在 `assets/css` 目录下新建 `icon.css` 文件 , 
```
    [class*=" el-icon-xx"], [class^=el-icon-xx] {
        font-family: xx-iconfont!important;
    }
```

上面 `el-icon-xx` 中的 `xx`为你自定义的名称

5. 在 `App.vue` 和 `main.js` 中分别引入文件。    
```
/*App.vue*/

<style>
    @import "./assets/css/iconfont/iconfont.css";
</style>

/*main.js*/
import './assets/css/iconfont/iconfont.css';
```
最后，就可以在项目中使用自定义的字体文件了。

```
 <i :class="el-icon-xx-home"></i>
```

## TroubleShooting

 - **图标库 和 图标显示成小方块?**  
 
这个可能是字体文件名称跟原来冲突导致的，假如我们没有使用自定义的名称 `xx-iconfont`, 而是默认的 `iconfont` ，那么就可能会出现 icon 不能正确显示的问题。