# Python Google地图抓取教程：用ScraperAPI绕过反爬、稳定采集商家数据的完整方案

## 为什么我开始折腾Google地图抓取

去年接了个本地商家数据分析的项目，需要批量拉取某个城市所有餐饮店的名称、评分、地址和电话。听起来简单，实际动手才发现Google地图是我碰过反爬最狠的目标之一——请求频率稍微高一点就弹验证码，换IP池也撑不过几百条就被封。折腾了两周各种免费代理方案后，我决定试ScraperAPI，主要是看中它声称能自动处理验证码和IP轮换。跑了三个月，把踩过的坑和能跑通的代码整理成这篇。

## Google地图反爬机制到底难在哪

Google地图的数据并不是静态HTML，而是通过内部API动态加载的。你在浏览器里看到的搜索结果，背后走的是一套protobuf编码的请求，直接用requests库去抓只会拿到一堆空壳。

几个核心难点：

- 请求频率限制极其敏感，同一IP连续请求十几次就可能触发人机验证

- 页面内容依赖JavaScript渲染，传统爬虫拿不到完整数据

- Google会根据请求头、Cookie、TLS指纹等多维度判断是否为机器人

- 地理位置影响搜索结果，需要精确控制出口IP的地区

我之前用Selenium配合免费代理池跑，成功率大概只有30%左右，而且速度慢得让人崩溃。

## ScraperAPI如何解决这些问题

ScraperAPI本质上是一个智能代理网关——你把目标URL丢给它的API端点，它帮你处理IP轮换、请求头伪装、验证码识别和JavaScript渲染，返回给你干净的HTML。

对Google地图抓取来说，几个关键能力：

- 自动管理超过4000万个IP的代理池，覆盖全球50多个国家和地区

- 内置JavaScript渲染引擎，能拿到动态加载后的完整页面

- 针对Google系产品做了专门优化，成功率比通用代理高很多

- 支持地理定位参数，可以指定从哪个国家/地区发起请求

我自己跑下来，针对Google地图搜索结果页的成功率稳定在90%以上，偶尔失败重试一次基本就过了。

## 环境准备与基础配置

先把依赖装好：

```bash

pip install requests beautifulsoup4 lxml

```

注册ScraperAPI账号后会拿到一个API Key。免费套餐给5000次请求额度，足够你跑通整个流程再决定要不要付费。

```python

import requests

from bs4 import BeautifulSoup

import time

import json

SCRAPER_API_KEY = "你的API_KEY"

BASE_URL = "http://api.scraperapi.com"

```

## 抓取Google地图搜索结果的完整代码

### 方法一：通过Google Maps搜索URL抓取

这是最直接的方式——构造Google地图的搜索URL，让ScraperAPI渲染后返回HTML，再从中解析商家信息。

```python

import requests

from bs4 import BeautifulSoup

import urllib.parse

import time

import json

SCRAPER_API_KEY = "你的API_KEY"

def build_google_maps_search_url(query, location="):

"""构造Google地图搜索URL"""

search_query = f"{query} {location}".strip()

encoded_query = urllib.parse.quote(search_query)

return f"https://www.google.com/maps/search/{encoded_query}/"

def scrape_with_scraperapi(target_url, render_js=True, country="us"):

"""

通过ScraperAPI发起请求

render_js: 是否启用JavaScript渲染（Google地图必须开启）

country: 出口IP所在国家代码

"""

params = {

"api_key": SCRAPER_API_KEY,

"url": target_url,

"render": str(render_js).lower(),

"country_code": country,

"wait_for_selector": "div[role='feed']", # 等待搜索结果容器加载

"timeout": 6000

}

response = requests.get(

"http://api.scraperapi.com",

params=params,

timeout=90

)

if response.status_code == 200:

return response.text

else:

print(f"请求失败，状态码: {response.status_code}")

return None

def parse_map_results(html_content):

"""从渲染后的HTML中解析商家信息"""

soup = BeautifulSoup(html_content, "lxml")

businesses = []

# Google地图搜索结果通常在带有特定aria-label的链接中

result_links = soup.find_all("a", attrs={"aria-label": True})

for link in result_links:

href = link.get("href", "")

if "/maps/place/" not in href:

continue

business = {

"name": link.get("aria-label", "").strip(),

"url": href

}

# 尝试提取评分

rating_span = link.find_next("span", attrs={"aria-label": True})

if rating_span:

label = rating_span.get("aria-label", "")

if "star" in label.lower() or "颗星" in label:

business["rating"] = label

businesses.append(business)

return businesses

def scrape_google_maps(query, location="", country="us", max_retries=3):

"""主函数：抓取Google地图搜索结果"""

url = build_google_maps_search_url(query, location)

print(f"目标URL: {url}")

for attempt in range(max_retries):

print(f"第 {attempt + 1} 次尝试...")

html = scrape_with_scraperapi(url, render_js=True, country=country)

if html:

results = parse_map_results(html)

if results:

print(f"成功抓取 {len(results)} 条结果")

return results

else:

print("页面已返回但未解析到结果，可能需要调整解析逻辑")

time.sleep(3) # 失败后等待几秒再重试

print("达到最大重试次数，抓取失败")

return []

# 使用示例

if __name__ == "__main__":

results = scrape_google_maps(

query="餐厅",

location="上海市静安区",

country="us"

)

for biz in results:

print(json.dumps(biz, ensure_ascii=False, indent=2))

```

### 方法二：抓取单个商家详情页

拿到搜索结果里的商家URL后，可以进一步抓取详情页获取完整信息：

```python

def scrape_business_detail(place_url, country="us"):

"""抓取单个Google地图商家详情页"""

params = {

"api_key": SCRAPER_API_KEY,

"url": place_url,

"render": "true",

"country_code": country,

"timeout": 60000

}

response = requests.get(

"http://api.scraperapi.com",

params=params,

timeout=90

)

if response.status_code != 200:

return None

soup = BeautifulSoup(response.text, "lxml")

detail = {}

# 商家名称

name_tag = soup.find("h1")

if name_tag:

detail["name"] = name_tag.get_text(strip=True)

# 地址、电话等信息通常在带有data-item-id属性的按钮中

info_buttons = soup.find_all("button", attrs={"data-item-id": True})

for btn in info_buttons:

item_id = btn.get("data-item-id", "")

text = btn.get_text(strip=True)

if "address" in item_id:

detail["address"] = text

elif "phone" in item_id:

detail["phone"] = text

elif "authority" in item_id:

detail["website"] = text

# 评分和评论数

rating_section = soup.find("div", attrs={"role": "img"})

if rating_section:

label = rating_section.get("aria-label", "")

detail["rating_info"] = label

return detail

def batch_scrape_details(business_list, delay=2):

"""批量抓取商家详情"""

detailed_results = []

for i, biz in enumerate(business_list):

print(f"正在抓取第 {i+1}/{len(business_list)} 个: {biz['name']}")

if "url" in biz and biz["url"].startswith("http"):

detail = scrape_business_detail(biz["url"])

if detail:

detailed_results.append(detail)

else:

print(f" 抓取失败: {biz['name']}")

time.sleep(delay) # 控制请求间隔，避免触发限制

return detailed_results

```

### 方法三：使用ScraperAPI的Google Maps专用端点

ScraperAPI提供了结构化数据端点，直接返回JSON格式的搜索结果，省去HTML解析的麻烦：

```python

def scrape_maps_structured(query, location_lat=31.23, location_lng=121.47, zoom=14):

"""

使用ScraperAPI的结构化搜索端点

直接返回JSON格式数据，无需手动解析HTML

"""

params = {

"api_key": SCRAPER_API_KEY,

"engine": "google_maps",

"q": query,

"ll": f"@{location_lat},{location_lng},{zoom}z",

"type": "search

}

response = requests.get(

"https://api.scraperapi.com/structured/google/maps/search",

params=params,

timeout=60

)

if response.status_code == 200:

data = response.json()

return data.get("local_results", [])

else:

print(f"结构化端点请求失败: {response.status_code}")

return []

# 使用示例

results = scrape_maps_structured(

query="coffee shop",

location_lat=31.2304,

location_lng=121.4737,

zoom=14

)

for place in results:

print(f"名称: {place.get('title')}")

print(f"评分: {place.get('rating')}")

print(f"地址: {place.get('address')}")

print(f"电话: {place.get('phone')}")

print("---")

```

这个方法是我目前用得最多的，返回的数据结构清晰，不用担心Google改版导致解析失败。

## 处理翻页与大批量采集

Google地图的搜索结果默认一次加载约20条，需要模拟滚动加载更多。用ScraperAPI配合分页策略：

```python

def scrape_maps_with_pagination(query, location, total_needed=100, country="us"):

"""

分页抓取Google地图结果

通过修改搜索参数实现翻页效果

"""

all_results = []

start = 0

page_size = 20

while len(all_results) < total_needed:

# 使用结构化端点的分页参数

params = {

"api_key": SCRAPER_API_KEY,

"engine": "google_maps",

"q": f"{query} {location}",

"type": "search",

"start": start

}

response = requests.get(

"https://api.scraperapi.com/structured/google/maps/search",

params=params,

timeout=60

)

if response.status_code != 200:

print(f"第 {start // page_size + 1} 页请求失败")

break

data = response.json()

results = data.get("local_results", [])

if not results:

print("没有更多结果了")

break

all_results.extend(results)

start += page_size

print(f"已采集 {len(all_results)} 条，继续下一页...")

time.sleep(2) # 页间隔

return all_results[:total_needed]

```

## 数据存储与导出

抓到的数据存成CSV或JSON方便后续分析：

```python

import csv

def save_to_csv(results, filename="google_maps_data.csv"):

"""将结果保存为CSV文件"""

if not results:

print("没有数据可保存")

return

# 收集所有可能的字段

all_keys = set()

for r in results:

all_keys.update(r.keys())

with open(filename, "w", newline="", encoding="utf-8-sig") as f:

writer = csv.DictWriter(f, fieldnames=sorted(all_keys))

writer.writeheader()

writer.writerows(results)

print(f"已保存 {len(results)} 条数据到 {filename}")

def save_to_json(results, filename="google_maps_data.json"):

"""将结果保存为JSON文件"""

with open(filename, "w", encoding="utf-8") as f:

json.dump(results, f, ensure_ascii=False, indent=2)

print(f"已保存 {len(results)} 条数据到 {filename}")

```

## 实际使用中的坑和解决办法

### 请求超时问题

Google地图页面加载本身就慢，加上ScraperAPI的渲染时间，超时是常见问题。我的做法是把timeout设到90秒，同时用`wait_for_selector`参数让ScraperAPI等到关键元素出现再返回：

```python

params = {

"api_key": SCRAPER_API_KEY,

"url": target_url,

"render": "true",

"wait_for_selector": "div[role='feed']",

"timeout": 6000 # ScraperAPI内部超时，单位毫秒

}

```

### 地理定位不准

如果你要抓特定城市的结果，光靠`country_code`参数不够精确。建议直接在搜索词里加上具体地名，或者用结构化端点的经纬度参数：

```python

# 方式一：搜索词里带地名

query = "火锅店 成都市武侯区"

# 方式二：用经纬度精确定位（结构化端点）

params["ll"] = "@30.5728,104.066814z" # 成都市中心坐标

```

### 请求额度管理

ScraperAPI按请求次数计费，开启JavaScript渲染的请求消耗额度是普通请求的10倍。我的省钱策略：

- 搜索结果页必须开渲染（没办法，不渲染拿不到数据）

- 如果只需要商家名称和链接，搜索结果页就够了，不用再抓详情页

- 用结构化端点比自己渲染+解析更划算，因为它一次请求就返回结构化数据

- 做好本地缓存，同一个URL不要重复请求

## ScraperAPI套餐对比

| 套餐名称 | 月请求量 | 并发数 | JavaScript渲染 | 地理定位 | 结构化数据端点 | 月费 | 链接 |
| --------- | ------ | ------------ | ------------ | --- | ------ | --- | --- |
| Free | 5,000次 | 1个 | 支持 | 支持 | 有限 | 免费 | [查看](url) |
| Hobby | 100,000次 | 5个 | 支持 | 支持 | 支持 | $49/月 | [查看](url) |
| Startup | 500,000次 | 10个 | 支持 | 支持 | 支持 | $149/月 | [查看](url) |
| Business | 3,000,000次 | 50个 | 支持 | 支持 | 支持 | $299/月 | [查看](url) |
| Enterprise | 自定义 | 自定义 | 支持 | 支持 | 支持 | 联系销售 | [查看](url) |

所有付费套餐都支持按年付费享受折扣，年付大概能省20%左右。如果你的项目只是偶尔跑一次，Hobby套餐10万次请求绰绰有余；如果是持续性的数据采集业务，Startup或Business更合适。

## 和其他方案的对比：为什么我最终选了ScraperAPI

在用ScraperAPI之前，我试过这几种方案：

**自建代理池 + Selenium**：成本低但维护成本高，免费代理质量参差不齐，成功率低于30%，而且Selenium跑起来慢得要命，一个小时能抓200条就不错了。

**Google Maps Platform官方API**：数据最准确，但Places API按请求计费，批量拉数据的成本远高于ScraperAPI。而且官方API对搜索结果数量有严格限制，一次最多返回60条。

**其他代理服务**：试过几家，对Google系产品的成功率普遍不如ScraperAPI。主要是ScraperAPI针对Google做了专门的指纹伪装和请求策略优化。

ScraperAPI的优势在于它把代理管理、验证码处理、浏览器渲染这些脏活累活全包了，我只需要关心业务逻辑和数据解析。我自己跑了三个月，平均成功率在92%左右，偶尔遇到失败重试一次基本就过了。

## 完整项目示例：采集某城市所有咖啡店数据

把上面的模块串起来，这是一个可以直接跑的完整脚本：

```python

"""

完整示例：采集指定城市的咖啡店数据并导出CSV

"""

import requests

import json

import csv

import time

from datetime import datetime

SCRAPER_API_KEY = "你的API_KEY"

def scrape_coffee_shops(city="上海", district="", max_results=200):

"""采集指定城市的咖啡店数据"""

query = f"咖啡店 {city}{district}".strip()

all_results = []

start = 0

print(f"开始采集: {query}")

print(f"目标数量: {max_results} 条")

print("-" * 50)

while len(all_results) < max_results:

params = {

"api_key": SCRAPER_API_KEY,

"engine": "google_maps",

"q": query,

"type": "search",

"start": start,

"hl": "zh-CN"

}

try:

response = requests.get(

"https://api.scraperapi.com/structured/google/maps/search",

params=params,

timeout=90

)

if response.status_code == 200:

data = response.json()

results = data.get("local_results", [])

if not results:

print(f"第 {start // 20 + 1} 页无更多结果，采集结束")

break

for r in results:

all_results.append({

"名称": r.get("title", ""),

"评分": r.get("rating", ""),

"评论数": r.get("reviews", "),

"地址": r.get("address", ""),

"电话": r.get("phone", ""),

"类型": r.get("type", "),

"营业状态": r.get("open_state", ""),

"采集时间": datetime.now().strftime("%Y-%m-%d %H:%M")

})

print(f"已采集 {len(all_results)} 条 (第 {start // 20 + 1} 页)")

start += 20

time.sleep(2)

elif response.status_code == 429:

print("触发频率限制，等待10秒后重试...")

time.sleep(10)

else:

print(f"请求失败: {response.status_code}")

break

except requests.exceptions.Timeout:

print("请求超时，等待5秒后重试...")

time.sleep(5)

# 导出数据

filename = f"coffee_shops_{city}_{datetime.now().strftime('%Y%m%d')}.csv"

if all_results:

keys = all_results[0].keys()

with open(filename, "w", newline="", encoding="utf-8-sig") as f:

writer = csv.DictWriter(f, fieldnames=keys)

writer.writeheader()

writer.writerows(all_results)

print(f"\n采集完成！共 {len(all_results)} 条数据，已保存到 {filename}")

return all_results

if __name__ == "__main__":

results = scrape_coffee_shops(city="上海", district="浦东新区", max_results=100)

```

## 常见问题

### ScraperAPI的免费额度够跑多少条Google地图数据？

免费套餐给5000次API请求。如果用结构化端点，每次请求返回约20条结果，理论上能拿到大约10万条商家基础信息（5000次 × 20条）。但如果用渲染模式抓取原始HTML，每次渲染请求消耗10个额度，实际能跑500次渲染请求。建议优先用结构化端点，性价比最高。

### 抓取Google地图数据合法吗？

这取决于你的用途和所在地区的法律。一般来说，抓取公开可见的商家信息（名称、地址、电话、评分）用于数据分析、市场调研属于灰色地带。但要注意不要违反Google的服务条款，不要抓取用户隐私数据，不要对Google服务器造成过大压力。ScraperAPI的请求频率控制和智能调度在一定程度上帮你避免了"暴力请求"的问题。

### 为什么我的解析代码突然拿不到数据了？

Google地图的前端结构经常改版，HTML的class名和DOM结构隔几周就可能变一次。这也是我推荐用ScraperAPI结构化端点的原因——它的团队会持续维护解析逻辑，你拿到的永远是干净的JSON数据，不用自己跟着Google的改版跑。如果你坚持自己解析HTML，做好定期检查和更新选择器的准备。

### ScraperAPI和Google Maps Platform官方API怎么选？

如果你的需求是在自己的应用里展示地图或调用地理编码服务，用官方API。如果你的需求是批量采集搜索结果、竞品分析、市场调研，ScraperAPI更合适。官方Places API单次搜索最多返回60条结果，而且按请求计费的价格对批量采集来说太贵了。ScraperAPI没有结果数量限制，成本也更可控。

### 被封IP了怎么办？

用ScraperAPI基本不用担心这个问题。它的代理池有超过4000万个IP，每次请求自动轮换，单个IP被封不影响后续请求。如果你发现成功率突然下降，通常是Google临时加强了某个地区的反爬策略，等几个小时或换个`country_code`参数试就行。我用了三个月，没有遇到过需要手动处理IP封禁的情况。

### 能抓取Google地图上的用户评论吗？

可以，但需要额外的请求。搜索结果页只包含评分和评论数量，具体评论内容需要进入商家详情页后再抓取评论区。每个商家的评论页是独立的请求，如果你需要大量评论数据，要注意额度消耗。建议先用搜索结果筛选出目标商家，再有针对性地抓取评论。

---

如果你的项目是中小规模的本地商家数据采集（几百到几千条），Hobby套餐的10万次请求完全够用，月费$49比自己维护代理池省心太多。大规模持续采集的话直接上Startup或Business。不确定的话先用免费额度跑一遍你的目标场景，确认成功率和数据质量再决定。

👉 [现在注册ScraperAPI免费拿5000次请求额度](https://www.scraperapi.com/?fp_ref=coupons)
