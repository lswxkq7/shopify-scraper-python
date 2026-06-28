# Scrape Shopify Products: 完整实战指南——products.json 技巧、Python 代码、反封锁方案与工具对比，手把手教你抓取竞品数据

做电商的人，多少都会遇到这样的时刻：

盯着竞品的 Shopify 店铺，想知道他们到底上了多少 SKU，定价策略是什么，哪些产品卖得好、哪些已经下架——但一个个点进去看？太慢了。手动整理成表格？算了吧。

其实 Shopify 这个平台有一个公开的"后门"，让你不用写多复杂的代码，就能把一整家店的产品数据一网打尽。这篇文章就来聊这件事，从最简单的 `/products.json` 技巧，到规模化抓取时怎么不被封，一步步说清楚。

---

## Shopify 的"公开秘密"：/products.json 端点

先说个很多人不知道的事：**每一家 Shopify 店铺，默认都开放了一个 JSON 数据接口**。

你只需要在任意 Shopify 店铺的域名后面加上 `/products.json`，就能直接拿到该店铺的产品数据，不需要登录，不需要 API 密钥，返回的是结构整洁的 JSON。

比如：


https://某shopify店铺域名.com/products.json


你会看到每个产品的标题、价格、变体（颜色、尺码等）、库存状态、图片链接、标签……一应俱全。大约有 40+ 个字段，足够做竞品分析、价格追踪、选品研究。

这个端点已经存在很多年了，Shopify 从未主动关闭它——因为它本来就是公开的产品展示数据。根据目前的法律判例（包括 hiQ v. LinkedIn 案和 Meta v. Bright Data 2024 年的裁决），抓取公开可访问的数据通常被认定为合法行为。当然，具体情况还是要参考目标店铺的服务条款和所在地区的法规。

---

## 用 Python 抓取 Shopify 产品：从零开始

### 方法一：最简单的单次请求

先安装 `requests` 库：

bash
pip install requests


然后几行代码就搞定基础抓取：

python
import requests
import json

shop_url = "https://目标店铺域名.com"
response = requests.get(f"{shop_url}/products.json")
products = response.json()["products"]

for product in products:
    print(product["title"], product["variants"][0]["price"])


运行一下，你会发现整个店铺第一页的产品数据全在眼前了。

### 方法二：处理分页，抓取全量数据

Shopify 默认每次返回最多 250 条产品，大一点的店铺需要分页抓取：

python
import requests

def scrape_shopify(url):
    all_products = []
    page = 1
    while True:
        json_url = f"{url}/products.json?limit=250&page={page}"
        response = requests.get(json_url)
        products = response.json().get("products", [])
        if not products:
            break
        all_products.extend(products)
        page += 1
    return all_products

products = scrape_shopify("https://目标店铺域名.com")
print(f"共抓取 {len(products)} 个产品")


这个脚本会一直翻页直到没有更多数据为止，适合中小规模的店铺全量抓取。

### 方法三：导出为 CSV

抓完数据，通常得整理成表格方便分析：

python
import csv

with open("shopify_products.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["title", "price", "sku", "available"])
    writer.writeheader()
    for p in products:
        for variant in p["variants"]:
            writer.writerow({
                "title": p["title"],
                "price": variant["price"],
                "sku": variant.get("sku", ""),
                "available": variant["available"]
            })


---

## 会遇到的坑：封 IP 和 JavaScript 渲染

说完轻松的部分，得聊聊现实。

`/products.json` 这招对大多数店铺都管用——有测试显示约 71% 的 Shopify 店铺这个端点是开放的。但剩下的那 29%，要么启用了 Cloudflare 保护，要么做了自定义安全配置，直接请求会被拒绝或返回 404。

还有另一种情况：如果你需要抓取的不只是产品列表，而是具体的产品详情页（比如抓取页面上的评分、评论、某些动态加载的内容），这时候 `/products.json` 就不够用了，你需要渲染 JavaScript 才能看到完整内容。

这两种情况加在一起，就是大部分人做 Shopify 大规模抓取时卡住的地方。

### 解决方案：用 ScraperAPI 处理代理和反爬

这时候就需要一个能帮你处理所有脏活的工具了。

👉 [ScraperAPI 免费试用（含 5,000 次 API 调用）](https://www.scraperapi.com/?fp_ref=coupons)

ScraperAPI 的使用方式极其简单——你只需要把目标 URL 传给它的 API 端点，剩下的事全由它来：

- 自动轮换 IP（40M+ 代理池，覆盖 50+ 国家）
- 处理 CAPTCHA 和反爬机制
- 支持 JavaScript 渲染（加上 `render=true` 参数）
- 自动重试失败的请求

代码层面只需要改一行——把原来的请求 URL 换成通过 ScraperAPI 的代理格式：

python
import requests

API_KEY = "你的ScraperAPI密钥"
target_url = "https://目标店铺域名.com/products.json"

response = requests.get(
    f"https://api.scraperapi.com?api_key={API_KEY}&url={target_url}"
)
products = response.json()["products"]


如果遇到需要 JS 渲染的页面：

python
response = requests.get(
    f"https://api.scraperapi.com?api_key={API_KEY}&url={target_url}&render=true"
)


就这么简单。原来可能要花几天时间搭建的代理管理和反爬基础设施，现在一行代码搞定。

---

## 多竞品同时监控：批量抓取 Shopify 数据

做竞品研究通常不是只盯一家店，而是要同时监控多个竞品。下面是一个可以直接用的批量抓取框架：

python
import requests
import json
import time

API_KEY = "你的ScraperAPI密钥"

COMPETITORS = [
    "allbirds.com",
    "gymshark.com",
    "fashionnova.com",
]

def scrape_shopify_store(domain):
    all_products = []
    page = 1
    while True:
        url = f"https://{domain}/products.json?limit=250&page={page}"
        response = requests.get(
            f"https://api.scraperapi.com?api_key={API_KEY}&url={url}"
        )
        data = response.json().get("products", [])
        if not data:
            break
        all_products.extend(data)
        page += 1
    return all_products

for domain in COMPETITORS:
    print(f"正在抓取 {domain}...")
    products = scrape_shopify_store(domain)
    with open(f"{domain.replace('.', '_')}.json", "w") as f:
        json.dump(products, f, indent=2, ensure_ascii=False)
    print(f"  ✓ 已保存 {len(products)} 个产品")
    time.sleep(3)  # 礼貌性间隔


跑起来之后，每家竞品的完整产品目录就会分别保存成 JSON 文件，后续可以接入数据库或者用 pandas 分析。

---

## 价格追踪：把 Shopify 抓取变成持续监控系统

单次抓取是快照，真正有价值的是**持续追踪价格变化**。比如竞品什么时候打折了、某个产品是否已经下架、哪些 SKU 出现了涨价——这些信息需要定期跑脚本才能捕捉到。

一个简单的价格历史记录方案（使用 SQLite）：

python
import sqlite3
from datetime import datetime

def init_db():
    conn = sqlite3.connect("prices.db")
    conn.execute("""
        CREATE TABLE IF NOT EXISTS price_history (
            domain TEXT,
            sku TEXT,
            title TEXT,
            price REAL,
            available BOOLEAN,
            scraped_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    return conn

def record_prices(conn, domain, products):
    for p in products:
        for v in p["variants"]:
            conn.execute(
                "INSERT INTO price_history (domain, sku, title, price, available) VALUES (?, ?, ?, ?, ?)",
                (domain, v.get("sku", ""), p["title"], float(v["price"]), v["available"])
            )
    conn.commit()


配合定时任务（比如 crontab 或者 GitHub Actions），可以每天自动跑一次，积累一段时间的数据后，就能画出每个产品的价格走势图。

---

## 如果不想写代码：无代码方案

不是所有人都想写 Python 脚本。有几个选项可以让你不写一行代码就抓到 Shopify 数据：

**ScraperAPI DataPipeline**：ScraperAPI 内置的可视化数据管道工具，通过界面配置抓取任务、设置调度频率，然后把结果通过 Webhook 推送到你的 Google Sheets 或数据库。适合定期需要数据但不想维护代码的团队。

**Apify Shopify Actor**：预构建的 Shopify 产品抓取器，不需要写代码，但定制空间有限。

**No-code 工具（如 Thunderbit、Simplescraper）**：浏览器插件或在线工具，适合一次性的少量抓取。

如果你的需求是持续、大量的数据收集，写代码 + ScraperAPI 的组合仍然是性价比最高的方案。

👉 [点击注册 ScraperAPI，免费获取 5,000 次 API 调用额度](https://www.scraperapi.com/?fp_ref=coupons)

---

## ScraperAPI 套餐全览与价格对比

ScraperAPI 提供 7 天试用期，注册后可免费获得 5,000 个 API Credits，无需信用卡。以下是全部付费套餐的详细对比：

| 套餐名称 | 月付价格 | 年付价格（省10%） | API Credits | 并发线程 | 地理定向 | 分析历史 | Pay-as-you-go | 购买链接 |
|---|---|---|---|---|---|---|---|---|
| **Hobby** | $49/月 | $44.10/月 | 100,000 | 20 | 仅美国&欧盟 | 最近30天 | ✗ |  [立即开始](https://www.scraperapi.com/signup/?fp_ref=coupons) |
| **Startup** | $149/月 | $134.10/月 | 1,000,000 | 50 | 仅美国&欧盟 | 最近30天 | ✗ |  [立即开始](https://www.scraperapi.com/signup/?fp_ref=coupons) |
| **Business** | $299/月 | $269.10/月 | 3,000,000 | 100 | 全球国家级 | 无限制 | ✗ |  [立即开始](https://www.scraperapi.com/signup/?fp_ref=coupons) |
| **Scaling** ⭐ 最受欢迎 | $475/月 | $427.50/月 | 5,000,000 | 200 | 全球国家级 | 无限制 | ✅ |  [立即开始](https://www.scraperapi.com/signup/?fp_ref=coupons) |
| **Professional** | $975/月 | $877.50/月 | 10,500,000 | 300 | 全球国家级 | 无限制 | ✅ |  [立即开始](https://www.scraperapi.com/signup/?fp_ref=coupons) |
| **Advanced** | $1,975/月 | $1,777.50/月 | 21,500,000 | 500 | 全球国家级 | 无限制 | ✅ |  [立即开始](https://www.scraperapi.com/signup/?fp_ref=coupons) |
| **Enterprise** | 定制 | 定制 | 22,000,000+ | 500+ | 全球国家级 | 无限制 | ✅ |  [联系销售](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

所有套餐均包含：JS 渲染、Premium 代理、JSON 自动解析、代理池轮换、自定义 Header 支持、CAPTCHA 处理、自动重试、无限带宽，以及 99.9% 在线时间保证。

> **选套餐建议**：只是个人项目或测试，Hobby 够用；中小团队做竞品监控，Startup 或 Business 是主流选择；有大规模、持续性抓取需求的，Scaling 以上套餐的 Pay-as-you-go 功能会非常有用——超出额度不会直接中断，而是按固定单价继续计费。

---

## 常见问题解答

**抓取 Shopify 产品数据合法吗？**

从公开可访问的 Shopify `/products.json` 端点获取数据，通常被认为是合法行为，因为这些数据本来就是公开展示的。2024 年的 Meta v. Bright Data 案也确认了 TOS 限制仅适用于登录用户。但具体情况建议参考目标店铺的服务条款和所在地区的法律。

**如果 /products.json 返回 404 怎么办？**

说明该店铺可能启用了安全配置或 Cloudflare 保护。这时候需要使用带 JS 渲染功能的抓取工具（比如 ScraperAPI 加上 `render=true`），或者考虑直接抓取产品页面的 HTML。

**API Credits 怎么计算？**

ScraperAPI 中，一次标准页面请求消耗 1 个 Credit。Shopify 的 `/products.json` 属于标准请求，成本很低。如果你使用 JS 渲染或者抓取有 Cloudflare 保护的页面，每次请求会额外消耗 10 个 Credits。

**未用完的 Credits 可以滚入下月吗？**

不可以，Credits 在每个计费周期结束时重置。如果担心浪费，可以选择带 Pay-as-you-go 的套餐（Scaling 及以上），超出额度按单价计费，灵活性更高。

---

## 总结

scrape shopify products 这件事，技术层面并没有多复杂。Shopify 的 `/products.json` 端点本身就是个开放的礼物，几行 Python 代码就能把竞品数据拿到手。

真正的挑战是规模化之后怎么不被封、怎么持续监控、怎么处理有 JavaScript 渲染需求的页面——这些地方，一个好用的 API 工具会省掉你大量的时间和精力。

👉 [免费注册 ScraperAPI，5,000 次 API 调用免费体验](https://www.scraperapi.com/?fp_ref=coupons)
