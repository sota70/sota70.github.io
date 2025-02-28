---
date: 2025-02-28T17:19:28+09:00
# description: ""
# image: ""
lastmod: 2025-02-28
showTableOfContents: false
# tags: ["",]
title: "Nameless Cookieを用いたクッキーチェックのバイパス"
type: "post"
---

## はじめに

Nameless Cookieは名前無しクッキーとも言われ、名前が存在せず値だけが存在するクッキーのことです。
今回は名前無しクッキーを利用して特定のクッキーのセットを拒否するチェックをバイパスする方法を解説します。

## Nameless Cookieとは

Nameless Cookieとは名前が存在せず、値だけが存在するクッキーのことです。
```
Cookie: =value
```
ですがこれは非推奨のクッキーです。
これはRFC6265のクッキーの定義を見ることでも分かります。
https://datatracker.ietf.org/doc/html/rfc6265#section-4.1
```
set-cookie-string = cookie-pair *( ";" SP cookie-av )
cookie-pair       = cookie-name "=" cookie-value
cookie-name       = token
```
tokenはRFC2616から参照しています。
https://datatracker.ietf.org/doc/html/rfc2616#section-2.2
```
token          = 1*<any CHAR except CTLs or separators>
```
このことからクッキーの名前は最低でも特殊文字を除く1文字が必要だということが分かります。
Nameless CookieはRFC6265bisで明確に使うべきでないと主張しています。
https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-rfc6265bis
```
Per the grammar above, servers SHOULD NOT produce nameless cookies (i.e.: an empty cookie-name) as such cookies may be unpredictably serialized by UAs when sent back to the server.
```
ですが現状名前無しのクッキーを使用することができてしまいます。

## 特定のクッキー設定を拒否するチェックのバイパス

以下のような任意のクッキーを設定できるエンドポイントにおいて
特定のクッキーは設定することができないようなチェック機構が存在しているとします。
```javascript
app.get("/setcookie", (req, res) => {
    const { cookie } = req.query;
    console.dir(cookie);
    if (!cookie) {
        res.send("Invalid cookie");
        return;
    }
    const cookies = cookie.split(";");
    for (const pair of cookies) {
        if (pair.startsWith("connect.sid=")) {
            res.send("Invalid cookie detected");
            return;
        }
    }
    res.setHeader("Set-Cookie", cookie);
    res.send("Cookie sent");
});
```
connect.sidという名前のクッキーは設定できないようになっています。
ですが名前無しクッキーを使うと面白いことが起きます。
以下はconnect.sid=SESSIONIDを値に持つ名前無しクッキーです。
```
=connect.sid=SESSIONID
```
これは以下のような形でCookieヘッダに付きます。
```
Cookie: connect.sid=SESSIONID
```
そしてこれはconnect.sidを名前に持ち、SESSIONIDを値に持つクッキーと解釈されます。
このようにして名前無しクッキーを利用することで禁止されているクッキーでも設定することができます。

## おまけ

この仕様を知ったのはChromiumのフォーラムを閲覧していたのがきっかけです。
https://issues.chromium.org/issues/40060539
このissueの中で名前無しの問題について言及されています。
