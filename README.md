# tiny-post-office · 只给你

两个人的小邮局（PWA）。后端零依赖（Node 内置 SQLite），前端零构建（纯静态 ES5/ES6 文件）。

## 快速开始

需要 Node.js 22.13 或更高版本。

```bash
cp .env.example .env        # 按需修改
npm run setup               # 设置两个人的名字、暗号（默认 6 位数字）、在一起的日期
npm start                   # http://127.0.0.1:3000
```

可选：`npm run vendor` 把 three.js 下载到 `web/vendor/`（开屏水晶球优先用本地文件，失败才走 CDN）。

测试：

```bash
npm test                                    # 后端冒烟测试（21 项，不需要浏览器）
pip install playwright && playwright install chromium
python tests/e2e.py                         # 端到端（20 项），截图在 tests/shots/
```

## 「每个页面都是沙盒」是怎么做到的

```
shell（web/index.html + web/shell/）      只做登录判断、导航、转发请求，不渲染任何业务内容
 ├─ iframe sandbox="allow-scripts"  → pages/login   登录
 ├─ iframe sandbox="allow-scripts"  → pages/boot    开屏（水晶球）
 ├─ iframe sandbox="allow-scripts"  → pages/home    首页
 ├─ iframe sandbox="allow-scripts"  → pages/studio  工坊
 ├─ iframe sandbox="allow-scripts"  → pages/mail    信箱
 └─ iframe sandbox="allow-scripts"  → pages/me      我的
```

- **隔离**：每个页面是独立的 iframe，不给 `allow-same-origin`。页面读不到 Cookie 和 localStorage，碰不到 shell 和别的页面，
  样式、全局变量互不污染；CSP 还设置了 `connect-src 'none'`，页面自己连网都做不到。
- **通信**：页面和 shell 之间只有一条 MessageChannel。请求 API、震动、弹提示、跳转，都由 shell 代办（`web/sdk/tpo.js`）。
  shell 只放行 `/api/` 下的路径，并限制每页请求频率。
- **出错只影响那一页**：页面脚本报错、没加载出来、10 秒没响应，shell 只在那一页上盖一张「出了点小问题 / 重试」，
  其余页面和底部导航照常。登录页坏了，会出现一个应急的暗号输入框。开屏坏了，会直接跳过去进首页。
- **页面内再分一层**：每个页面里的功能块用 `TPO.guard(name, el, fn)` 包起来，一块出错只显示这一块的「重试」。
- **看得见**：页面报错会转发到服务器 `data/client-errors.log`；「我的」页底部有每个页面的运行状态，可以单独重载。
- **Service Worker 故意什么都不缓存**：它是整站的单点。万一它出问题，在 `.env` 里设 `SW_KILL=1` 重启，所有设备下次打开会自动注销它。

**做不到的事**：一个页面里写出死循环或占满内存，浏览器可能整个标签页一起卡住，iframe 隔离不了这个。
重计算（比如以后的画板导出）请放进 Web Worker。

## 安全

- 暗号只存 scrypt 哈希；登录时同时校验两人的暗号，用匹配到的那个判断身份，所以两人暗号必须不同。
- 同一 IP 15 分钟错 5 次锁 15 分钟，全站 1 小时错 40 次锁 10 分钟。
- 会话是 HttpOnly + SameSite=Strict Cookie（走 HTTPS 时带 Secure）；写请求必须带 `x-tpo` 头并校验 Origin。
- 悄悄话的文字只有解锁后才会下发；密码锁在服务端校验，错 5 次锁 10 分钟。
- 阅后即焚在服务端计时，到时把正文从数据库里清除；时间胶囊没到时间，服务端不会返回内容。

## 部署到云服务器

```bash
sudo mkdir -p /opt/tiny-post-office && sudo cp -r . /opt/tiny-post-office
cd /opt/tiny-post-office && cp .env.example .env     # 放在 nginx 后面：HOST=127.0.0.1 TRUST_PROXY=1
node server/scripts/setup.js
sudo cp deploy/tiny-post-office.service /etc/systemd/system/ && sudo systemctl enable --now tiny-post-office
sudo cp deploy/nginx.conf.example /etc/nginx/conf.d/tiny-post-office.conf   # 改域名、签证书
```

备份只需要复制 `data/` 目录。

## 目录

```
server/   index.js 入口 · routes.js 全部接口 · auth.js 登录与防爆破 · db.js 建表与迁移 · static.js 静态文件与 CSP
          scripts/setup.js 初始化 · test/smoke.mjs 后端测试
web/      shell/ 外壳 · sdk/ 页面端 SDK 与共用部件 · pages/ 各页面（互相独立） · sw.js · manifest
tests/    e2e.py 端到端测试
deploy/   nginx 与 systemd 示例
```

新增一个页面：建 `web/pages/xxx/`（`index.html` + `main.js`），在 `web/shell/shell.js` 的 `PAGES` 里登记一行即可。

## 下一步

1. 工坊：手绘画板、杯贴小票（信箱表里已有 `payload` 字段给它们用）
2. Web Push + Android APK（TWA）
3. 每日一问、早晚安打卡、纪念日
4. NFC 碰一碰
