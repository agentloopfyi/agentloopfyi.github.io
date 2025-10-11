---
layout: post
comments: false
title:  "API Design and engineering best practices"
date:   2025-09-15 10:00:00
---

Modern software are usually built API-first. Without APIs your product is isolated within its own boundaries. In this post I collect essential things you need to engineer your APIs elegantly and build robust systems.

<br />
<span class="important">Well-crafted APIs</span>
<br />
<br />

✨ Check this post on [A survival kit to beat APIs interview](https://newsletter.systemdesignclassroom.com/p/a-survival-kit-to-beat-apis-interview) by [Raul Junco](https://substack.com/@rauljuncov). Though this article is written in an "interview prep" way, but it's a masterpiece on elegant API design and engineering, covering nearly all the important topics.
<br />

✅ CRUD operations and HTTP verb <br />
✅ HTTP status codes and the ones which are most frequently used <br />
✅ Stateless APIs <br />
✅ Different ways you can secure your APIs using AuthN and AuthZ, rate limiting, input validation, HTTPS and encrypting data on storage <br />
✅ API Keys (for application identification) vs tokens (for user identification) <br />
✅ Versioning your APIs either by URL based versioning or using custom headers <br />
✅ Pagination using limit-offset, cursor, page numbers or keysets <br />
✅ Rate Limiting and why it is important <br />
✅ Idempotency and ensuring it with an `IdempotencyKey` added by the client for each request, that violates a unique constraint on the database when executed more than once <br />
✅ Ways to make your APIs faster using caching on the client, server, or CDN, distributed caching, query optimization, connection pooling, data serialization and compressiong <br />
✅ Documenting APIs using OpenAPI (Swagger) including endpoint descriptions, req-res examples, query parameters, headers, error codes, etc <br />

One more: `ETag Header` [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/ETag)<br />
<br />
<br />

<span class="important">Secure your APIs</span>
<br />

✨ Next, this post on [Siz strategies to build secure APIs](https://newsletter.systemdesigncodex.com/p/6-strategies-to-build-secure-apis) by [Saurabh Dashora](https://substack.com/@saurabhdashora). Must read! Describes six strategies that help you bullet-proof your APIs:
<br />

✅ Using HTTPs <br />
✅ Rate limiting and throttling <br />
✅ Validation of inputs <br />
✅ AuthN and AuthZ <br />
✅ Role-based access control (RBAC) <br />
✅ Monitoring and logging <br />

<br />
<span class="important">Manage transactions</span>
<br />
<br />

✨ We can't imagine a software system that doesn't use a database.. and with database, comes the responsibility to manage data and transactions. Another article by [Raul Junco](https://substack.com/@rauljuncov) on [Transaction Isolation](https://newsletter.systemdesignclassroom.com/p/transaction-isolation-and-read-and-write-anomalies?r=1m1f9z&utm_campaign=post&utm_medium=web).
<br />

✅ What transactions are <br />
✅ Read Write anomalies: Dirty reads, non-repeatable reads, phantom reads and write skew <br />
✅ Transaction Isolation Levels: Read uncommitted, read committed, repeatable read and serializable. <br />
✅ Performance trade-offs <br />
✅ Choosing isolation levels <br />

<br />
<span class="important">Well maintained caching</span>
<br />
<br />

✨ Improving application and API performance need caching. We need to cache data at many levels. This can quickly start becoming an overhead as we cache more and more data. Wisdom says, you cache what is frequently required, and remove anything that's not required often. However, as scenarios call for it, there could be needs for different strategies. Read on [Seven cache eviction strategies](https://blog.algomaster.io/p/7-cache-eviction-strategies?r=1m1f9z&utm_campaign=post&utm_medium=web) by [Ashish Pratap Singh](https://substack.com/@ashishps).
<br />

✅ Least recently used (LRU) <br />
✅ Least frequently used (LFU) <br />
✅ First In, First Out (FIFO) <br />
✅ Random replacement (RR) <br />
✅ Most recently used (MRU) <br />
✅ Time to live (TTL) <br />
✅ Two-tiered caching <br />

<br />
<span class="important">Deal with entropy</span>
<br />
<br />

✨ Like any other machine, software (an intangible machine with moving parts) quality and maintainability degrade overtime. I asked ChatGPT for a very short summary on *Software Entropy*:

> In software, entropy refers to the increasing disorder, complexity, or unpredictability within a system over time. In software design, high entropy means the code becomes harder to understand, maintain, or extend, often due to quick fixes, poor documentation, or lack of refactoring. This leads to software rot and reduced reliability. 

Read on how to [Embrace Software Entropy](https://thetshaped.dev/p/embrace-software-entropy-imperfect-code-flexibility-maintainability) by [Petar Ivanov](https://substack.com/@petarivanovv9).
<br />

✅ Accept that change is inevitable <br />
✅ Build for now *(very very very important!)* <br />
✅ Stay flexible <br />
✅ Embrace entropy <br />

<br />

The software industry has many people of wisdom who write technical articles like above. Thanks to all the authors. I try to collect such well crafted knowledge articles and document them.
