---
layout: post
title:  "项目实训写爬虫时的一点感悟 —— 反爬与异步页面爬取"
date:   2019-07-17 1:34:00 +0700
categories: zijin update
---

Python爬虫的逻辑很简单，主要分成四个步骤：页面信息获取、基于html元素的内容抽取、条目封装、插入数据库。在这个过程中，我们主要遇到了两个挑战：服务器的反爬虫机制与异步网页解析。

反爬虫机制的设立主要出于对服务器稳定性的考量以及对自身数据资源的保护。该机制会通过检测请求频率、请求来源和请求内容等方式判断一个或一组请求是来源于浏览器还是爬虫脚本。当爬虫脚本暴露时，服务器一般会响应403错误阻止爬虫访问资源，对于一些行为“恶劣”的爬虫也可能直接禁止其来源IP。此外，浏览器还采取了一些事前预防措施，如要求请求携带Cookie或token（基于用户权限）、验证码等。

在本系统开发过程中，笔者所负责爬取的网站只使用了最基础的来源检测和频率检测，且没有在发现脚本后长时间禁止来源IP。为此，笔者使用的解决方案是在请求头部标记来源为浏览器加以伪装，并使用三个代理服务器进行轮换爬取，最大程度避免被禁用。在被首次禁用之后，笔者使用脚本大致估算了解禁时间，并在断点续爬后设置休眠时间以确保脚本不会因为服务器响应错误代码而终止运行。

同组同学在爬取百度指数和豆瓣网评分的过程中则遇到了Cookie验证和长时间IP禁用的问题。他们提供的解决方案是：使用百度账密向登录接口发送伪装成浏览器的登录请求获取Cookie，再使用该Cookie请求数据。Cookie过期后，换用新的百度账密，重复上面的步骤。同时，为防止被长时间禁用IP，他们使用了更为强大的代理池以规避服务器的流量监控。这些措施较好地规避了网站的安全策略，但也牺牲了效率，为此花费了不少时间用于数据的爬取。对于使用验证码的网站，一般的策略是对验证码图像进行文本识别，效率和准确性均较低。因在本系统开发过程中没有遇到类似的情况，故此处略去不表。

另一个挑战是异步元素爬取。所谓异步，就是一种通讯方式，异步双方不需要共同的时钟，也就是接收方不知道发送方什么时候发送，所以在发送的信息中就要有提示接收方开始接收的信息，如开始位，同时在结束时有停止位。异步的另外一种含义是计算机多线程的异步处理。与同步处理相对，异步处理不用阻塞当前线程来等待处理完成，而是允许后续操作，直至其它线程将处理完成，并回调通知此线程。

异步技术在B/S体系软件开发中被广泛使用。随着互联网技术的发展，为了改善用户体验，不少网页不再局限于静态页面，而是首先加载一个静态的页面“框架”，然后再使用JavaScript的异步请求分别加载页面的部分组件。这种页面组织模式在无形中给传统爬虫工具出了一道难题。以Python的requests库为例，该库常用的`requests.get()`方法无法直接获取异步内容和网页动态信息，对于使用异步策略的数据元素束手无策。从某种程度上说，异步加载机制也可以称得上是一种“无心插柳”的反爬虫机制。

### 解决方案一：使用抓包工具获取API

所幸，有多种方法可以解决这一问题。第一种是在抓包工具中（或者直接在Chrome调试页面的“Network”一栏）检测同对应AJAX异步请求相关的API，然后直接调用该API获取数据。该策略直接省去了页面加载的过程，并且可以获得比爬虫更为纯净JSON数据或XML数据，被笔者用来爬取“游民众评”网站的游戏列表。示例代码如下：

```python
headers = { 'Accept': 'text/javascript, application/javascript, application/ecmascript, application/x-ecmascript, */*; q=0.01',
'Accept-Encoding': 'gzip, deflate',
'Accept-Language': 'zh-CN,zh;q=0.9,zh-TW;q=0.8,en;q=0.7,ja;q=0.6',
'Connection': 'keep-alive',
'Cookie': 'UM_distinctid=16b42c587c93a4-0fbe59f9c42c1e-e353165-1fa400-16b42c587ca906; Search=1; CNZZDATA1256195895=1142608531-1562507310-%7C1562814353; Hm_lvt_dcb5060fba0123ff56d253331f28db6a=1563343159,1563343182,1563343193,1563343203; Hm_lpvt_dcb5060fba0123ff56d253331f28db6a=1563343247',
'Host': 'ku.gamersky.com',
'Referer': 'http://ku.gamersky.com/sp/',
'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/75.0.3770.142 Safari/537.36',
'X-Requested-With': 'XMLHttpRequest'
} # 从浏览器抓包获得的请求头规范，用于伪装来源

r = requests.get("http://ku.gamersky.com/SearchGameLibAjax.aspx?jsondata=%7BrootNodeId%3A20039%2CpageIndex%3A2%2CpageSize%3A36%2Csort%3A%2700%27%7D&_=1563330124891", headers=headers) # 传入jsondata进行请求
r.encoding = "utf8" # 修改编码
ti = BeautifulSoup(r.text)
print(ti.prettify())
print(r.text)
```

而这一策略的缺陷也很明显，首先，非公开的API可能存在权限或身份认证问题，不能随意调用；其次，API参数可能较为模糊，且存在部分我们无法获取到的参数（如内部id）；最后，在部分B/S体系软件中，前后端并不是完全分离的（如使用PHP开发的网站），这意味着我们可能根本无法获取到对应API。

除去“游民众评”，笔者所爬取的另一个网站采用PHP+JS开发，前后端高度耦合，大部分页面元素直接使用PHP加载，少部分页面元素使用JS执行AJAX请求加载。检测某处需要爬取的异步加载元素，发现获取该元素数据的API需要传入一个内部id。考虑到待爬取网页均为PHP页面，访问这些网页并不需要id（而是通过PHP文件名），请求网页静态资源的过程中也无法获取该id，因此该API实用性较低，策略一在此处并不适用。

### 解决方案二：Selenium WebDriver + PhantomJS

第二种策略是使用“无头浏览器”。这一策略由Python的Selenium WebDriver库支持。

selenium 是一套完整的web应用程序测试系统，包含了测试的录制（selenium IDE）,编写及运行（Selenium Remote Control）和测试的并行处理（Selenium Grid）。Selenium的核心Selenium Core基于JsUnit，完全由JavaScript编写，因此可以用于任何支持JavaScript的浏览器上。selenium可以模拟真实浏览器，自动化测试工具，支持多种浏览器，爬虫中主要用来解决JavaScript渲染问题。

而所谓“无头浏览器”，就是使用WebDriver库通过加载浏览器引擎的方式模拟浏览器请求，从而读取网页内容。由于整个过程可以配置浏览器不显示界面以提高爬取效率，因此被成为“无头模式”。通过这种方式读取的网页内容“所得即所见”，从根本上解决了异步加载元素比静态元素“慢半拍”的问题。WebDriver支持Chrome、Firefox和Safari等主流浏览器引擎，也支持完全无头的纯净轻量级引擎PhantomJS。

PhantomJS执行爬取的部分代码示例如下：

```python
driver = webdriver.PhantomJS()  # 初始化引擎
driver.get(url(year))   # 访问url
time.sleep(5) # 等待一定时间确保页面加载完毕
gamelist = driver.find_elements_by_class_name("gamelist") # 根据class名查找对应元素的列表
# ... #
game_name = game.find_element_by_tag_name('p').text # 根据tag查找第一个匹配的元素内容
# ... #
game_id = driver.find_element_by_class_name('tit_CH').get_attribute('gameid') # 获取查询结果的attribute
# ... #
driver.find_element_by_class_name("nexe").click() # 模拟鼠标点按操作
```

相比传统爬虫库，WebDriver的优势很明显。一方面，它从根本上规避了异步加载的问题，可以获取可视网页中所有的元素，并且内置一套根据标签名、类名和id进行内容抽取的实用方法，十分简便易用；另一方面，由于WebDriver的工作原理是调用浏览器内核，因此可以使用它模拟自然人访问网页时的滚动、定位和点按操作。这一点使它比传统爬虫操作更具象化，可拓展性也更强。最为重要的是，无头浏览器模式很难被反爬虫机制拦截，因此可以适用于一些对爬虫不友好的站点。

WebDriver的缺点也是显而易见的。首先，它的效率及其低下。虽然我们可以在请求过程中设置“无头”避免部分无用进程启动，但调用外部引擎这一过程还是会耗费大量时间，在爬取大量页面的过程中，这一缺陷是致命的；此外，WebDriver对页面加载是否完成的判断是不甚准确的，有时会出现页面元素丢失的情况；最后，WebDriver代码移植性很差，需要本机配置正确的环境变量并安装对应引擎才能正常运行。考虑到笔者在这个网站上爬取的页面较少，并且对精准性要求不高，因此WebDriver的执行效果基本达到了预期。但在对大量元素进行精准抽取时，WebDriver毫无疑问是无法胜任的。













