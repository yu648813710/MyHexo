# hexo教程

## 安装初始化

### 安装脚手架

` npm install -g hexo-cli `

### 初始化

`hexo init `

### 安装依赖

`npm install`

### 生成页面

`hexo g`

### 本地预览

`hexo s`

## 部署

### 安装部署插件

`npm install hexo-deployer-git --save`

### 修改 _config.yml

配置文件的最后
``` yml
deploy:
  type: git
  repository: git的项目地址
  branch: master
```

### 修改资源文件目录

_config.yml文件

```  yml
url: https://gitee.com/seven_l/
root: /sevenflow.blog/
permalink: :year/:month/:day/:title/
permalink_defaults:
pretty_urls:
  trailing_index: true # Set to false to remove trailing 'index.html' from permalinks
  trailing_html: true # Set to false to remove trailing '.html' from permalinks
```


### 一键部署

`hexo d`

### 其他文档

https://hexo.io/zh-cn/docs/