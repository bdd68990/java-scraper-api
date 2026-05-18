# Java爬虫教程：从零搭建数据采集系统，附反封锁实战方案与ScraperAPI深度体验

上个月我接了个私活，要从某电商平台抓三万条商品数据。写了个简单的HttpClient脚本，跑了不到五百条就被封IP了。换了代理池，质量参差不齐，一晚上下来有效数据还不到两千条。折腾了三天，我才意识到一个问题——2024年的反爬机制已经不是换个User-Agent就能糊弄过去的了。

后来我试了ScraperAPI，说实话一开始只是抱着"再试一个工具"的心态，没想到它直接帮我把代理轮换、验证码处理、浏览器指纹这些破事全包了。三万条数据一晚上跑完，成功率在97%以上。

---

## 为什么纯手写Java爬虫越来越难跑通

我写Java爬虫有四年了，早期用Jsoup解析静态页面，后来上了Selenium处理动态渲染，再后来搞了HtmlUnit做无头浏览。技术栈一直在加码，但核心痛点没变过：

**IP被封。** 你用的免费代理十个里面八个是死的。付费代理池每月几百块，质量也不稳定。

**验证码拦截。** reCAPTCHA、hCaptcha、滑块验证，每一种都要单独写对接逻辑。

**JavaScript渲染。** 越来越多页面是SPA架构，不执行JS根本拿不到数据。Selenium太重，资源消耗大。

**请求头指纹检测。** 光改User-Agent没用了，TLS指纹、HTTP/2特征、Canvas指纹都在被检测。

这些问题叠在一起，一个"简单的爬虫项目"能吃掉你80%的时间在对抗反爬，只有20%在处理业务逻辑。

---

## 我的Java爬虫技术栈演进路线

给刚入门的朋友理一下我这几年的路径：

**第一阶段：Jsoup + HttpURLConnection**
适合静态页面，代码简洁。但凡遇到Ajax加载就歇菜。

**第二阶段：OkHttp + Jsoup**
OkHttp的连接池和拦截器机制比原生HttpURLConnection好用太多。配合Cookie管理能模拟登录态。

**第三阶段：Selenium / Playwright**
处理动态渲染页面。代价是启动慢、内存占用高、并发能力差。我跑10个Chrome实例，16G内存直接拉满。

**第四阶段：API化采集**
把"发请求拿HTML"这件事交给专门的服务，自己只负责解析和存储。ScraperAPI就是这个思路——你给它URL，它返回渲染好的HTML，中间的代理、指纹、验证码全部透明处理。

---

## ScraperAPI在Java项目里怎么用

接入方式极其简单，不需要SDK，纯HTTP调用。我贴一下我实际在用的代码结构：

```java
// 核心请求方法
String apiKey = "你的API_KEY";
String targetUrl = "https://example.com/products?page=1";
String apiEndpoint = "https://api.scraperapi.com?api_key=" + apiKey 
    + "&url=" + URLEncoder.encode(targetUrl, "UTF-8")
    + "&render=true";  // 需要JS渲染时加这个参数

OkHttpClient client = new OkHttpClient.Builder()
    .connectTimeout(60, TimeUnit.SECONDS)
    .readTimeout(60, TimeUnit.SECONDS)
    .build();

Request request = new Request.Builder().url(apiEndpoint).build();
Response response = client.newCall(request).execute();
String html = response.body().string();
```

几个我踩过的坑：

- **超时要设长一点。** ScraperAPI后端要做代理轮换和渲染，响应时间比直连慢，建议60秒起步。
- **`render=true`别乱加。** 不需要JS渲染的页面别开这个参数，会多消耗5倍额度。
- **并发控制。** 免费版只支持5个并发，Business套餐到100个。超了会返回429，记得做重试逻辑。

---

## ScraperAPI全套餐对比

我把官网现在售的所有套餐整理成表，方便你按需求选：

| 套餐名 | API调用额度 | 并发数 | 地理定位 | JS渲染 | 月价格 | 适合人群 | 购买链接 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Free | 5,000次/月 | 5 | ✓ | ✓ | $0 | 学习测试、小规模验证 | [免费注册拿5000次额度](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | 100,000次/月 | 10 | ✓ | ✓ | $49/月 | 个人开发者、小项目 | [开通Hobby套餐开始采集](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | 500,000次/月 | 50 | ✓ | ✓ | $149/月 | 中小团队、日常数据监控 | [拿下Startup套餐50并发](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | 3,000,000次/月 | 100 | ✓ | ✓ | $299/月 | 企业级采集、大规模数据管道 | [升级Business套餐解锁百万级调用](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | 自定义 | 自定义 | ✓ | ✓ | 联系销售 | 超大规模、定制需求 | [联系销售定制企业方案](https://www.scraperapi.com/?fp_ref=coupons) |

年付的话每个套餐都有折扣，Hobby年付大概省20%左右。免费版不需要绑卡，注册就能用。

---

## 实战：用Java + ScraperAPI批量采集电商数据

我分享一个我实际跑过的场景——采集某平台的商品列表页，提取标题、价格、评分。

```java
// 批量采集框架
ExecutorService pool = Executors.newFixedThreadPool(5); // 配合套餐并发数
List<Future<String>> futures = new ArrayList<>();

for (int page = 1; page <= 100; page++) {
    final int p = page;
    futures.add(pool.submit(() -> {
        String url = "https://target-site.com/products?page=" + p;
        return callScraperApi(url, false); // 静态页面不开render
    }));
}

for (Future<String> f : futures) {
    String html = f.get();
    Document doc = Jsoup.parse(html);
    // 解析逻辑...
}
```

关键点：线程池大小要和你套餐的并发上限匹配。我用Hobby套餐时开10个线程刚好打满，再多就会触发限流。

---

## 对比自建代理池和用ScraperAPI的真实成本

我算过一笔账。自建方案：

- 代理服务商月费：$80–$200（质量好的住宅代理）
- 验证码打码平台：$30–$50/月
- 服务器资源（跑Selenium集群）：$100+/月
- 维护时间：每周至少3–5小时排查封禁、更换代理

加起来每月$250+，还不算我的时间成本。ScraperAPI的Startup套餐$149搞定，我只需要写解析逻辑。省下来的时间我又接了两个私活。

👉 [用ScraperAPI省掉代理维护的麻烦](https://www.scraperapi.com/?fp_ref=coupons)

---

## 几个提升采集成功率的技巧

这些是我实际跑了几十万次请求总结出来的：

**1. 地理定位参数要用对。** 采集美国站点加`&country_code=us`，采集日本站点加`&country_code=jp`。不加的话ScraperAPI会随机分配出口IP，有些站点会根据IP地区返回不同内容。

**2. 自定义请求头。** ScraperAPI支持传自定义header，有些站点会检查Referer和Accept-Language，加上能提高成功率。

**3. 失败重试用指数退避。** 偶尔会遇到目标站点临时不可用，别傻等，用2s、4s、8s的退避策略重试三次。

**4. 结果缓存。** 同一个URL短时间内别重复请求，浪费额度。我本地用Redis做了一层缓存，TTL设24小时。

---

## 常见问题

### ScraperAPI的免费额度够做什么？

每月5000次调用，够你跑通整个开发流程、测试解析逻辑。我当时用免费额度把三个目标站点的页面结构全摸清了，确认能稳定拿到数据后才升级付费套餐。

### Java里用ScraperAPI需要装SDK吗？

不需要。它本质就是个HTTP API，你用OkHttp、HttpClient、甚至最原始的HttpURLConnection都能调。拼接URL参数就行，零依赖。

### 渲染模式和普通模式怎么选？

看目标页面。打开浏览器开发者工具，禁用JavaScript刷新页面——如果数据还在，用普通模式；如果数据消失了，说明是JS动态加载的，必须开`render=true`。渲染模式消耗额度是普通模式的5倍，别浪费。

### 被目标网站封了怎么办？

这基本不会发生。ScraperAPI的代理池有几百万个IP，自动轮换。我跑了三个月，没遇到过因为IP被封导致的持续失败。偶尔单次请求失败是正常的，重试就好。

### 支持采集哪些类型的网站？

电商、搜索引擎、社交媒体、新闻站点都行。我试过亚马逊、Google搜索结果页、Twitter（现在叫X）的公开页面，都能正常返回。它还有专门的Amazon和Google结构化数据端点，直接返回JSON。

### 年付和月付差多少？

年付大概能省20%。如果你确定要长期用，年付划算。不确定的话先月付跑一两个月看看用量再决定。

---

## 写在最后

Java爬虫这件事，技术门槛其实不高，真正消耗精力的是和反爬机制的对抗。我现在的做法是把"获取HTML"这一层完全交给ScraperAPI，自己只专注数据解析和业务逻辑。代码量少了一半，稳定性反而上去了。

如果你也在被IP封禁、验证码、JS渲染这些问题折磨，建议先用免费额度试试。5000次调用不花一分钱，够你验证整个方案是否可行。

👉 [现在注册ScraperAPI，免费拿5000次API调用额度](https://www.scraperapi.com/?fp_ref=coupons)
