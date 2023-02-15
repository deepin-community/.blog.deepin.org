## deepin 社区博客

deepin 社区公共博客平台。

## 用法说明

此博客平台使用了 [HUGO](https://gohugo.io/) 作为文章平台工具，进行博客文章的静态生成，并最终利用 GitHub Pages 呈现网页。故整个过程你只需按照 HUGO 的正常使用方式进行使用即可。

若未安装 HUGO，您可以从 [HUGO 官方 GitHub 仓库的 Release 页面](https://github.com/gohugoio/hugo/releases)获取适用于您设备的 HUGO 版本，并放置其于 PATH 内以便调用。

安装就绪后，大致常用命令如下（在此源码仓库的根目录下执行）：

```
$ hugo server # 启动一个本地服务器，预览目前状态下的网站内容
$ hugo new posts/my-post.md # 创建一篇新文章，以便进行编辑
$ hugo server -D # 启用本地服务器，并且能够预览状态为草稿（`draft: true`）的文章
$ hugo -D # 生成静态页面（如果需要），生成的文件将位于 public 目录下
```

创建文章时，创建格式为 `<分类>/<文件名>.<格式>`，上面给出的例子中，分类为 posts，文件名为 my-post，格式为 md （markdown）。创建文章后，默认会使用 [YAML front-matter](https://gohugo.io/content-management/front-matter/) 标记文章的一些元信息，请留意 draft 草稿状态的文章最终不会显示。

若要获取 HUGO，请参考[官方文档给出的安装方式](https://gohugo.io/getting-started/installing)。

## 文章内容指导建议

向此博客平台投递文章需要经过相关人员的 review，合入后即投递完成。需要注意的是，向此处投递的文章必须是与 deepin 社区发展所相关的文章。对于与 deepin 开源社区贡献相关的个人感想或随笔等博客，可以考虑发布到 planet.deepin.org 聚合平台，此博客平台的文章也会出现 planet.deepin.org 之中。

在发布您的文章时，建议使用标签来标记您文章所相关的主题，以便读者更方便的查阅您的文章。例如 `tags: ["持续集成"]` 或 `tags: ["指南文档", "CMake"]`。另外也建议在文章的元信息中附带作者信息，例如 `authors: ["张三"]`（可为多人）。

## 改善此项目的样式风格

此项目使用了 [geekblog](https://themes.gohugo.io/hugo-geekblog/) 主题（基于 `v0.5.3`，有改动）。若你有任何建议，可以进行讨论并修改或提供更适合的主题。
