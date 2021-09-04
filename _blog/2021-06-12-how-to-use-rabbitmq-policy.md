---
title: 學會使用 RabbitMQ Policy 彈性的設定 Queue 的參數，幫助未來的自己！
tags: RabbitMQ Policy
cover: /assets/images/2021-06-12-how-to-use-rabbitmq-policy.jpg
article_header:
  # type: overlay
  # theme: dark
  # background_color: '#203028'
  # image:
  #   src: /assets/images/2021-06-12-how-to-use-rabbitmq-policy.jpg
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
    src: /assets/images/2021-06-12-how-to-use-rabbitmq-policy.jpg

---

> *學會使用 RabbitMQ Policy 彈性的設定 Queue 的參數，幫助未來的自己！*

<!--more-->
---

The cover image is photo by <a href="https://unsplash.com/@anyadiary?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Waranya Mooldee</a> on <a href="https://unsplash.com/s/photos/rabbit?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>.

# 在宣告 Queue 時進行參數設定，遇到了什麼問題？

> *對了，新的 Project 要開始了，使用者人數預計會比開發時估計得要多，我們應該要調整與 Microservice 溝通的 RabbitMQ Queue 的設定，增加 message TTL 跟 Queue length，不然短時間的 request 數量超過設定會無法負荷。*

當時我天真的以為修改已經存在的 queue 的設定只是改幾個數字就結束的事情，當我打開 VSCode 把 queue `x-message-ttl` 、 `x-max-length` 參數的值修改並將 server 起起來後，terminal 印出了下面的錯誤訊息。

```tsx
Error: Channel closed by server: 406 (PRECONDITION-FAILED) with message "PRECONDITION_FAILED - inequivalent arg 'x-message-ttl' for queue 'QUEUE' in vhost '/': received '5000' but current is '1000'"
```

幸好錯誤訊息還是蠻好理解的，`inequivalent arg 'x-message-ttl'` 看來是因為剛剛修改的值與目前 queue 設定的值不一樣導致的。

## 為什麼只是想修改 Queue 的參數值也不行

這得提到當初開發時是怎麼設定 Queue 的參數值，程式碼如下：

```tsx
await this.channel.assertQueue(requestQueueName, {
    durable: true,
    arguments: {
        'x-message-ttl': messageTTL,
        'x-max-length': maxQueueLength,
        'x-overflow': overflowBehaviour,
        'x-dead-letter-exchange': this.deadLetterExchangeName,
        'x-dead-letter-routing-key': this.deadLetterQueueName,
    },
});
```

就是在宣告 Queue 時直接把 message TTL、max length... 等值一併設定下去，設定時很直覺很簡單，當想修改時就不是這麼直覺了，於是只好打開 chome 開始 google 解決方法。

## 有什麼解決方法？

- [RabbitMQ change queue parameters on a production system](https://stackoverflow.com/questions/25274182/rabbitmq-change-queue-parameters-on-a-production-system)

- [How to Change RabbitMQ Queue Parameters in Production?](https://teddyma.cn/2016/01/19/how-to-change-rabbitmq-queue-parameters-in-production/)

以上是找到的前幾篇文章，但裡面的解決方法主要著重於在不影響 queue 中的 message 的狀況下做轉移，但對於我們來說 queue 裡面的 message 消失是可以接受的，所以上面文章的方法有點太複雜了。

- [Change arguments of an existing queue](https://stackoverflow.com/questions/36602702/change-arguments-of-an-existing-queue)

於是開始繼續搜尋，然後找到了上面這篇文章，據發文者說下面這段是 RabbitMQ team 對於這個問題的回覆：

> Q: Is it possible to change arguments on an existing queue without deleting and recreating it?

> **A: No.**

很簡潔的 No，後續查到的幾篇文章也都是這個結論，看來想要用同一個 queue name 無法避免刪掉 queue 重建了， 為了不想之後每次修改設定都要手動重覆這個步驟，於是繼續查有沒有方法可以避免這個問題，比較符合我們情況的方法有下面兩項：

### 1. Try Catch
這個方法就是在宣告 queue 的程式碼外，包一層 try catch 抓到 Exception 就刪掉 queue 然後用新的設定重建一個，這個方法的確可以解決我們遇到的問題，但總覺得不是很踏實，如果發生的 Exception 跟 queue 設定衝突無關呢？當然我們可以從 error msg 去判斷，假設 msg 中包含 `PRECONDITION_FAILED - inequivalent arg` 才去刪 queue，但還是擔心之後 error msg 有更改的話就會出問題。

### 2. RabbitMQ Policy
第二個方法則是在搜尋解法時被推薦的方法 `RabbitMQ Policy`。

# 為什麼需要使用 RabbitMQ Policy？

### Policy 能帶給我們的幫助主要有以下幾點：

**1. 彈性修改 queue 的 parameters**

如上面提到想修改在宣告 queue 時建立的 parameters (也就是 client-controlled properties)，無法避免需要刪掉舊的 queue 重新宣告後再重新部署服務的步驟。

**2. 將 parameters 針對同組 queue 或 exchanges 統一管理**

Policy 可藉由正規表示式 (regular expression) 對應到多個 queue。

更詳細的說明可到 RabbitMQ 官方文件看： [Why Policy Exist](https://www.rabbitmq.com/parameters.html#why-policies-exist)。

# 如何使用 Policy 定義 Queue 的 Parameters

### 下面先介紹 policy 主要的 attributes：

[Policy Attributes](https://www.notion.so/a31dfa6b37ae45bfbadcaa49d797b29c)

1. name: required，policy 的名稱，需要是 ASCII-based。
2. pattern: required，一個正規表示式，用來指定要使用這組設定的一組 queue 或  exchange。
3. definition: required，由 key/value 定義的 parameters。
4. priority: optional，指定 policy 的優先順序，值越大越優先，預設是 `0` 。
5. apply-to: optional，值可以是 `exchanges`, `queues` 或 `all` ，預設是 all。

當有符合 policy pattern 的 queue 或 exchange 被建立時，就會自動套用該 policy，但要注意一個 queue 或 exchange 最多能對應到一個 policy，如有多個 policy 對應到同一個 queue 或 exchange，則會優先挑選 priority 值大的，如 priority 相同則會優先選擇建立時間較晚的。

## 建立 Policy 的方法

Policy 是 RabbitMQ Server 端的機制，而建立 policy 的方法有以下幾種：

**1. Web UI**

**2. rabbitmqctl**

**3. HTTP API**

- 如有起 RabbitMQ 則可以輸入對應的 URL 找到 API 的 help page。例如在 local 15672 port 起的 RabbitMQ Server 就可以輸入 [http://localhost:15672/api/index.html](http://localhost:15672/api/index.html) 。

最後是選擇在 application 中使用 HTTP API 去建立 policy，如想了解其他方法一樣可以到 [RabbitMQ 官方文件](https://www.rabbitmq.com/parameters.html) 查看。最後比起文字說明以下直接提供使用 JavaScript 寫的範例讓大家參考會更清楚：

1. URI: `http://{username}:{password}@{rabbitmq_host}:{rabbitmq_port}/api/policies/{virtual_host}/{policy_name}`
2. method: `PUT` 。
3. body: 帶入定義 policy 的參數。

發送 request 的 URI 需要指定 virtual host，而預設的 virtual host 是 `/` ，因此在 URI 中需要編碼成 `%2F`。
{:.info}

```jsx
// * 1. define policy
const requestQueuePolicy = {
  name: 'request_queue_policy',
  arguments: {
    pattern: '^QUEUE-request-.',
    definition: {
      'message-ttl': 5 * 1000,
      'max-length': 30,
      'overflow': 'drop-head',
      'dead-letter-exchange': 'QUEUE-dlx.direct',
      'dead-letter-routing-key': 'QUEUE-dl',
    },
    priority: 0,
    'apply-to': 'queues',
  },
};

// * 2. call API to create defined policy
// specify the policy name in URL and carry the arguments in body
const mqAPIEndpoint = `http://${mqUsername}:${mqPassword}@${mqHost}:${mqPort}/api`;
const vHost = '%2F';
const url = `${mqAPIEndpoint}/policies/${vHost}/${requestQueuePolicy.name}`;
const payload = {
  method: 'PUT',
  headers: {
    Accept: 'application/json',
    'Content-Type': 'application/json',
  },
  body: JSON.stringify(requestQueuePolicy.arguments),
};

const fetchResponse = await fetch(url, payload);
```

## 參考資料

- [Parameters and Policies](https://www.rabbitmq.com/parameters.html)
- [設定 RabbitMQ 的 Mirrored Queues - 讓 Queue 內容可以在多組 RabbitMQ 同步](https://blog.yowko.com/rabbitmq-mirrored-queues/)
