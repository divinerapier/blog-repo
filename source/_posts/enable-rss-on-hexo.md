---
title: 让你的 Hexo 博客支持 RSS
date: 2020-09-02 20:37:46
tags:
  - hexo
  - rss
---

让博客支持 `RSS` 是一种美好的品德。

* 安装 `RSS` 插件

    ``` bash
    npm install hexo-generator-feed
    ```

* 配置博客 `_config.uml`

    ``` yml
    # Extensions
    plugins:
    - hexo-generator-feed

    #Feed Atom
    feed:
      type: atom
      path: atom.xml
      limit: 20
    ```

* 配置主题 `_config.yml`

    ``` yml
    # RSS订阅
    rss: /atom.xml
    ```

* 客户端订阅

    使用 `RSS` 客户端订阅 `http(s)://${YOUR_BLOG_HOST}/atom.xml`。
