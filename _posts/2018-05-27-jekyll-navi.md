---
layout: post
title:  "GitHub Page+Jekyllに前回/次回の記事リンクを追加した"
categories: blog
---

GitHub Page+Jekyllで運用しているブログに対し、前回の記事、次回の記事へのリンクを設置した。

記事が多くなると、シリーズなど関連する記事を探しにくくなるはずだ。

そこで、例えば今読んでいる内容が複数記事にまたがっているとして、
これが中編ならば、「前編」「後編」へのリンクを表示することにした。

2ファイルを編集してコミットしたらできた。

### 1. ファイル変更

_layouts/post.html に以下を追加。

```
<div class="post-navigation">
  <div class="post-navigation-prev">
    {% if page.previous.url %}
    <a href="{{ page.previous.url }}">&laquo; {{ page.previous.title }}</a>
    {% endif %}
  </div>
  <div class="post-navigation-next">
    {% if page.next.url %}
    <a href="{{ page.next.url }}">{{ page.next.title }} &raquo;</a>
    {% endif %}
  </div>
</div>
```

_sass/_layout.scss に以下を追加。

```
.post-navigation {
    display: flex;
    justify-content: space-between;
    
    .post-navigation-prev {
	width: 50%;
	text-align: left;
    }
    
    .post-navigation-next {
	width: 50%;
	text-align: right;
    }
}
```

### 2. 参考

- [Jekyll – how to link to next/previous post on your blog - David Elbe](https://david.elbe.me/jekyll/2015/06/20/how-to-link-to-next-and-previous-post-with-jekyll.html)
