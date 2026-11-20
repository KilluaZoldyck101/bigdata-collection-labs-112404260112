# 实验三：QQ邮箱采集（Playwright版）

与[Selenium版](实验指导书-Selenium版.md)二选一，完成其中一版即可。**正文必做，附件选做。** 不混用两版代码。

| 阶段 | 课时 | 完成效果 |
|---|---|---|
| 1. 扫码登录 | 第1—2课时 | 未扫码时等待；扫码后进入收件箱 |
| 2. 采集和保存 | 第3—4课时 | 采集10封邮件，用pandas保存并核对CSV |
| 3. 滚动采集 | 第5—6课时 | 连续采集3批，按邮件ID去重 |
| 4. 详情正文 | 第7—8课时 | 逐封读取正文并返回；附件下载选做 |

## 使用方法

在 `workspace/lab03/` 中自己新建 `qqmail_lab03_playwright.ipynb`。按单元顺序添加代码，补全本阶段全部 `____` 后再运行。

本版使用异步API。`await` 表示等待这个浏览器操作完成。直接在Notebook单元中运行，不使用 `!python` 或 `%run` 执行这些片段。

四阶段填写[实验报告](实验报告-Playwright版.md)，最后统一提交一次GitHub。

## 第一阶段：扫码登录（第1—2课时）

### 目标与准备

**目标：** 打开QQ邮箱，扫码后输出“登录成功”，进入收件箱。

未安装visual的同学，先按[可视化安装说明](可视化环境安装说明.md)完成安装。每次上课运行 `start-course-visual.bat`，打开JupyterLab和浏览器画面。

`lab03` 与 `L07_L08_visual` 同级。学生自己编写代码，不领取完整采集程序。

### 单元1：准备浏览器

环境路径配置不要求背诵。补全启动、创建上下文和新页面的方法。

```python
import sys
from pathlib import Path
from uuid import uuid4
import pandas as pd
from playwright.async_api import async_playwright

PRIVATE = Path.cwd() / "private_data"
PRIVATE.mkdir(exist_ok=True)

# 使用visual安装时已有的显示服务配置。
sys.path.insert(0, str(Path.cwd().parent / "L07_L08_visual"))
import course_visual
course_visual.prepare()

p = await async_playwright().____()
browser = await p.chromium.____(headless=____, args=["--disable-dev-shm-usage"])
context = await browser.____(
    viewport={"width": 1000, "height": 620}, accept_downloads=True
)
page = await context.____()
page.set_default_timeout(20000)  # 配置默认超时，单位毫秒，不需要await。
```

提示：`start()` 启动工具，`launch()` 启动浏览器，`new_context()` 创建环境，`new_page()` 创建页面。需要显示窗口时，`headless=False`。

### 单元2：打开邮箱并等待扫码

```python
# ① 补打开网页的方法。
await page.____("https://mail.qq.com/", wait_until="domcontentloaded")
print("请在浏览器画面中扫码", flush=True)

# ② 人工扫码；自己查找左侧“收件箱”所在的区域。
inbox = page.____("____").get_by_text("收件箱", exact=True)

# ③ 最长等待10分钟。出现后立即继续。
await inbox.____(timeout=____)
print("登录成功", flush=True)
await inbox.____()
```

| 关键函数 | 用途 |
|---|---|
| `page.goto(url)` | 打开网页 |
| `page.locator("选择器")` | 按CSS定位 |
| `get_by_text("文字", exact=True)` | 按完整文字定位 |
| `locator.wait_for()` | 等待元素出现 |
| `locator.click()` | 点击，自动等待可操作条件 |

这里600000毫秒等于10分钟。定位器先描述位置；点击、取文字等实际操作才需要 `await`。

**完成证据：** 登录代码、成功输出和收件箱截图。框出定位、等待、点击，并遮挡私人信息。

## 第二阶段：采集10封邮件并保存（第3—4课时）

### 本阶段目标

采集发件人、邮箱、主题、时间。用pandas保存并回读10条记录、4列。

### 单元3：自己找选择器

只给两条线索，其余在开发者工具中查找。

| 内容 | 线索 |
|---|---|
| 发件人姓名 | `.cmp-account-nick` |
| 广告聚合入口标记 | `.gg-unread-count`，应跳过 |

```python
ROW_CSS = "____"
SENDER_CSS = ".cmp-account-nick"
AD_CSS = ".gg-unread-count"
SUBJECT_CSS = "____"
TIME_CSS = "____"
HOVER_CSS = "____"
EMAIL_CSS = "____"
```

分别找一条邮件、主题、时间、发件人悬停区域和卡片中的邮箱。卡片在整个页面中查找。

### 单元4：定义列表解析函数

```python
async def get_ids():
    await page.locator(ROW_CSS).first.wait_for()
    # 一次读取整批ID，避免列表更新时逐个保存旧元素。
    ids = await page.locator(ROW_CSS).evaluate_all(
        "rows => rows.map(row => row.getAttribute('data-mailid'))"
    )
    if not ids or not all(ids):
        raise ValueError("没有取得有效邮件ID，请检查页面")
    return ids


def get_row(mail_id):
    return page.locator(f'{ROW_CSS}[data-mailid="{mail_id}"]')


async def read_list_mail(mail_id):
    row = get_row(mail_id)
    if await row.locator(AD_CSS).____():
        return None
    sender = await row.locator(SENDER_CSS).____()
    subject = await row.locator(SUBJECT_CSS).____()
    time_text = await row.locator(TIME_CSS).____()

    # 先移开鼠标，等待上一张卡片消失。
    await row.locator(TIME_CSS).____()
    await page.locator(EMAIL_CSS).____(state="hidden")
    # 先把发件人滚入可见区域。滚动时页面会暂时停止响应悬停。
    await row.locator(HOVER_CSS).____()
    await page.locator(".mail-list-page .ui-float-scroll-body-scrolling").____(state="hidden")
    # 滚动结束后，再悬停本封发件人，等待卡片显示。
    await row.locator(HOVER_CSS).____()
    await page.locator(EMAIL_CSS).____(state="visible")
    email = await page.locator(EMAIL_CSS).____()
    return {"邮件ID": mail_id, "发件人": sender,
            "邮箱": email, "主题": subject, "时间": time_text}
```

| 关键函数 | 用途 |
|---|---|
| `locator.count()` | 统计匹配数，0表示没找到 |
| `locator.inner_text()` | 读取显示文字 |
| `locator.get_attribute(...)` | 读取属性 |
| `locator.hover()` | 鼠标悬停 |
| `locator.scroll_into_view_if_needed()` | 必要时滚动，让元素可见 |
| `locator.wait_for(state="hidden")` | 等待消失 |
| `locator.wait_for(state="visible")` | 等待显示 |

`evaluate_all()` 对匹配到的整批元素执行一次JavaScript。这里用于取得ID，不保存旧元素对象。

### 单元5：采集10封

```python
basic_records = []
for mail_id in await get_ids():
    record = await read_list_mail(mail_id)
    if record is None:
        continue
    basic_records.____(record)
    if len(basic_records) == ____:
        break
print("成功采集：", len(basic_records))
```

### 单元6：用pandas保存和回读

这两个函数会继续用于后面的阶段。pandas函数直接调用，不加 `await`。

```python
def save_records(records, filename, columns):
    df = pd.____(records, columns=columns)
    path = PRIVATE / filename
    df.____(path, index=____, encoding="utf-8-sig")
    return path


def read_records(path):
    return pd.____(path, dtype=str, keep_default_na=False)


BASIC_COLUMNS = ["发件人", "邮箱", "主题", "时间"]
basic_path = save_records(basic_records, "mail_basic.csv", BASIC_COLUMNS)
basic_df = read_records(basic_path)
print("CSV记录数：", len(basic_df))
basic_df.head(10)
```

提示：`DataFrame()` 建表，`to_csv()` 保存，`read_csv()` 回读。`index=False` 不多写一列行号，`utf-8-sig` 便于Excel读取。

**完成证据：** 字段提取与pandas代码截图；10条记录的回读结果；第一条、中间一条、最后一条与网页一致的核对截图。

## 第三阶段：滚动采集与ID去重（第5—6课时）

### 本阶段目标

连续采集3批，记录数等于唯一ID数。重复采集已有ID时，新增为0。

### 1. 认识滚动接口

`locator.evaluate()` 是Playwright接口，引号内是浏览器执行的JavaScript。

| 属性 | 含义 |
|---|---|
| `scrollTop` | 当前滚动位置，可读取和修改 |
| `clientHeight` | 一屏可见高度 |
| `scrollHeight` | 内容总高度 |

`await box.evaluate("box => box.scrollTop")` 将位置返回Python。

写入时用 `await box.evaluate("(box, value) => box.scrollTop = value", 目标位置)`。`box` 是定位到的元素，`value` 是Python传入的第二个参数。

下一屏通常是 `top + height`，末尾最大位置是 `scrollHeight - clientHeight`。先记本批位置，再采集，最后滚动。

### 单元7：补全解析和批次循环

```python
SCROLL_CSS = "____"


def get_box():
    return page.locator(SCROLL_CSS)


async def parse_batch(mail_ids, seen_ids, include_details=False, max_mails=0):
    result = []
    for mail_id in mail_ids:
        if max_mails and len(seen_ids) >= max_mails:
            break
        if ____:
            continue
        record = await read_list_mail(mail_id)
        if record is None:
            continue
        if include_details:
            # 第四阶段定义正文函数；本阶段False，不执行这里。
            record.update(await read_detail(mail_id))
        result.____(record)
        seen_ids.____(mail_id)
    return result


async def collect_batches(include_details=False, max_mails=0):
    await inbox.click()
    await page.locator(ROW_CSS).first.wait_for()
    await get_box().evaluate("box => box.scrollTop = 0")
    records, seen_ids = [], set()
    for batch in range(____):
        top = await get_box().evaluate("box => box.scrollTop")
        result = await parse_batch(await get_ids(), seen_ids, include_details, max_mails)
        records.extend(result)
        print(f"第{batch + 1}批，新增{len(result)}，累计{len(seen_ids)}")
        if batch == 2 or (max_mails and len(seen_ids) >= max_mails):
            break
        box = get_box()
        height = await box.evaluate("box => box.clientHeight")
        full_height = await box.evaluate("box => box.scrollHeight")
        next_top = min(____, full_height - height)
        if next_top <= top:
            print("已到列表末尾")
            break
        old_ids = await get_ids()
        await box.evaluate("(box, value) => box.scrollTop = value", next_top)
        # 在浏览器中比较新旧ID，改变后才继续。
        await page.____("""data => {
            const ids = Array.from(document.querySelectorAll(data.selector),
                                   node => node.getAttribute('data-mailid'));
            return ids.length > 0 && JSON.stringify(ids) !== JSON.stringify(data.ids);
        }""", arg={"selector": ROW_CSS, "ids": old_ids})
    return records, seen_ids
```

提示：`in` 判断ID是否存在，`append()` 加记录，`set.add()` 加ID，`wait_for_function()` 等待JavaScript条件成立。3批共用一个去重集合。

### 单元8：执行并保存3批

```python
PAGE_COLUMNS = ["邮件ID"] + BASIC_COLUMNS
page_records, seen_ids = await ____()
page_path = ____(page_records, "mail_pages.csv", PAGE_COLUMNS)
page_df = ____(page_path)
print("CSV记录数：", len(page_df))
print("唯一ID数：", page_df["邮件ID"].____())
print("重复ID数：", page_df["邮件ID"].duplicated().sum())
```

提示：调用已定义的采集、保存、回读函数；`nunique()` 统计唯一值。批次数量固定为3，条数以实际结果为准。

### 单元9：重复采集验证

```python
replay_ids = [r["邮件ID"] for r in page_records[:5]]
assert replay_ids, "先成功采集一些邮件，再做重复验证"
before = page_path.read_bytes()
replayed = await ____(replay_ids, seen_ids)
print("重复采集新增：", len(replayed))
____(page_records + replayed, "mail_pages.csv", PAGE_COLUMNS)
print("CSV内容未变：", before == page_path.read_bytes())
```

再次使用同一个去重集合，不是重启程序后重新覆盖文件。

**完成证据：** 滚动、等待和去重代码；3批日志；CSV记录数、唯一ID数；重复新增0条的结果截图。

## 第四阶段：详情正文（第7—8课时）

### 本阶段目标

打开详情、读取正文、返回列表，再继续下一封。保留前三阶段代码，追加下面的单元。

### 单元10：补全正文函数

自己查找正文容器和“返回”控件。`BACK_TEXT` 填控件的完整文字。

```python
BODY_CSS = "____"
BACK_TEXT = "____"
DOWNLOAD_ATTACHMENTS = False


async def read_detail(mail_id):
    top = await get_box().evaluate("box => box.scrollTop")
    row = get_row(mail_id)
    await row.locator(SUBJECT_CSS).____()
    try:
        body_node = page.locator(BODY_CSS)
        await body_node.____(state="____")
        body = (await body_node.____()).strip()
        files = ""
        if DOWNLOAD_ATTACHMENTS:
            files = await download_attachments(mail_id)
        return {"正文": body, "附件文件": files}
    finally:
        await page.get_by_text(BACK_TEXT, exact=True).____()
        await get_box().wait_for()
        await get_box().evaluate("(box, value) => box.scrollTop = value", ____)
        await get_row(mail_id).wait_for(state="attached")
```

提示：`click()` 点击；`wait_for(state="attached")` 等待节点出现在DOM；`inner_text()` 读取文字。返回后恢复之前的 `top`。

### 单元11：先试3封

```python
DETAIL_COLUMNS = PAGE_COLUMNS + ["正文", "附件文件"]
trial_records, _ = await ____(include_details=____, max_mails=____)
trial_path = ____(trial_records, "mail_details_trial.csv", DETAIL_COLUMNS)
trial_df = ____(trial_path)
print(trial_df[["邮件ID", "正文"]].assign(正文长度=trial_df["正文"].str.____()))
```

思考：本次调用哪个函数？是否采正文？限几封？核对3封正文和返回操作，再继续。

### 单元12：正式采集3批正文

```python
detail_records, _ = await ____(include_details=____)
detail_path = ____(detail_records, "mail_details.csv", DETAIL_COLUMNS)
detail_df = ____(detail_path)
print("正文CSV记录数：", len(detail_df))
print("重复ID数：", detail_df["邮件ID"].____().sum())
```

正文CSV共7列：邮件ID、发件人、邮箱、主题、时间、正文、附件文件。未下载时附件列为空。正文含换行时，用pandas统计记录数，不数文本行。纯图片邮件如实注明，不要求OCR。

### 单元13：选做附件下载

不选做就跳过本单元，保持开关关闭。选做时补完框架后，只试3封，不覆盖正式正文CSV。

```python
ATTACH_CSS = "____"
DOWNLOAD_TEXT = "____"


async def download_attachments(mail_id):
    cards = page.locator(ATTACH_CSS)
    if await cards.count() == 0:
        return ""
    # 每次建新目录，避免覆盖上次的附件。
    folder = PRIVATE / "attachments" / mail_id / uuid4().hex[:8]
    folder.mkdir(parents=True, exist_ok=False)
    files = []
    for number, card in enumerate(await cards.all(), 1):
        # 先监听下载，再点击。保存方法会等待下载完成。
        async with page.____(timeout=90000) as event:
            await card.get_by_text(DOWNLOAD_TEXT, exact=True).____()
        download = await event.value
        name = Path(download.suggested_filename.replace("\\", "/")).name
        target = folder / f"{number:02d}_{name}"
        await download.____(target)
        files.append(str(target.relative_to(PRIVATE)))
    return "\n".join(files)


DOWNLOAD_ATTACHMENTS = True
attachment_records, _ = await collect_batches(include_details=True, max_mails=3)
attachment_path = save_records(attachment_records, "mail_attachments_trial.csv", DETAIL_COLUMNS)
print(read_records(attachment_path)[["邮件ID", "附件文件"]])
DOWNLOAD_ATTACHMENTS = False
```

提示：`expect_download()` 监听下载；`suggested_filename` 取得文件名；`save_as()` 保存文件。选择无敏感练习附件，不要求发票。

**完成证据：** 正文函数、3封核对样本、正文长度、正式CSV条数和返回后继续处理的截图。选做时附文件名、大小。

## 最后统一提交

提交自己补全的 `qqmail_lab03_playwright.ipynb` 和 `实验报告-Playwright版.md`。三份正式原始CSV保留在本地 `private_data/`；三份脱敏CSV放在 `data/`，文件名加 `_public`。截图放在 `images/`。

四阶段完成后，一次上传整理好的 `lab03/` 到原GitHub仓库，学习通仍交原仓库链接。不要上传私人原文、附件、Cookie、登录态或密码。

下课前保存Notebook，再执行 `await browser.close()`、`await p.stop()`，最后运行 `stop-course-visual.bat`。下次课按顺序执行前面已补全的单元。
