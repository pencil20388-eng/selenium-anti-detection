# Selenium 被检测怎么办：绕过反爬虫检测的完整方案（Python）

用 Selenium 做自动化或爬虫，跑着跑着被网站识别成机器人、弹验证码、甚至封 IP，是几乎每个人都会遇到的问题。这篇讲清楚 Selenium 为什么会被检测，以及从免费到进阶的几种解决办法，附完整可运行的 Python 代码。

## 目录

- [Selenium 为什么会被检测](#selenium-为什么会被检测)
- [先测一下你的浏览器暴露了什么](#先测一下你的浏览器暴露了什么)
- [方案一：用 selenium-stealth 插件伪装](#方案一用-selenium-stealth-插件伪装)
- [方案一的局限](#方案一的局限)
- [方案二：用指纹浏览器提供真实环境](#方案二用指纹浏览器提供真实环境)
- [多账号场景：为什么插件方案会失效](#多账号场景为什么插件方案会失效)
- [总结](#总结)

## Selenium 为什么会被检测

网站识别 Selenium，靠的是几类特征。

第一类是自动化标志。Selenium 驱动的浏览器会暴露 `navigator.webdriver = true` 这样的特征，很多反爬系统第一步就是检查这个。

第二类是浏览器指纹。无头模式（headless）的 Chrome，它的 User-Agent、WebGL 渲染器信息、Canvas 指纹、字体、屏幕参数等，和真实用户的浏览器有明显差异。反爬系统把这些特征组合起来，就能判断出这是个自动化脚本。

第三类是行为和 IP。请求太规律、频率太高、IP 来自机房而非住宅，都会触发风控。

CAPTCHA（验证码）通常是这些检测的结果，而不是原因。当系统判定你可疑，才会弹验证码拦你。所以要少弹验证码，核心是让浏览器不被判定为机器人。

## 先测一下你的浏览器暴露了什么

在动手之前，先看看当前 Selenium 环境暴露了哪些特征。用 `bot.sannysoft.com` 这个页面，它会跑一系列检测，告诉你哪些项通过、哪些暴露了。

先装 Selenium：

```bash
pip install selenium
```

写个脚本访问检测页并截图：

```python
from selenium import webdriver

options = webdriver.ChromeOptions()
options.add_argument("--headless")

driver = webdriver.Chrome(options=options)
driver.get("https://bot.sannysoft.com/")

driver.save_screenshot("before.png")
driver.quit()
```

运行后打开 `before.png`，你会看到无头 Chrome 有好几项检测没通过（比如 webdriver 那一项会标红）。这意味着在有防护的网站上，你的脚本很可能被判定为机器人。

<img width="1418" height="1646" alt="image" src="https://github.com/user-attachments/assets/d9d01bb0-f713-481d-be50-042b960b9e71" />

## 方案一：用 selenium-stealth 插件伪装

最常见的免费办法是用 `selenium-stealth` 这个插件，它通过修改浏览器的一些属性，减少被识别为自动化的概率。

安装：

```bash
pip install selenium-stealth
```

配置：

```python
from selenium import webdriver
from selenium_stealth import stealth

options = webdriver.ChromeOptions()
options.add_argument("--headless")

driver = webdriver.Chrome(options=options)

stealth(
    driver,
    languages=["zh-CN", "zh"],
    vendor="Google Inc.",
    platform="Win32",
    webgl_vendor="Intel Inc.",
    renderer="Intel Iris OpenGL Engine",
    fix_hairline=True,
)

driver.get("https://bot.sannysoft.com/")
driver.save_screenshot("after.png")
driver.quit()
```

再打开 `after.png`，你会发现之前标红的几项现在通过了。对付基础的反爬检测，这一步通常够用。

<img width="1418" height="1646" alt="image" src="https://github.com/user-attachments/assets/b6b3ceb2-ce0b-4294-80f7-ee9b5030a8bb" />

## 方案一的局限

`selenium-stealth` 是"打补丁"式的伪装，它有几个绕不过去的问题。

第一，指纹伪装不彻底。它改的是浏览器暴露出来的一部分 JS 属性，但 Canvas 指纹、WebGL 的更深层特征、TLS 指纹这些，它覆盖不到或覆盖得不干净。面对 Cloudflare、DataDome 这类高级反爬，基础伪装很容易被识破。

第二，要频繁维护。反爬系统一直在更新检测手段，插件的伪装参数需要跟着改。今天能过，下个月可能就不行了。

第三，也是最容易被忽略的——**多账号场景下会互相关联**。如果你要用 Selenium 管理多个账号（比如多个店铺、多个社媒号），这些账号都跑在同一套浏览器环境、同一个指纹下，网站会通过指纹发现"这些账号是同一台设备在操作"，直接判定为关联账号，一封一串。插件伪装解决不了这个问题，因为它没法给每个账号一套真正独立的指纹。

## 方案二：用指纹浏览器提供真实环境

比"打补丁"更彻底的思路，是不再伪装，而是直接用一个真实的、指纹独立的浏览器环境。这就是指纹浏览器的作用。

指纹浏览器从内核层面提供完整、真实的浏览器指纹，而不是在 JS 层面打补丁。每个环境有独立的指纹、cookie，可以各自配置独立 IP。对反爬系统来说，它就是一个真实用户的浏览器，而不是一个"伪装过的自动化工具"。

关键是，它不影响你现有的代码。以 AdsPower 为例，它给每个浏览器环境（profile）暴露一个 CDP 端口，你可以用 Selenium 直接接管，原来的自动化逻辑基本不用改。

<img width="1348" height="729" alt="image" src="https://github.com/user-attachments/assets/0d9d0894-8c09-4499-b4b3-51f663b85279" />

先通过 AdsPower 的本地接口启动一个浏览器环境：

```python
import requests

# AdsPower 本地 API，profile_id 是你在 AdsPower 里创建的环境 ID
resp = requests.get(
    "http://local.adspower.net:50325/api/v1/browser/start",
    params={"user_id": "你的_profile_id"},
)
data = resp.json()["data"]

# 拿到 selenium 连接地址和 webdriver 路径
selenium_address = data["ws"]["selenium"]
webdriver_path = data["webdriver"]
```

然后用 Selenium 接管这个已经启动的、指纹独立的浏览器：

```python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options

options = Options()
options.add_experimental_option("debuggerAddress", selenium_address)

service = Service(executable_path=webdriver_path)
driver = webdriver.Chrome(service=service, options=options)

# 接下来就是你熟悉的 Selenium 代码
driver.get("https://bot.sannysoft.com/")
driver.save_screenshot("adspower.png")
```

用完关闭环境：

```python
requests.get(
    "http://local.adspower.net:50325/api/v1/browser/stop",
    params={"user_id": "你的_profile_id"},
)
```
<img width="2338" height="7715" alt="image" src="https://github.com/user-attachments/assets/3ddf55c5-76d3-4750-b462-4e39c841f884" />

因为这里的浏览器是一个真实的指纹环境，`bot.sannysoft.com` 上那些检测项会自然通过，不需要额外伪装。而且你的 Selenium 代码几乎原样保留，只是把"自己启动的 Chrome"换成了"接管 AdsPower 启动的环境"。

## 多账号场景：为什么插件方案会失效

如果你的需求是多账号（多店铺、多社媒号、多广告账户），指纹浏览器的价值会更明显。

在 AdsPower 里为每个账号创建一个独立环境，每个环境有独立的指纹和 IP，然后用代码批量启动、用 Selenium 分别接管：

```python
import requests
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

profile_ids = ["profile_1", "profile_2", "profile_3"]

for pid in profile_ids:
    resp = requests.get(
        "http://local.adspower.net:50325/api/v1/browser/start",
        params={"user_id": pid},
    )
    addr = resp.json()["data"]["ws"]["selenium"]

    options = Options()
    options.add_experimental_option("debuggerAddress", addr)
    driver = webdriver.Chrome(options=options)

    # 对每个账号执行你的自动化逻辑
    driver.get("https://example.com")
    # ...
```

这样每个账号在网站看来都是一台独立设备、一个独立用户，彼此之间不会因为指纹相同而被关联。这是插件伪装方案做不到的——插件能让一个浏览器看起来不像机器人，但没法让多个账号看起来像多个不同的人。

## 总结

Selenium 被检测的处理，分几个层次。

基础需求，用 `selenium-stealth` 打补丁伪装，成本低，对付简单反爬够用。

进阶需求，尤其是面对高级反爬、或者需要多账号隔离时，用指纹浏览器提供真实、独立的浏览器环境更稳，而且能通过 CDP 接口无缝接入现有的 Selenium 代码。

指纹浏览器可以参考 [AdsPower]([https://www.adspower.net/](https://www.adspower.net/share/githubx))，它支持通过本地 API 启动环境、用 Selenium/Playwright 接管，适合自动化和多账号场景。

想更深入理解反检测原理，可以看这两份资源：

- [为什么你的脚本一跑就被封](https://github.com/pencil20388-eng/why-bots-get-banned)
- [awesome-anti-detect](https://github.com/pencil20388-eng/awesome-anti-detect)

---

以上代码用于合规的自动化测试与多账号运营。请遵守目标网站的服务条款和当地法律法规。
