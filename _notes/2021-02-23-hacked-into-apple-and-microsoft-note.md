---
title: 藉由 NPM Packages 駭入科技巨頭筆記
tags: Supply Chain Attack Security npm
cover: /assets/images/2021-02-23-hacked-into-apple-and-microsoft-note-luis-villasmil-S2qA7JhjI6Y-unsplash.jpg
article_header:
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
    src: /assets/images/2021-02-23-hacked-into-apple-and-microsoft-note-luis-villasmil-S2qA7JhjI6Y-unsplash.jpg

---

> *使用 npm Packages 駭入科技巨頭筆記*

<!--more-->
---

Photo by <a href="https://unsplash.com/@villxsmil?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Luis Villasmil</a> on <a href="https://unsplash.com/s/photos/security?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>.

# The Idea

- 在 PayPal 的 `package.json` 發現上面列有 PayPal 內部使用的 private packages。

# 問題

> What happens if malicious code is uploaded to npm under these names?

> Is it possible that some of PayPal’s internal projects will start defaulting to the new public packages instead of the private ones?

- 大公司有嚴密的防護(管理 outbound...)，所以無法讓其從內部隨便 access 外面的資源。

# 如何破解

> Thankfully, npm allows arbitrary code to be executed automatically upon package installation, allowing me to easily create a Node package that collects some basic information about each machine it is installed on through its `preinstall` script.

- npm 可以在安裝 package 時自動執行一些 code，使用 `preinstall` script，這個 script 在安裝 package 時就會執行。 npm 的這個功能之前也被利用過([We Need a Solution to NPM Trojans - post-install hell](https://www.youtube.com/watch?v=CDVDWJ5HE2k))，或許應該透過上傳 npm package 需要得到 npm 官方認證等方式去修改這個問題。總之在安裝 npm package 前應該要保證是可信的。
- 作者使用此去收集安裝這個 package 的機器的基本資訊，如 hostname, username 等等。
- regular traffic 如果是要去不被允許的地方會被 firewall / IDS 阻擋，無法透過 post 把資料傳出去，所以作者是透過 **DNS Query**。

## 步驟
1. 發現 private package ，並在 npm 上傳同樣名稱的 public package
2. Use `preinstall` script of npm package.
3. 建立 custom DNS server
4. 在安裝 malicious package 的機器，透過 DNS Query 把取得的資訊帶給自己建的 custom DNS server，DNS Query 會被防火牆視為合法的，因此會去詢問外部的 DNS provider 直到找到作者建立的 DNS server。
- 資訊如何被帶出的? 透過把資訊插入 query 的 URL
- 假設 domain name 是 xxx.attacker.com，則讓安裝 malicious package 的機器 bobPC 發送尋找 bobPC.attacker.com 的 DNS Query。

# Reference
1. [Dependency Confusion: How I Hacked Into Apple, Microsoft and Dozens of Other Companies](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610)
2. [I can’t believe how smart this is! He Hacked Into Apple and Microsoft with this genius trick.](https://www.youtube.com/watch?v=43g3PF-e4ik)
