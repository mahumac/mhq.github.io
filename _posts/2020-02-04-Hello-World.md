---
title: Hello World ！
date: 2020-02-04 12:12:12 +0800
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
description: 
pin: true
tags: [生活] 
---

## 前言

Blog 就这么开通了。

之前一直使用OneNote记录了大量笔记，后续因为OneNote一直不支持markdown格式，只能使用typora将md文件保存在本地。 

很早就想搭建自己的 Blog ，不曾料想踩了很多坑。。。，就放弃了

现在 Blog 总算是搭建好了。

---

## 正文

接下来说说搭建这个博客的技术细节。  

之前就有关注过 [GitHub Pages](https://pages.github.com/) + [Jekyll](http://jekyllrb.com/) 快速 Building Blog 的技术方案，非常轻松时尚。

其优点非常明显：

* **Markdown** 带来的优雅写作体验
* Git workflow ，**Git Commit 即 Blog Post**，非常方便
* 利用 GitHub Pages 的域名和免费无限空间，不用自己折腾主机
	* 如果需要自定义域名，也只需要简单改改 DNS 加个 CNAME 就好了 
* Jekyll 的自定制非常容易，基本就是个模版引擎

---

主题我直接 套用了[jekyll](https://github.com/cotes2020/jekyll-theme-chirpy) 的模板，简单粗暴，不过遇到了很多坑😂，好在都填完了。。。

根据 [Chirpy 的官方文档](https://sspai.com/link?target=https%3A%2F%2Fchirpy.cotes.page%2Fposts%2Fgetting-started%2F)，需要先在 github 中对 [https://github.com/cotes2020/chirpy-starter](https://sspai.com/link?target=https%3A%2F%2Fgithub.com%2Fcotes2020%2Fchirpy-starter) 选择 Use this template 来创建一个新仓库，并将这个仓库命名为 `githubusername.github.io` 。

别的主题方法稍有差别，具体看文档。fork 仓库也是常见的方法。

### clone 到本地

此时，我已经拥有了一个自己的仓库，我将这个仓库 clone 到了本地。在本地仓库的根目录下，打开 terminal 运行：

```bash
$ bundle
```

就可以在本地运行了。

### 修改 config 进行个性化

打开 clone 下来的文件夹，找到名为 `_config.yml` 的文件，进行自定义设置。每个主题的 config 都会大同小异，最关键的一项是 `url` ，填入仓库名称，也就是 `username.github.io` 。其他的按照文件上的说明填写即可。

需要特别提醒评论功能的配置，有 disqus, utterances, giscus 三种方式可选。选择的方式不同，配置的方式也会不同。

```yaml
comments:
  active: # The global switch for posts comments, e.g., 'disqus'.  Keep it empty means disable
  # The active options are as follows:
  disqus:
    shortname: # fill with the Disqus shortname. › <https://help.disqus.com/en/articles/1717111-what-s-a-shortname>
  # utterances settings › <https://utteranc.es/>
  utterances:
    repo: # <gh-username>/<repo>
    issue_term: # < url | pathname | title | ...>
  # Giscus options › <https://giscus.app>
  giscus:
    repo: # <gh-username>/<repo>
    repo_id:
    category:
    category_id:
    mapping: # optional, default to 'pathname'
    input_position: # optional, default to 'bottom'
    lang: # optional, default to the value of `site.lang`
    reactions_enabled: # optional, default to the value of `1`
```

我选择的是 giscus ，进入 [giscus 的网站](https://sspai.com/link?target=https%3A%2F%2Fgiscus.app%2Fzh-CN)，填写仓库等信息，并按要求进行一系列简单的配置。结束后中页面底端会获取到 repo, repo_id, category, category_id 等信息，填入 _config.yml。

## 部署在本地运行

在本地仓库根目录的 terminal 下运行：

```yaml
$ bundle exec jekyll s
```

## 写下第一篇文章

在 *_posts 文件夹内新建一个 markdown 文件，命名格式为`YYYY-MM-DD-title.md`yyyy-mm-dd-hello world.md，例如 `2020-10-10-Hello World.md`，输入以下内容。

```markdown
---
title: hello world
date: 2020-10-10 21:00:00 +0800
categories: []
tags: []
pin: false
---

hello world!
```

上面使用 `---` 隔开的内容是这份文件的元数据，需要包含标题、日期、类别、标签等信息。

## 部署在 github 上

进入仓库的 `settings` 页面，选择 `pages`。

将 `deloyment source` 选择为 `GitHub Actions`。

回到本地仓库，commit and push changes。此时可以进入仓库的 Actions 页面查看 Workflow。

需要等待几分钟，Deployments完成后就可以通过 url 进入自己的博客啦！

## 后记

如果你恰好逛到了这里，希望你也能喜欢这个博客主题，感兴趣的话可以自己动手搭建一个。

—— 后记于 2020.02

