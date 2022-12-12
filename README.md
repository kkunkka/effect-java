# Effect Java

此项目使用 [Docusaurus 2](https://docusaurus.io/)构建, 一个以React开发的静态网站生成器

### 初始化项目

```
$ pnpm install
```

### 本地运行

```
$ pnpm start
```

这个命令启动一个本地开发服务器并打开一个浏览器窗口。大多数更改都是实时反映的，无需重新启动服务器。

### 构建

```
$ pnpm build
```

This command generates static content into the `build` directory and can be served using any static contents hosting service.

### Deployment

Using SSH:

```
$ USE_SSH=true yarn deploy
```

Not using SSH:

```
$ GIT_USER=<Your GitHub username> yarn deploy
```

If you are using GitHub pages for hosting, this command is a convenient way to build the website and push to the `gh-pages` branch.
