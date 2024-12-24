---
title: 'Conoha × Cloudflareでお手軽デプロイ'
emoji: '☁️'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['Conoha', 'cloudflare']
published: true
---

## Conoha 使ってますか？

### Conoha とは

清楚かわいいこのはちゃんが応援団長を務めるレンタルサーバーサービス！

VPS はもちろん、サイト運営のための「Conoha Wing」やゲーム用の「for GAME」などなど、用途毎に選べるのが特徴かなって個人的に思ってます

### Conoha VPS の料金体系

Conoha とほかのサービスの料金を比べてみました

他のサービスと比較する条件  
・メモリ 1GB
・CPU 2Core
・SSD 100GB

|              |                |                       |
| ------------ | -------------- | --------------------- |
| Conoha       | ￥ 1.9 円 / h  | 1GB / 2Core/ 100GB    |
| Google Cloud | ￥ 1.5 円 / h  | 1GB / 1vCPU / 10GB    |
| AWS          | ￥ 2.46 円 / h | 0.5GB / 2vCPU / 100GB |

:::details 他のプラットフォームの料金について
計算リソースのほかにも IP アドレスやデータの通信に料金がかかったりすることがありますが、詳しくないので概算で比較させてください...
:::

## バックエンドサーバーに困っていませんか

手軽に Web でアプリ作ってみたいってことがよくあると思います()  
その中で、やっぱり一番ネックになるのは API や DB をどこに置くかではないでしょうか
フロントエンドを簡単に公開できるサービスって Cloudflare Pages や Vercel 等々あるんですけど、バックエンドまで面倒見てくれるサービスってあんまりないんですよね...

そこでせっかくバックエンドを構築するなら**Conoha にしましょう！！！**

### 今回のインフラ構成

![](/images/20241224/infra.png)

### Conoha と Cloudflare の接続

Cloudflare で DNS を管理している場合は、Conoha のパブリック IP アドレスを A レコードに設定してもいいのですが自分で使うアプリを外に公開したくないので今回は Cloudflare Tunnel を使って Cloudflare と Conoha の間にトンネルを張ったうえで Pages とともに Cloudflare Access でアクセス制限を付けたいと思います。

:::details Cloudflare Tunnel とは

> 物理的な世界では、トンネルは通常では越えられない地形や境界を越えるための方法です。同様に、ネットワークの世界でのトンネルは、そのネットワークでサポートされていないプロトコルを使用して、ネットワーク上でデータを転送するための方法です。トンネルは、パケットをカプセル化することによって機能します。パケットは他のパケットでラッピングされます。（パケットとは、送信先でより大きなファイルに再度組み立てることができる小さなデータ片のことです）
> トンネリングは仮想プライベートネットワーク（VPN）でよく使用されます。ネットワーク間の効率的で安全な接続を確立し、サポートされていないネットワークプロトコルの使用を可能にし、場合によってはユーザーがファイアウォールをバイパスすることができるようにもします。
> [トンネリングとは | Cloudflare](https://www.cloudflare.com/ja-jp/learning/network-layer/what-is-tunneling/)

らしいです。
簡単にまとめると Cloudflare と Conoha の間に専用の道を作ってあげることでセキュリティの面は Cloudflare に任せて Conoha のサーバーを外部の通信から守るって感じです
:::

## Conoha で VPS を建てる

[ダッシュボード](https://manage.conoha.jp/Dashboard/)から作っていきましょう
今回は以下の構成で作成しました

・OS 　 Ubuntu
・メモリ 1GB

### Cloudflare Tunnel の設定をする

Cloudflare にログインして、Zero Trust > ネットワーク > Tunnels から新しいトンネルを作成します。

:::message
Cloudflare のページに表示されたコマンドをコピーするときに Conoha のページから開けるコンソールではうまくいかないことがあるので SSH で繋いで操作するのがおすすめです
:::

トンネル一覧に表示されたら成功です ✨

![](/images/20241224/cf-tunnel-success.png)

### Web サーバーを建ててみる

本当は API サーバーを建ててみたいところですが今回は Cloudflare を介して Conoha にアクセスすることの体験するために簡単な Web サーバーを建ててみましょう！

#### apt を最新版に更新する

```bash
sudo apt update
```

#### nginx のインストール

```bash
sudo apt install nginx
```

#### nginx が動いていることを確認してみる

```bash
curl http://localhost
```

```bash
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Nginx が動作していることが確認できました！

しかし！
自身の環境で`http://<VPSのIPアドレス>`にアクセスしても、何も表示されないはずです。
これには、Conoha のセキュリティグループが関係しています。

### セキュリティグループについて

> セキュリティグループは、仮想ファイアウォールとして動作しますので、任意のポートの開閉や接続元制限ができます。

Conoha のセキュリティグループ機能は画面上でポートの開閉などができる機能です（UI からできるのすごく便利）

今は、ssh 以外のセキュリティグループを追加していないため http でアクセスすることができません。

このセキュリティグループに追加してもいいのですが、「せっかくなら https 化したい！」と思ってもなかなか難しいのです...

### Cloudflare の力を借りる

そこで、先ほどセットアップした cloudflare の力を借りたいと思います。
Cloudflare Tunnel のダッシュボードから「パブリックホスト名を追加する」と進みサブドメインを決めて、ドメインを選択します。

![](/images/20241224/cf-tunnel-url.png)

また、サービスには`http://localhost`を設定します
これは、VPS 内部にインストールしたトンネルの出口から見た URL を指定します。

今回は行っていませんが、例えば、プライベートネットワーク上にトンネルの出口を１つ作ることでそのネットワーク内のすべてのリソースに URL を設定することができます。

## アクセスしてみる

![](/images/20241224/nginx.png)

無事に自分の設定した URL に https でアクセスすることができました！

## 最後に

ぜひ、Conoha x Cloudflare で自分だけのサイトを作りましょう！！！
自分は家族用のアプリを作ったりと使い倒しています～！
（最近は、オンプレ環境を作り始めっちゃったので VPS を使う機会は減っちゃったけど）

何よりも、スマホのダッシュボードのこのはちゃんがかわいい！
ちなみに自分はパソコンの壁紙もこのはちゃんのイラストを使ってます ♪

[壁紙サイト](https://conoha.mikumo.com/wallpaper/)
