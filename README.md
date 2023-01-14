# android开发者#pythona爱好者



## 博客搭建

使用Jekyll框架+Hux 风格（感谢[Hux 作者](https://github.com/Huxpro)）

目录结构

```java
.
├── _config.yml 
├── _drafts 
├── _includes
|   ├── header.html
├── _layouts
|   ├── default.html
|   └── post.html
├── _posts
└── index.html 
└── tags.html
```

- **_config.yml **: 用于配置博客的文件
- **_drafts**：存放草稿的地方
- **_includes**：存放Head，foot 的html文件，或者markdown
- **_layouts**：存放页面布局
- **_posts**：存放博客，用于发布
- **index.html**：首页
- **tags.html**：tag页

如果需要在头部新建一个TAB链接，可以直接在根目录下创建一个html文件，类似tags.html
