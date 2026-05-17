# 亚马逊搜索结果抓取 Python 实战指南：从踩坑到稳定跑通的完整方案

## 为什么抓取亚马逊搜索结果这么让人头疼

我第一次尝试用 Python 抓亚马逊搜索结果的时候，信心满满地写了个 requests + BeautifulSoup 的脚本，结果跑了不到 20 次请求就被封 IP 了。换了代理池，又遇到验证码。折腾了一整个周末，成功率还不到 30%。

亚马逊的反爬机制在电商平台里算是最激进的那一档。动态渲染、行为检测、IP 指纹识别、请求频率限制——每一层都在劝退你。对于做选品调研、价格监控、竞品分析的人来说，手动复制粘贴根本不现实，但自己搭建稳定的抓取系统成本又太高。

后来我开始用 ScraperAPI 这类专门处理反爬的代理服务，才算真正把成功率稳定在 95% 以上。这篇文章就把我这段时间摸索出来的经验整理一下，从原理到代码到套餐选择，一次讲清楚。

## 亚马逊反爬机制到底在防什么

亚马逊的反爬系统不是单一策略，而是多层叠加的防御体系。搞清楚它在检测什么，才能针对性地绕过。

**IP 层面**：同一 IP 短时间内发起大量请求会直接触发封禁。普通数据中心代理的 IP 段早就被标记了，用了跟没用差不多。

**浏览器指纹层面**：请求头里的 User-Agent、Accept-Language、Cookie 行为如果不像真实浏览器，直接返回验证码页面。

**行为模式层面**：请求间隔太规律、不带 Referer、不加载 CSS/JS 资源——这些都是机器人的典型特征。

**地理位置层面**：亚马逊会根据 IP 所在地返回不同的搜索结果和价格，如果你的 IP 地理位置频繁跳变，也会触发风控。

我自己测试下来，单纯靠轮换免费代理 + 随机 User-Agent 这种"土办法"，对付亚马逊的成功率大概只有 40%-60%。一旦并发量上去，基本就废了。

## ScraperAPI 怎么解决这些问题

ScraperAPI 的核心思路是把反爬处理这一层完全抽象掉。你只需要把目标 URL 传给它的 API，它在后端自动处理代理轮换、浏览器指纹模拟、验证码识别、重试逻辑这些脏活。

几个我实际用下来觉得有用的能力：

1. **住宅代理池自动轮换**——不是那种一眼就能被识别的数据中心 IP，而是真实的住宅 IP，亚马逊很难区分这些请求和正常用户

2. **自动处理验证码**——遇到 CAPTCHA 不需要你接入打码平台，ScraperAPI 内部就消化了

3. **地理定位控制**——可以指定从美国、英国、德国等特定国家发起请求，拿到对应站点的本地化结果

4. **结构化数据解析**——针对亚马逊有专门的解析器，直接返回 JSON 格式的商品标题、价格、评分、ASIN 等字段，省掉你自己写 XPath 的时间

对我来说最省心的是第四点。亚马逊的页面结构经常小改，自己维护解析规则是个持续的负担。用它的结构化端点，这部分维护成本直接归零。

## Python 代码实战：三种抓取方式对比

### 方式一：基础 API 调用

最简单的用法，把亚马逊搜索 URL 作为参数传给 ScraperAPI：

```python

import requests

API_KEY = "你的ScraperAPI密钥"

target_url = "https://www.amazon.com/s?k=wireless+earbuds"

params = {

"api_key": API_KEY,

"url": target_url,

"country_code": "us"

}

response = requests.get("http://api.scraperapi.com", params=params)

if response.status_code == 200:

html = response.text

# 用 BeautifulSoup 解析 html

else:

print(f"请求失败，状态码：{response.status_code}")

```

这种方式返回的是原始 HTML，你还需要自己写解析逻辑。适合对页面结构很熟悉、需要提取非标准字段的场景。

### 方式二：结构化数据端点（推荐）

这是我日常用得最多的方式。直接调用亚马逊专用的结构化端点，返回干净的 JSON：

```python

import requests

import json

API_KEY = "你的ScraperAPI密钥"

params = {

"api_key": API_KEY,

"query": "wireless earbuds",

"country": "us",

"tld": "com"

}

response = requests.get(

"https://api.scraperapi.com/structured/amazon/search",

params=params

)

if response.status_code == 200:

data = response.json()

for product in data.get("results", []):

print(f"商品：{product.get('name')}")

print(f"价格：{product.get('price')}")

print(f"评分：{product.get('rating')}")

print(f"ASIN：{product.get('asin')}")

print("---")

else:

print(f"请求失败：{response.status_code}")

```

返回的 JSON 里直接包含商品名称、价格、评分、评论数、ASIN、商品链接等字段。不用写一行解析代码，也不用担心亚马逊改版导致选择器失效。

### 方式三：异步批量抓取

如果你需要一次抓取几百个关键词的搜索结果，同步请求太慢。用 asyncio + aiohttp 可以大幅提升效率：

```python

import asyncio

import aiohttp

API_KEY = "你的ScraperAPI密钥"

keywords = ["wireless earbuds", "bluetooth speaker", "usb c hub"]

async def fetch_search(session, keyword):

params = {

"api_key": API_KEY,

"query": keyword,

"country": "us",

"tld": "com"

}

async with session.get(

"https://api.scraperapi.com/structured/amazon/search",

params=params

) as response:

if response.status == 200:

return await response.json()

return None

async def main():

async with aiohttp.ClientSession() as session:

tasks = [fetch_search(session, kw) for kw in keywords]

results = await asyncio.gather(*tasks)

for keyword, result in zip(keywords, results):

if result:

print(f"[{keyword}] 找到 {len(result.get('results', []))} 个商品")

asyncio.run(main())

```

不过要注意并发数不要设太高。ScraperAPI 不同套餐有并发线程数限制，超了会返回 429 错误。我一般控制在套餐允许的并发数的 80% 左右，留点余量。

## 全套餐对比：选哪个档位合适

ScraperAPI 目前提供免费试用和四档付费套餐。我把官网的信息整理成表格，方便你直接对比：

| 套餐名称 | API 请求额度 | 并发线程数 | 地理定位 | 结构化数据端点 | 月费 | 链接 |
| ------ | ------------ | -------- | ------------ | -------------- | ------ | ---- |
| Free Trial | 5,000 次 | 1 | ✅ | ✅ | $0 | |
| Hobby | 100,000 次 | 10 | ✅ | ✅ | $49/月 | |
| Startup | 500,000 次 | 25 | ✅ | ✅ | $149/月 | |
| Business | 3,000,000 次 | 50 | ✅ | ✅ | $299/月 | |
| Enterprise | 自定义 | 自定义 | ✅ | ✅ | 联系销售 | |

几个选择建议：

- 如果你只是偶尔做选品调研，每天抓几十个关键词，Hobby 够用了

- 做日常价格监控、跟踪几百个 ASIN 的价格变动，Startup 比较合适

- 大规模数据采集、多站点多品类同时跑的，Business 起步

我自己用的是 Startup，每个月大概用掉 30 万次左右的额度，主要是跑美国站和日本站的关键词排名监控。

## 实际使用场景和适配人群

**跨境电商选品**：批量抓取某个品类下的搜索结果，分析价格区间、评分分布、上架时间，快速判断市场竞争程度。我认识一个做亚马逊 FBA 的朋友，每周用脚本跑一次目标品类的 Top 100 商品数据，比手动翻页效率高了不知道多少倍。

**竞品价格监控**：设定定时任务，每天抓取竞品的价格变动。格一降你就能第一时间知道，及时调整自己的定价策略。

**关键词排名追踪**：监控自己的商品在特定关键词下的排名位置变化。这个对优化 Listing 标题和广告投放策略很有参考价值。

**市场调研报告**：抓取多个站点（美国站、欧洲站、日本站）的同品类数据，做跨市场对比分析。

**学术研究和数据分析**：一些做电商研究的团队需要大规模的商品数据集，手动采集根本不可能完成。

不太适合的场景：如果你每个月只需要查几个商品的价格，手动看一下就行了，没必要上 API。

## 我自己用下来的真实感受

说几个我觉得好的地方，也说几个需要注意的点。

**好的方面**：

成功率确实高。我跑了大概两个月的日常任务，整体成功率在 97% 左右。偶尔失败的那 3% 基本都是亚马逊那边页面本身加载异常，不是被封。

结构化端点省了我大量维护解析代码的时间。之前自己写的 BeautifulSoup 解析器，亚马逊每隔一两个月改一次页面结构，我就得跟着改。现在这部分完全不用操心了。

响应速度还行，普通请求大概 3-5 秒返回，结构化端点稍微慢一点，5-8 秒。对于批量任务来说完全可以接受。

**需要注意的**：

每次 API 调用都会消耗请求额度，如果你的代码有 bug 导致重复请求，额度会消耗得很快。建议加上本地缓存逻辑，同一个 URL 24 小时内不重复请求。

结构化端点每次调用消耗的额度比普通 API 调用多（大概是 5-25 倍，取决于具体端点），选套餐的时候要把这个算进去。

并发数超限后会返回 429，代码里一定要加重试逻辑和退避策略，不然批量任务跑到一半就断了。

## 常见问题

**Q：ScraperAPI 只能抓亚马逊吗？**

不是，它支持任意网站的抓取。但针对亚马逊、Google、Walmart 这些大站有专门优化的结构化端点，用起来最省事。我主要用它抓亚马逊和 Google 搜索结果，其他小站一般自己写脚本就够了。

**Q：免费试用的 5000 次额度够测试吗？**

做基础验证够了。我当时用免费额度跑了大概 200 个关键词的搜索结果，确认返回数据的完整性和准确性没问题后才升级付费套餐。不过如果你要测试结构化端点，因为每次消耗额度更多，可能只够跑几十次。

**Q：抓取亚马逊数据合法吗？**

抓取公开可见的商品信息（价格、标题、评分等）在大多数司法管辖区属于灰色地带。ScraperAPI 本身是合法的代理服务工具。体的合规边界建议咨询你所在地区的法律顾问，特别是如果你要大规模商用的话。

**Q：返回的数据格式稳定吗？会不会经常变？**

结构化端点返回的 JSON 格式我用了两个多月没变过。这也是我选择用结构化端点而不是自己解析 HTML 的主要原因——维护成本转嫁给了服务方。

**Q：支持抓取亚马逊商品详情页吗？还是只能抓搜索结果？**

都支持。搜索结果、商品详情页、评论页卖家页面都可以抓。结构化端点目前覆盖了搜索和商品详情两个场景，其他页面用普通 API 模式抓 HTML 再自己解析。

**Q：请求失败会扣额度吗？**

不会。只有成功返回 200 状态码的请求才计入额度消耗。超时、被拦截、服务端错误这些情况不扣。这点我确认过，确实如此。

## 最后说两句

折腾亚马逊数据抓取这件事，我前后后花了不少时间在自建方案上——买代理、搭代理池、写反爬逻辑、维护解析器。算下来每个月花在维护上的时间成本，远超一个 API 服务的订阅费。

如果你的核心需求是稳定地拿到数据，而不是享受和反爬系统斗智斗勇的过程，那直接用 ScraperAPI 这类服务是性价比最高的选择。先用免费额度跑通你的场景，确认数据质量没问题再决定要不要付费。
