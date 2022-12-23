---
title: "NginxリバースプロキシでALBやS3などへのルーティング設定をした際に499エラーが発生した件について"
date: 2022-12-08T00:00:00+09:00
lastmod: 2022-12-17T00:00:00+09:00
tags: ["Nginx", "AWS", "Elastic Load Balancing(ELB)", "Amazon S3"]
draft: false
externalUrl: https://qiita.com/chacco38/items/2e96763a62bbfd2fccbc
---

みなさん、こんにちは。Nginxでリバースプロキシを立ててパスベースルーティングなどでALBやS3などのURLへ振り分け設定をしていたところ、Nginxが499を返してくるという事象が発生したので、今回はこちらの事象の原因と解決方法について紹介していきたいと思います。

## 499が発生した原因

事象発生時のリバースプロキシ設定は次のような定義となっていたのですが、この定義だとNginxのproxy_passに指定しているURLの名前解決はNginx起動時のみに行われ、それ以降はそのIPアドレスを使い続けるという挙動となってしまうことがわかりました。

```txt:499を発生させてしまう/etc/nginx/conf.d/*.confの例
server {
    :
  (省略)
    :

  location /api/ {
    proxy_pass http://XXXXXXXX.ap-northeast-1.elb.amazonaws.com/;
  }
}
```

そのため、何らかの要因でALBやS3などのIPアドレスに変更が生じたものの、それにNginxが追従できずに古いIPアドレスへの転送を続けてしまったため499を返してくるという事象へと至っておりました。

## 解決方法

名前解決をNginx起動時のみにしていることが原因だったため、次の例のように`resolver`ディレクティブを定義し、`valid`オプションを使って適宜キャッシュの時間を調整することによって解決することができました。めでたしめでたし。

```txt:IPアドレス変更に追従できる/etc/nginx/conf.d/*.confの例
server {
    :
  (省略)
    :

  location ~ /api/(.*) {
    resolver    ${NAMESERVER} valid=30s;
    set         $endpoint_url http://XXXXXXXX.ap-northeast-1.elb.amazonaws.com/;
    proxy_pass  $endpoint_url$1$is_args$args;
  }
}
```

{{< alert >}}
setで変数化している理由ですが、proxy_passへは変数で指定しないと名前解決をしに行かない、といった事例があったためsetにて変数化を行っています。
{{< /alert >}}

{{< alert >}}
locationおよびproxy_passの指定値が変わった理由ですが、setで変数化したことによって/api/以降の文字列がすべてなかったことにされてしまう挙動となったため、それを解決するためにlocationおよびproxy_passの指定値を変更しています。
{{< /alert >}}

## 終わりに

Nginxでリバースプロキシを立てていて原因不明の499エラーがでて困ってます、といった方はぜひ参考にしてみてはいかがでしょうか。以上、NginxリバースプロキシでALBやS3などへのルーティング設定をしていたときに発生した499エラーの原因と解決方法のご紹介でした。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- NGINX は NGINX, Inc. の登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
