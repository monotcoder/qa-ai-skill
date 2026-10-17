# 平台 · Web

第 1 问答「Web」时读这个文件（没问过端的老项目，也按这个文件走）。
安卓 App 读 `平台-安卓.md`，微信小程序读 `平台-小程序.md`。

六件和平台有关的事：怎么连上、怎么看界面、怎么定位、怎么知道在哪一页、登录态、证据。
六步骨架本身不变。

## 一、怎么连上

默认用 Playwright。**项目里已经有测试工具就用它的，不要替用户换掉。**

### 要装东西：先查有没有，再问，用户同意了才装

要下载的东西，耗时全看网速，用户不知道在等什么就会以为卡死了。
所以**先查清楚缺什么，一次列给用户，说清每样是什么、为什么要、多大、大概多久，他点头才装**。
已经有的不装，也不问。

| 要什么 | 怎么查有没有 | 没有怎么办 |
| --- | --- | --- |
| Node.js | `node -v` 能出版本号 | **请用户自己装**，给官网链接 https://nodejs.org （选 LTS）。安装包会弹系统窗口，本来就得他点 |
| Playwright 本身 | 项目里有 `node_modules/@playwright/test` | 问过再跑 `npm i`：十几 MB，网好一两分钟 |
| 浏览器 | 电脑上有没有 Chrome、Edge，见下面 | 见下面，**默认不下载** |

问的时候照这个样子，一次问完，别装一样问一样：

> 要跑测试，还得装一样东西：
> - Playwright：测试工具，替你点网页用的。十几 MB，大概一两分钟
>
> 浏览器用你电脑上的 Chrome，不用另外下载。
> 可以装吗？

装的过程中**每隔一会儿报一句进度**（装到哪了、还剩什么），别长时间没动静。

`npm i` 卡住不动的（国内常见），停下来告诉用户，问他要不要换国内镜像再试：
这一次加 `--registry=https://registry.npmmirror.com`，不改用户全局的 npm 设置。

### 浏览器：先用电脑上已经有的，别上来就下载

`npx playwright install` 要下好几百 MB，网不好要等很久，国内还经常下不动。
而且 Playwright 每升一个版本要的浏览器就不一样，以前下过的也用不上。
**大多数人电脑上已经有 Chrome 或 Edge，直接用它。**

1. 先查有没有（只看文件在不在，不打开浏览器）：
   - Mac：`/Applications/Google Chrome.app`、`/Applications/Microsoft Edge.app`
   - Windows：`Program Files` 或 `Program Files (x86)` 下的 `Google\Chrome\Application\chrome.exe`、
     `Microsoft\Edge\Application\msedge.exe`，还有 `%LOCALAPPDATA%\Google\Chrome\Application\chrome.exe`。
     Windows 10 以后都自带 Edge
   - Linux：`which google-chrome microsoft-edge`
2. 有 Chrome 用 Chrome，没有就用 Edge。在 `playwright.config.ts` 的 `use` 里写
   `channel: 'chrome'`（Edge 写 `'msedge'`），登录命令也带上 `--channel=chrome`。
   **这样不用下载任何浏览器**。测试用的是单独的一份空白资料，不碰用户平时浏览器里的书签和登录
3. 两个都没有：**请用户自己装一个**，给他官网链接，装好告诉你一声。
   打不开 Chrome 官网（国内常见）就装 Edge：https://www.microsoft.com/zh-cn/edge 。
   用户自己点安装比 AI 在后台下载快，也看得见进度
4. 用户明确说「你帮我下」，才跑 `npx playwright install chromium`。
   跑之前告诉他：要下几百 MB，可能要十几分钟，下不动就回到第 3 条

用了哪个浏览器写进 `项目适配.md` 的「工具」那一段。

**工具自己的输出放进一个隐藏目录**，别摊在项目根目录：Playwright 的 `outputDir` 设成 `.playwright/结果`，
html 报告的 `outputFolder` 设成 `.playwright/报告页`（`npm run report` 跟着改），回放用 `trace: 'retain-on-failure'`。
`.gitignore` 里写 `.playwright/`。要留下来给人看的，由 `/qa-4-跑测试` 挑出来放进 `报告/`。

常见的一个：Playwright 的登录脚本放在 `测试数据/`，而全局 `testDir` 是 `./脚本` ——
登录那个 project 要单独写 `testDir: './测试数据'`，不然报 `No tests found`。

## 二、怎么看界面：读无障碍树，不要截图

同一屏（1440x900）实测：

| 读法 | 约耗 token |
| --- | --- |
| 截图 | 1700 |
| 无障碍树全量 | 470 |
| 纯文字 | 150 |
| **只取可交互元素** | **60** |

**默认只取可交互元素和可见文字**，丢掉布局容器；看不明白再读全量。
截图是最后手段，只在"样式错位、图片没加载、颜色不对"这类非看图不可时用。
功能对不对，无障碍树里全都有，而且比看图准——模型看图会认错字，读文字不会。

## 三、怎么定位一个元素

按这个优先级：可访问名称（role + name）> `data-testid` 这类测试专用属性 > 文本 > CSS 选择器 > XPath。
越靠后越容易被改版打断。

**定位信息只能写进 `绑定/`，`用例/` 里一个都不许出现。**

## 四、怎么知道现在在哪一页

看地址栏。每一步操作之前先确认当前在哪一页，不要假设"上一步点完就一定到了那一页"。

地图 yaml 里「路径」这一栏填 URL 路径（`/profile`）。

## 五、登录态

Playwright 的 storageState：登录一次，存成文件，以后每次带着跑。

- **同一个浏览器里，一个站点只能登一个账号。** 要换账号，就换一个互相隔离的浏览器环境，
  不要在同一个里退出再登
- 登录要密码的：**不要替用户输密码**。让他自己登录一次，存下登录状态。
  这条命令包成 npm 命令，用户只敲 `npm run login`，不用敲中文路径：

```json
"scripts": {
  "login": "playwright codegen --channel=chrome --save-storage=测试数据/登录态/企业用户.json <登录页>"
}
```

用的是 Edge，把 `--channel=chrome` 换成 `--channel=msedge`。

密码不进任何文件。用户把密码发给你了，**明确拒绝使用，并建议他改密码**。

**存完必须验一次：拿这份登录状态打开首页，看落地在哪一页、页面上显示的是谁。**
没真正登录时文件照样会生成，大小跟有效的那份差不多，看文件根本看不出来。
脚本"跑通了"不等于"存对了"——这是这一步最容易过去的假通过。

顺手记下登录态的有效期，写进 `项目适配.md`。过期了用户得重新登录一次，
不然下一步所有用例都会卡在登录页。

## 六、证据怎么留

| 要什么 | 怎么来 |
| --- | --- |
| 回放 | Playwright 的 trace（`trace: 'retain-on-failure'`），点开能一步步看点了什么 |
| 页面结构 | 把读到的无障碍树存成 yaml |
| 接口原文 | 从网络面板或 trace 里取 |
| 截图 | 只在样式类问题时留 |

## 七、存成脚本用什么

就用 Playwright，脚本放 `脚本/<流程名>.spec.ts`。按编号筛单条：`npx playwright test -g "批次-01"`。
