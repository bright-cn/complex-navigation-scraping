# 使用 Selenium 和浏览器自动化抓取具有复杂导航的网站

[![Promo](https://github.com/bright-cn/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://www.bright.cn) 

本指南将讲解如何利用 Selenium 和浏览器自动化来抓取具有复杂导航模式（如动态分页、无限滚动以及“加载更多”按钮等）的网站。

- [什么是复杂导航？](#什么是复杂导航)
- [应对复杂导航网站的工具](#应对复杂导航网站的工具)
- [抓取常见的复杂导航模式](#抓取常见的复杂导航模式)
  - [动态分页](#动态分页)
  - [“加载更多”按钮](#加载更多按钮)
  - [无限滚动](#无限滚动)
- [总结](#总结)

## 什么是复杂导航？

在网页抓取中，所谓复杂导航，是指网站结构中，内容或页面不那么容易访问的情况。复杂导航通常涉及动态元素、异步数据加载或用户操作触发的事件等。这些特性可以提升用户体验，但同时也让数据提取变得更困难。以下是一些常见示例：

- **基于 JavaScript 渲染的导航**：使用 JavaScript 框架在浏览器中直接生成内容的网站。  
- **分页内容**：数据分布于多个分页中，且分页是通过 AJAX 动态加载的。  
- **无限滚动**：当用户向下滚动时，页面会动态加载更多内容，社交媒体动态流、基于 Discourse 的论坛以及新闻网站中十分常见。  
- **多级菜单**：网站的菜单层级嵌套较深，需要多次点击或悬停才能显示更深层级的内容，如电商平台的商品分类树。  
- **交互式地图界面**：网站在地图或图表中展示数据，信息会根据用户的拖动或缩放而动态加载。  
- **选项卡或手风琴式布局**：页面中有些内容隐藏在动态渲染的选项卡或可折叠手风琴中，而并非直接嵌入服务器返回的 HTML 中。  
- **动态的过滤和排序**：有些网站提供了复杂的过滤系统，应用多个过滤器后，会在不改变 URL 的情况下载入新的信息。

## 应对复杂导航网站的工具

上述许多复杂的交互都需要执行 JavaScript，仅浏览器才能完成。因此，你不能只依赖于简单的 [HTML 解析器](https://www.bright.cn/blog/web-data/best-html-parsers) 来抓取此类页面。你必须使用像 Selenium、Playwright 或 Puppeteer 这样的浏览器自动化工具。这些工具允许你以编程方式为浏览器指定在网页上的执行动作，模拟用户行为。

## 抓取常见的复杂导航模式

本指南涵盖三种常见的复杂导航模式：

- **动态分页**：通过 AJAX 动态加载分页数据  
- **“加载更多”按钮**：一种常见的基于 JavaScript 的导航方式  
- **无限滚动**：页面会在用户向下滚动时不断加载新的数据  

我们将以 Python 中的 Selenium 为例，但同样的逻辑也可应用于 Playwright、Puppeteer 或其他浏览器自动化工具。并且本指南默认你已熟悉 [使用 Selenium 进行网页抓取](https://www.bright.cn/blog/how-tos/using-selenium-for-web-scraping) 的基础知识。

### 动态分页

我们将使用 “[Oscar Winning Films: AJAX and Javascript](https://www.scrapethissite.com/pages/ajax-javascript/#2014)” 这个抓取练习场：

![目标页面。请注意，它的分页数据是动态加载的](https://github.com/bright-cn/complex-navigation-scraping/blob/main/Images/Dynamic-pagniation-example-1536x752.gif)

该网站会动态加载关于奥斯卡获奖电影的数据，并按年份分页。

若要有效地抓取和浏览此页面，你需要遵循以下操作：

1. 点击新年份按钮以触发数据加载（会出现一个加载指示器）。  
2. 等待加载指示器消失（表示数据已完全加载）。  
3. 确认页面上的数据表格已经正确渲染。  
4. 数据可用后再进行抓取。

以下是使用 Python 中 Selenium 实现的示例：

```python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.chrome.options import Options

# 配置 Chrome 选项为无头模式
options = Options()
options.add_argument("--headless")

# 创建 Chrome WebDriver 实例
driver = webdriver.Chrome(service=Service(), options=options)

# 访问目标页面
driver.get("https://www.scrapethissite.com/pages/ajax-javascript/")

# 点击“2012”分页按钮
element = driver.find_element(By.ID, "2012")
element.click()

# 等待加载指示器不再可见
WebDriverWait(driver, 10).until(
    lambda d: d.find_element(By.CSS_SELECTOR, "#loading").get_attribute("style") == "display: none;"
)

# 数据现在应该加载完成...

# 等待表格出现在页面上
WebDriverWait(driver, 10).until(
    EC.presence_of_element_located((By.CSS_SELECTOR, ".table"))
)

# 存储抓取到的数据
films = []

# 从表格中抓取数据
table_body = driver.find_element(By.CSS_SELECTOR, "#table-body")
rows = table_body.find_elements(By.CSS_SELECTOR, ".film")
for row in rows:
    title = row.find_element(By.CSS_SELECTOR, ".film-title").text
    nominations = row.find_element(By.CSS_SELECTOR, ".film-nominations").text
    awards = row.find_element(By.CSS_SELECTOR, ".film-awards").text
    best_picture_icon = row.find_element(By.CSS_SELECTOR, ".film-best-picture").find_elements(By.TAG_NAME, "i")
    best_picture = True if best_picture_icon else False

    # 将抓取到的数据存储到列表中
    films.append({
      "title": title,
      "nominations": nominations,
      "awards": awards,
      "best_picture": best_picture
    })

# 后续数据导出逻辑...

# 关闭浏览器
driver.quit()
```

下面对该代码的解析：

1.  代码启动了一个无头的 Chrome 浏览器。  
2.  脚本打开目标页面并点击“2012”分页按钮，以触发数据加载。  
3.  Selenium 使用 [`WebDriverWait()`](https://selenium-python.readthedocs.io/waits.html) 等待加载指示器消失。  
4.  加载指示器消失后，脚本继续等待表格元素出现。  
5.  当表格加载完毕后，脚本提取电影标题、提名次数、获奖次数，以及该电影是否获得最佳影片，最后将信息存储在字典列表中。  

最终结果会是这样：

```json
[
  {
    "title": "Argo",
    "nominations": "7",
    "awards": "3",
    "best_picture": true
  },
  // ...
  {
    "title": "Curfew",
    "nominations": "1",
    "awards": "1",
    "best_picture": false
  }
]
```

需要注意的是，对于此导航模式，往往不只存在一种应对方式，具体取决于页面的行为。以下是一些可选思路：

- 将 `WebDriverWait()` 与特定的等待条件相结合，以等待特定的 HTML 元素出现或消失。  
- 监测网站 AJAX 请求的网络流量，以判断何时加载新内容。这可能需要使用浏览器日志功能。  
- 找到分页触发的 API 请求，直接使用编程方式（，例如用 [`requests` 库](https://www.bright.cn/blog/web-data/python-requests-guide)）调用该 API 来获取数据。

### “加载更多”按钮

为演示基于 JavaScript 的复杂导航（需要用户操作），我们再来看一个“加载更多”按钮的示例。思路很简单：页面先显示一部分条目，点击该按钮后加载更多条目。

相关示例来自 [Scraping Course 的“加载更多”示例页面](https://www.scrapingcourse.com/button-click)：

![“加载更多”示例页面](https://github.com/bright-cn/complex-navigation-scraping/blob/main/Images/Clicking-on-the-load-more-button-1536x752.gif)

要抓取这种导航方式，一般步骤如下：

1.  找到“加载更多”按钮并点击它。  
2.  等待新元素加载到页面上。

下面是使用 Selenium 的完整代码：

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.support.ui import WebDriverWait

# 配置 Chrome 选项为无头模式
options = Options()
options.add_argument("--headless")

# 创建 Chrome WebDriver 实例
driver = webdriver.Chrome(options=options)

# 访问目标页面
driver.get("https://www.scrapingcourse.com/button-click")

# 获取页面初始商品数量
initial_product_count = len(driver.find_elements(By.CSS_SELECTOR, ".product-item"))

# 找到“加载更多”按钮并点击
load_more_button = driver.find_element(By.CSS_SELECTOR, "#load-more-btn")
load_more_button.click()

# 等待页面上的商品数量增加
WebDriverWait(driver, 10).until(lambda driver: len(driver.find_elements(By.CSS_SELECTOR, ".product-item")) > initial_product_count)

# 存储抓取的数据
products = []

# 抓取商品详情
product_elements = driver.find_elements(By.CSS_SELECTOR, ".product-item")
for product_element in product_elements:
    # 提取商品信息
    name = product_element.find_element(By.CSS_SELECTOR, ".product-name").text
    image = product_element.find_element(By.CSS_SELECTOR, ".product-image").get_attribute("src")
    price = product_element.find_element(By.CSS_SELECTOR, ".product-price").text
    url = product_element.find_element(By.CSS_SELECTOR, "a").get_attribute("href")

    # 将数据存储到列表中
    products.append({
        "name": name,
        "image": image,
        "price": price,
        "url": url
    })

# 后续数据导出逻辑...

# 关闭浏览器
driver.quit()
```

在此示例中，脚本先记录页面上初始的商品数量，然后点击“加载更多”按钮，再等待商品数量增加（表示成功加载了新条目）。

这种方法既高效又通用，因为它无需知道具体需要加载多少新元素。不过，也还有其它替代或额外的方法可以实现相同目标。

### 无限滚动

无限滚动是一种广泛用于社交媒体和电商平台的交互模式，可提升用户体验。本例中，我们会使用与上例相同的页面，但是变为了 [无限滚动](https://www.scrapingcourse.com/infinite-scrolling)：

![无限滚动示例](https://github.com/bright-cn/complex-navigation-scraping/blob/main/Images/Infinite-scrolling-example-1024x501.gif)

大多数浏览器自动化工具并不提供直接滚动页面的功能，Selenium 也不例外。你需要在页面中执行 JavaScript 脚本来实现滚动行为。

我们可以编写自定义的 JavaScript 脚本来向下滚动页面：

1.  规定滚动的次数，或者  
2.  直到页面不再出现新内容。  

> **注意**：  
> 每次滚动都会加载新数据，页面中的元素数量也会随之增加。

之后，再抓取新加载的内容。

下面是通过 Selenium 来实现无限滚动的示例代码：

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.support.ui import WebDriverWait

# 配置 Chrome 选项为无头模式
options = Options()
# options.add_argument("--headless")

# 创建 Chrome WebDriver 实例
driver = webdriver.Chrome(options=options)

# 打开带无限滚动的目标页面
driver.get("https://www.scrapingcourse.com/infinite-scrolling")

# 当前页面高度
scroll_height = driver.execute_script("return document.body.scrollHeight")
# 当前页面商品数量
product_count = len(driver.find_elements(By.CSS_SELECTOR, ".product-item"))

# 最大滚动次数
max_scrolls = 10
scroll_count = 1

# 限制滚动次数为 10
while scroll_count < max_scrolls:
    # 滚动到页面底部
    driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")

    # 等待页面上商品数量增加，表示新的内容已加载
    WebDriverWait(driver, 10).until(lambda driver: len(driver.find_elements(By.CSS_SELECTOR, ".product-item")) > product_count)

    # 更新商品数量
    product_count = len(driver.find_elements(By.CSS_SELECTOR, ".product-item"))

    # 获取新的页面高度
    new_scroll_height = driver.execute_script("return document.body.scrollHeight")

    # 如果滚动后页面高度没有变化，说明没有新内容
    if new_scroll_height == scroll_height:
        break

    # 更新页面高度并增加滚动次数
    scroll_height = new_scroll_height
    scroll_count += 1

# 在无限滚动后抓取商品详情
products = []
product_elements = driver.find_elements(By.CSS_SELECTOR, ".product-item")
for product_element in product_elements:
    # 提取商品信息
    name = product_element.find_element(By.CSS_SELECTOR, ".product-name").text
    image = product_element.find_element(By.CSS_SELECTOR, ".product-image").get_attribute("src")
    price = product_element.find_element(By.CSS_SELECTOR, ".product-price").text
    url = product_element.find_element(By.CSS_SELECTOR, "a").get_attribute("href")

    # 将数据存储到列表中
    products.append({
        "name": name,
        "image": image,
        "price": price,
        "url": url
    })

# 导出 CSV/JSON ...

# 关闭浏览器
driver.quit() 
```

以上脚本通过先获取当前页面高度和商品数量，然后在循环中限制了最多滚动 10 次。每次循环都：

1.  滚动到页面底部。  
2.  等待商品数量增加，确认新内容已加载。  
3.  通过比较页面高度来判断是否有更多内容可加载。  

若在一次滚动后页面高度未变化，脚本就会停止循环，说明再无可加载数据。

## 总结

当遇到复杂的导航模式时，网页抓取可能变得非常具有挑战性，而网站还会使用各种反爬机制来阻止自动脚本。像 Selenium 这样单纯的浏览器自动化工具无法绕过这些限制。

解决方案是使用如 [Scraping Browser](https://www.bright.cn/products/scraping-browser) 这类云端浏览器。它可与 Playwright、Puppeteer、Selenium 等工具结合使用，并为每个请求自动轮换 IP，以及应对浏览器指纹检测、重试、验证码（CAPTCHA）识别等。这样，在处理复杂网站导航时就不必再担心被封锁！
