# URL 缩短服务

让我们设计一个 URL 缩短服务，类似于 [Bitly](https://bitly.com) 和 [TinyURL](https://tinyurl.com/app) 等服务。

## 什么是 URL 缩短服务？

URL 缩短服务为长 URL 创建一个别名或短 URL。当用户访问这些短链接时，会被重定向到原始 URL。

例如，以下长 URL 可以被转换为一个短 URL。

**长 URL**: [https://karanpratapsingh.com/courses/system-design/url-shortener](https://karanpratapsingh.com/courses/system-design/url-shortener)

**短 URL**: [https://bit.ly/3I71d3o](https://bit.ly/3I71d3o)

## 为什么需要 URL 缩短服务？

URL 缩短服务在分享 URL 时节省了空间。用户也不太可能输入错误的短 URL。此外，我们还可以跨设备优化链接，这使我们能够跟踪单个链接。

## 需求

我们的 URL 缩短系统应满足以下需求：

### 功能需求

- 给定一个 URL，我们的服务应生成一个_更短且唯一_的别名。
- 当用户访问短链接时，应重定向到原始 URL。
- 链接应在默认时间跨度后过期。

### 非功能需求

- 高可用性和最小延迟。
- 系统应具有可扩展性和高效性。

### 扩展需求

- 防止滥用服务。
- 记录重定向的分析和指标。

## 估算和约束

让我们从估算和约束开始。

_注意：确保与面试官确认任何与规模或流量相关的假设。_

### 流量

这是一个读操作为主的系统，所以我们假设读/写比为 `100:1`，每月生成 1 亿个链接。

**每月读/写请求**

对于每月的读请求：

$$
100 \times 100 \space million = 10 \space billion/month
$$

同样，对于写请求：

$$
1 \times 100 \space million = 100 \space million/month
$$

**我们的系统每秒的请求数 (RPS) 是多少？**

每月 1 亿次请求转换为每秒 40 次请求。

$$
\frac{100 \space million}{(30 \space days \times 24 \space hrs \times 3600 \space seconds)} = \sim 40 \space URLs/second
$$

按照 `100:1` 的读/写比，重定向的数量将是：

$$
100 \times 40 \space URLs/second = 4000 \space requests/second
$$

### 带宽

由于我们预计每秒约有 40 个 URL，如果我们假设每个请求的大小为 500 字节，那么写请求的总传入数据将是：

$$
40 \times 500 \space bytes = 20 \space KB/second
$$

同样，对于读请求，由于我们预计约有 4K 次重定向，总传出数据将是：

$$
4000 \space URLs/second \times 500 \space bytes = \sim 2 \space MB/second
$$

### 存储

对于存储，我们假设我们在数据库中存储每个链接或记录 10 年。由于我们预计每月有约 1 亿个新请求，我们需要存储的总记录数将是：

$$
100 \space million \times 10\space years \times 12 \space months = 12 \space billion
$$

同样，如果我们假设每个存储的记录大约为 500 字节。我们将需要大约 6TB 的存储空间：

$$
12 \space billion \times 500 \space bytes = 6 \space TB
$$

### 缓存

对于缓存，我们将遵循经典的 [帕累托原则](https://en.wikipedia.org/wiki/Pareto_principle)，也称为 80/20 法则。这意味着 80% 的请求是针对 20% 的数据，因此我们可以缓存大约 20% 的请求。

由于我们每秒大约有 4K 次读或重定向请求，这相当于每天 3.5 亿次请求。

$$
4000 \space URLs/second \times 24 \space hours \times 3600 \space seconds = \sim 350 \space million \space requests/day
$$

因此，我们每天需要大约 35GB 的内存。

$$
20 \space percent \times 350 \space million \times 500 \space bytes = 35 \space GB/day
$$

### 高级估算

这是我们的高级估算：

| 类型                 | 估算       |
| -------------------- | ---------- |
| 写入（新 URL）       | 40/s       |
| 读取（重定向）       | 4K/s       |
| 带宽（传入）         | 20 KB/s    |
| 带宽（传出）         | 2 MB/s     |
| 存储（10 年）        | 6 TB       |
| 内存（缓存）         | ~35 GB/day |

## 数据模型设计

接下来，我们将专注于数据模型设计。以下是我们的数据库模式：

![url-shortener-datamodel](https://raw.githubusercontent.com/karanpratapsingh/portfolio/master/public/static/courses/system-design/chapter-V/url-shortener/url-shortener-datamodel.png)

最初，我们可以从两个表开始：

**users**

存储用户的详细信息，如 [`name`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A143%2C%22character%22%3A31%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")、[`email`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A143%2C%22character%22%3A39%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")、[`createdAt`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A143%2C%22character%22%3A48%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition") 等。

**urls**

包含新短 URL 的属性，如 [`expiration`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A147%2C%22character%22%3A49%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")、[`hash`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A147%2C%22character%22%3A63%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")、[`originalURL`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A147%2C%22character%22%3A71%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition") 和创建短 URL 的用户的 [`userID`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A147%2C%22character%22%3A90%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")。我们还可以使用 [`hash`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A147%2C%22character%22%3A63%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition") 列作为 [索引](https://karanpratapsingh.com/courses/system-design/indexes) 以提高查询性能。

### 我们应该使用哪种数据库？

由于数据不是强关系型的，NoSQL 数据库如 [Amazon DynamoDB](https://aws.amazon.com/dynamodb)、[Apache Cassandra](https://cassandra.apache.org/_/index.html) 或 [MongoDB](https://www.mongodb.com) 将是更好的选择，如果我们决定使用 SQL 数据库，那么我们可以使用 [Azure SQL Database](https://azure.microsoft.com/en-in/products/azure-sql/database) 或 [Amazon RDS](https://aws.amazon.com/rds)。

_更多详细信息，请参考 [SQL vs NoSQL](https://karanpratapsingh.com/courses/system-design/sql-vs-nosql-databases)。_

## API 设计

让我们为我们的服务做一个基本的 API 设计：

### 创建 URL

此 API 应在给定原始 URL 的情况下在我们的系统中创建一个新的短 URL。

```tsx
createURL(apiKey: string, originalURL: string,

 expiration

?: Date): string
```

**参数**

API Key ([`string`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A164%2C%22character%22%3A18%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")): 用户提供的 API 密钥。

原始 URL ([`string`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A164%2C%22character%22%3A18%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")): 要缩短的原始 URL。

过期时间 ([`Date`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A164%2C%22character%22%3A60%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")): 新 URL 的过期日期 _(可选)_。

**返回**

短 URL ([`string`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A164%2C%22character%22%3A18%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")): 新生成的短 URL。

### 获取 URL

此 API 应从给定的短 URL 中检索原始 URL。

```tsx
getURL(apiKey: string, shortURL: string): string
```

**参数**

API Key ([`string`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A164%2C%22character%22%3A18%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")): 用户提供的 API 密钥。

短 URL ([`string`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A164%2C%22character%22%3A18%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")): 映射到原始 URL 的短 URL。

**返回**

原始 URL ([`string`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A164%2C%22character%22%3A18%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")): 要检索的原始 URL。

### 删除 URL

此 API 应从我们的系统中删除给定的短 URL。

```tsx
deleteURL(apiKey: string, shortURL: string): boolean
```

**参数**

API Key ([`string`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A164%2C%22character%22%3A18%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")): 用户提供的 API 密钥。

短 URL ([`string`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A164%2C%22character%22%3A18%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")): 要删除的短 URL。

**返回**

结果 ([`boolean`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A202%2C%22character%22%3A45%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition")): 表示操作是否成功。

### 为什么需要 API 密钥？

正如你所注意到的，我们使用 API 密钥来防止滥用我们的服务。使用这个 API 密钥，我们可以限制用户每秒或每分钟的请求数量。这是开发者 API 的标准做法，应能满足我们的扩展需求。

## 高级设计

现在让我们对我们的系统进行高级设计。

### URL 编码

我们系统的主要目标是缩短给定的 URL，让我们看看不同的方法：

**Base62 方法**

在这种方法中，我们可以使用 [Base62](https://en.wikipedia.org/wiki/Base62) 对原始 URL 进行编码，Base62 包含大写字母 A-Z、小写字母 a-z 和数字 0-9。

$$
Number \space of \space URLs = 62^N
$$

其中，

[`N`](command:_github.copilot.openSymbolFromReferences?%5B%22%22%2C%5B%7B%22uri%22%3A%7B%22scheme%22%3A%22file%22%2C%22authority%22%3A%22%22%2C%22path%22%3A%22%2Fhome%2Fmarco%2FgitRepo%2FcodeRepo%2F3_application%2F3_servers%2Furl_shorter.md%22%2C%22query%22%3A%22%22%2C%22fragment%22%3A%22%22%7D%2C%22pos%22%3A%7B%22line%22%3A232%2C%22character%22%3A34%7D%7D%5D%2C%22597f7dae-c1e1-418b-838f-7cc35a356ad4%22%5D "Go to definition"): 生成的 URL 中的字符数。

所以，如果我们想生成一个 7 个字符长的 URL，我们将生成约 3.5 万亿个不同的 URL。

$$
\begin{gather*}
62^5 = \sim 916 \space million \space URLs \\
62^6 = \sim 56.8 \space billion \space URLs \\
62^7 = \sim 3.5 \space trillion \space URLs
\end{gather*}
$$

这是这里最简单的解决方案，但它不能保证非重复或抗碰撞的键。

**MD5 方法**

[MD5 消息摘要算法](https://en.wikipedia.org/wiki/MD5) 是一种广泛使用的哈希函数，生成 128 位哈希值（或 32 个十六进制数字）。我们可以使用这 32 个十六进制数字生成 7 个字符长的 URL。

$$
MD5(original\_url) \rightarrow base62encode \rightarrow hash
$$

然而，这为我们带来了一个新问题，即重复和碰撞。我们可以尝试重新计算哈希值，直到找到一个唯一的，但这会增加我们系统的开销。最好寻找更具可扩展性的方法。

**计数器方法**

在这种方法中，我们将从一个维护生成键计数的单一服务器开始。一旦我们的服务收到请求，它可以访问计数器，返回一个唯一的数字并递增计数器。当下一个请求到来时，计数器再次返回唯一的数字，如此继续。

$$
Counter(0-3.5 \space trillion) \rightarrow base62encode \rightarrow hash
$$

这种方法的问题在于它很快会成为单点故障。如果我们运行多个计数器实例，我们可能会有碰撞，因为它本质上是一个分布式系统。

为了解决这个问题，我们可以使用分布式系统管理器，如 [Zookeeper](https://zookeeper.apache.org)，它可以提供分布式同步。Zookeeper 可以为我们的服务器维护多个范围。

$$
\begin{align*}
& Range \space 1: \space 1 \rightarrow 1,000,000 \\
& Range \space 2: \space 1,000,001 \rightarrow 2,000,000 \\
& Range \space 3: \space 2,000,001 \rightarrow 3,000,000 \\
& ...
\end{align*}
$$

一旦服务器达到其最大范围，Zookeeper 将为新服务器分配一个未使用的计数器范围。这种方法可以保证非重复和抗碰撞的 URL。此外，我们可以运行多个 Zookeeper 实例以消除单点故障。

### 键生成服务 (KGS)

正如我们所讨论的，在大规模生成唯一键而不重复和碰撞可能有点挑战。为了解决这个问题，我们可以创建一个独立的键生成服务 (KGS)，提前生成唯一键并将其存储在单独的数据库中以供以后使用。这种方法可以使我们的问题简单化。

**如何处理并发访问？**

一旦键被使用，我们可以在数据库中标记它，以确保我们不重复使用它。然而，如果有多个服务器实例同时读取数据，两个或多个服务器可能会尝试使用相同的键。

最简单的解决方法是将键存储在两个表中。一旦键被使用，我们将其移动到一个单独的表中，并适当加锁。此外，为了提高读取速度，我们可以在内存中保留一些键。

**KGS 数据库估算**

根据我们的讨论，我们可以生成多达约 568 亿个唯一的 6 个字符长的键，这将导致我们需要存储 300 GB 的键。

$$
6 \space characters \times 56.8 \space billion = \sim 390 \space GB
$$

虽然 390 GB 对于这个简单的用例来说似乎很多，但重要的是要记住，这是我们服务生命周期的全部，并且键数据库的大小不会像我们的主数据库那样增加。

### 缓存

现在，让我们谈谈 [缓存](https://karanpratapsingh.com/courses/system-design/caching)。根据我们的估算，我们每天需要大约 35 GB 的内存来缓存 20% 的传入请求。对于这个用例，我们可以在 API 服务器旁边使用 [Redis](https://redis.io) 或 [Memcached](https://memcached.org) 服务器。

_更多详细信息，请参考 [缓存](https://karanpratapsingh.com/courses/system-design/caching)。_

### 设计

现在我们已经确定了一些核心组件，让我们做出我们系统设计的初稿。

![url-shortener-basic-design](https://raw.githubusercontent.com/karanpratapsingh/portfolio/master/public/static/courses/system-design/chapter-V/url-shortener/url-shortener-basic-design.png)

它是如何工作的：

**创建新 URL**

1. 当用户创建新 URL 时，我们的 API 服务器向键生成服务 (KGS) 请求一个新的唯一键。
2. 键生成服务向 API 服务器提供一个唯一键并将键标记为已使用。
3. API 服务器将新 URL 条目写入数据库和缓存。
4. 我们的服务向用户返回 HTTP 201（已创建）响应。

**访问 URL**

1. 当客户端导航到某个短 URL 时，请求被发送到 API 服务器。
2. 请求首先命中缓存，如果在那里找不到条目，则从数据库中检索并发出 HTTP 301（重定向）到原始 URL。
3. 如果在数据库中仍然找不到键，则向用户发送 HTTP 404（未找到）错误。

## 详细设计

现在是讨论我们设计的细节的时候了。

### 数据分区

为了扩展我们的数据库，我们需要对数据进行分区。水平分区（也称为 [Sharding](https://karanpratapsingh.com/courses/system-design/sharding)）可以是一个很好的第一步。我们可以使用以下分区方案：

- 基于哈希的分区
- 基于列表的分区
- 基于范围的分区
- 复合分区

上述方法仍可能导致数据和负载分布不均，我们可以使用 [一致性哈希](https://karanpratapsingh.com/courses/system-design/consistent-hashing) 来解决这个问题。

_更多详细信息，请参考 [Sharding](https://karanpratapsingh.com/courses/system-design/sharding) 和 [一致性哈希](https://karanpratapsingh.com/courses/system-design/consistent-hashing)。_

### 数据库清理

这是我们服务的一个维护步骤，取决于我们是否保留过期条目。如果我们决定删除过期条目，我们可以通过两种不同的方式进行：

**主动清理**

在主动清理中，我们将运行一个单独的清理服务，定期从存储和缓存中删除过期链接。这将是一个非常轻量级的服务，如 [cron 作业](https://en.wikipedia.org/wiki/Cron)。

**被动清理**

对于被动清理，当用户尝试访问过期链接时，我们可以删除条目。这可以确保我们的数据库和缓存的延迟清理。

### 缓存

现在让我们谈谈 [缓存](https://karanpratapsingh.com/courses/system-design/caching)。

**使用哪种缓存驱逐策略？**

正如我们之前讨论的，我们可以使用 [Redis](https://redis.io) 或 [Memcached](https://memcached.org) 解决方案，并缓存 20% 的日常流量，但哪种缓存驱逐策略最适合我们的需求？

[最近最少使用 (LRU)](<https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU)>) 可以是我们系统的一个好策略。在这种策略中，我们首先丢弃最近最少使用的键。

**如何处理缓存未命中？**

每当发生缓存未命中时，我们的服务器可以直接访问数据库并使用新条目更新缓存。

### 指标和分析

记录分析和指标是我们的扩展需求之一。我们可以在数据库中存储和更新与 URL 条目相关的元数据，如访问者的国家、平台、查看次数等。

### 安全性

为了安全起见，我们可以引入私有 URL 和授权。可以使用一个单独的表来存储有权访问特定 URL 的用户 ID。如果用户没有适当的权限，我们可以返回 HTTP 401（未授权）错误。

我们还可以使用 [API 网关](https://karanpratapsingh.com/courses/system-design/api-gateway)，因为它们可以支持授权、速率限制和负载均衡等功能。


## 识别和解决瓶颈

![url-shortener-advanced-design](https://raw.githubusercontent.com/karanpratapsingh/portfolio/master/public/static/courses/system-design/chapter-V/url-shortener/url-shortener-advanced-design.png)

让我们识别和解决设计中的瓶颈，如单点故障：

- “如果 API 服务或密钥生成服务崩溃了怎么办？”
- “我们将如何在组件之间分配流量？”
- “我们如何减少数据库的负载？”
- “如果密钥生成服务使用的密钥数据库失败了怎么办？”
- “如何提高缓存的可用性？”

为了使我们的系统更具弹性，我们可以采取以下措施：

- 运行多个服务器和密钥生成服务实例。
- 在客户端、服务器、数据库和缓存服务器之间引入[负载均衡器](https://karanpratapsingh.com/courses/system-design/load-balancing)。
- 为我们的数据库使用多个只读副本，因为这是一个读密集型系统。
- 为密钥数据库设置备用副本，以防其失败。
- 为我们的分布式缓存使用多个实例和副本。