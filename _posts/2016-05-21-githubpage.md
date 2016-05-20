---
layout: post
title:  "GitHub Page+Jekyllによるブログの構築"
categories: blog
---

### GitHub Page+Jekyllでブログを構築した
GitHub Page+Jekyllがドンピシャで自分の求めている機能を持っていることがわかったので、
移転してみた。

もともとCUI、GUIによる更新方式があるブログをけっこう前から求めていた。
そして更新を管理したり、簡単に内容をクローンできそうなので、SCMベースがよかった。
自作したりもしていたけど、いまいちいけていなかった。

構築にあたりだいたい下記のコマンドを実行した気がする。

{% highlight bash %}
# Jekyllのインストール
sudo dnf install ruby
sudo dnf install ruby-dev
gem install jekyll

# Jekyllプロジェクトの作成
jekyll new kikei.github.io
cd kikei.github.io

# 設定
gem install jekyll-mentions
gem install jekyll-redirect-from
gem install jekyll-feed
vi _config.yml

# Webページに反映
git commit -a
git push
{% endhighlight %}

試行錯誤しながら構築または記事の作成をしたい場合には、GitHubへcommitせずとも、
ローカルで確認ができる。
その場合、JekyllでHTTPサーバを立てて確認する。
次のようなコマンドを実行することで、
`http://127.0.0.1:4000`へアクセスするとWebページを見られるようになる。

{% highlight bash %}
jekyll serve
{% endhighlight %}
