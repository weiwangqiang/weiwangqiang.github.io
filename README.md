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

## 本地测试

搭建 Jekyll 服务器即可，详情参考[Windows上搭建Jekyll本地测试环境](https://blog.csdn.net/diaoxu9717/article/details/101981137)

服务器启动，在博客根路径下执行如下命令即可

```java
jekyll server
```

在服务启动后，会在控制台打印访问地址，比如

```java
Server address: http://127.0.0.1:4000/
```

