# Cloudflare 邮件设置详解

这篇文档只讲一件事：

把 `Cloudflare 域名邮箱 + 一个简单的收信 API` 配好，让我们的 `tavily-key-generator` 可以直接使用。

如果你只想先记住一句话，可以先记这个：

> Cloudflare 负责收邮件，我们自己的 API 负责把邮件内容提供给项目读取。

---

## 一、先搞清楚项目到底需要什么

很多人第一次看到 `Cloudflare`、`邮箱`、`API` 这几个词放在一起，会误以为：

- 项目直接调用 Cloudflare 官方邮箱接口
- Cloudflare 后台里点几下就能直接给项目读邮件

这两个理解都不对。

我们项目当前的实际逻辑是：

1. 自动生成一个随机邮箱地址，比如 `tavily-ab12cd34@tvmail.example.com`
2. 用这个邮箱去注册 Tavily
3. 等 Tavily 把验证码邮件发过来
4. 项目去请求你提供的邮件 API：

```text
GET {EMAIL_API_URL}/messages?address=随机邮箱地址
Authorization: Bearer {EMAIL_API_TOKEN}
```

然后从返回结果里读取：

- `subject`
- `text`
- `html`

再提取 6 位验证码。

换句话说，项目真正需要的是两部分能力：

1. **能接住发到你域名上的邮件**
2. **能通过一个 HTTP API 把邮件内容查出来**

Cloudflare 只能很好地解决第 1 部分。  
第 2 部分需要我们自己补一层很薄的 API。

---

## 二、整体架构长什么样

建议你把整个链路理解成下面这样：

```text
Tavily 发送验证码邮件
        ↓
tavily-随机串@你的域名
        ↓
Cloudflare Email Routing 收到邮件
        ↓
Catch-all 规则命中
        ↓
邮件交给 Email Worker
        ↓
Worker 解析邮件并写入 D1
        ↓
我们的项目请求 /messages?address=...
        ↓
拿到邮件内容并提取验证码
```

这个结构看起来步骤多，但每一层都很清楚：

- `Email Routing` 负责收
- `Worker` 负责处理
- `D1` 负责存
- `API` 负责查

---

## 三、为什么推荐用子域名，而不是直接拿主域名上

如果你的主域名已经在跑正式邮箱，比如：

- Google Workspace
- Zoho Mail
- 腾讯企业邮
- 阿里云企业邮箱

那最稳的做法是：

**不要直接拿主域名硬改，单独准备一个子域名。**

例如：

- `tvmail.example.com`
- `keys.example.com`
- `verify.example.com`

为什么这么做：

1. 不会影响你现有正式邮箱
2. 配置更干净，排障更简单
3. 更适合这种“专门给脚本自动收验证码”的用途

你完全可以理解成：

- 主域名继续跑正常业务邮箱
- 子域名专门给 `tavily-key-generator` 收验证码

---

## 四、开始前你需要准备什么

正式开配前，建议先确认这几项：

1. 你已经有一个接入 Cloudflare 的域名
2. 你能正常登录 Cloudflare Dashboard
3. 你已经决定好要用哪个域名或子域名来收验证码邮件
4. 你愿意顺手部署一个很小的 Cloudflare Worker
5. 你本地能使用 `node`、`npm`、`wrangler`

如果还没有 `wrangler`，先安装：

```bash
npm install -g wrangler
```

然后登录：

```bash
wrangler login
```

---

## 五、第一步：把域名或子域名接入 Cloudflare

如果你的域名还没接入 Cloudflare：

1. 登录 Cloudflare
2. 点 `Add a site`
3. 输入你的域名
4. 按提示修改 nameserver
5. 等待状态变成 `Active`

如果域名早就在 Cloudflare 里，你只需要继续下一步。

如果你打算用子域名，建议先想好一个专门的名字，比如：

```text
tvmail.example.com
```

后面项目里 `EMAIL_DOMAIN` 就要填这个值。

---

## 六、第二步：开启 Email Routing

进入 Cloudflare 控制台：

```text
Cloudflare Dashboard
-> 选择你的域名
-> Email
-> Email Routing
```

第一次进入时，Cloudflare 一般会提示你开启 Email Routing。

你照着引导做就行，核心动作就两个：

1. 启用 Email Routing
2. 允许 Cloudflare 自动添加需要的 DNS 记录

这些记录通常包括：

- `MX`
- `TXT`

你在这里需要特别注意下面几件事。

### 1. 不要和现有 MX 记录冲突

如果这个域名本来就有别的邮件服务在使用 `MX` 记录，Cloudflare 可能会提示冲突。

简单说：

- 如果这个域名已经给正式邮箱用了，就别直接拿它做这套
- 最稳的是换一个子域名专门用

### 2. SPF 冲突要及时修

Cloudflare 后台如果提示 SPF 问题，别忽略。

一个域名通常只应该有一个有效 SPF 入口。  
如果你有多条 SPF，很容易导致 Email Routing 状态异常或者验证失败。

### 3. DNS 记录建议保持 Cloudflare 自动生成的状态

Email Routing 正常工作后，尽量不要手动乱改它自动生成的邮件 DNS 记录。

---

## 七、第三步：一定要开启 Catch-all

这一步是整个配置里最关键的一步。

为什么说关键？

因为我们的项目不是使用固定邮箱地址，而是每次自动生成随机地址。

比如这类地址：

```text
tavily-a1b2c3d4@tvmail.example.com
tavily-z9y8x7w6@tvmail.example.com
```

也就是说，你不可能在 Cloudflare 后台一个个手动创建这些地址。

正确做法是：

**开启 Catch-all，让所有随机地址都能被统一接住。**

操作方法：

1. 进入 `Email Routing`
2. 找到 `Routes`
3. 找到 `Catch-all address`
4. 开启它
5. 把动作设置成交给 `Worker`

这一步配置完成后，所有发到这个域名、但没有单独配置规则的邮箱地址，都会统一走 Catch-all。

这正好就是我们项目需要的效果。

---

## 八、第四步：不要把邮件转发到普通邮箱，直接交给 Worker

Cloudflare Email Routing 有几种处理方式，比如：

- 转发到另一个真实邮箱
- 交给 Worker 处理

对我们的项目来说，最佳选择是：

**直接交给 Worker。**

原因很简单：

如果你转发到 Gmail、Outlook 或者其他邮箱，再让项目去登录那个邮箱查信，会变得很绕：

- 还要搞 IMAP / POP3 / 网页抓取
- 延迟更高
- 风控更多
- 结构也不稳定

而直接交给 Worker 的好处是：

- 邮件一到就能被代码接住
- 可以按我们项目需要的格式保存
- 后续只用请求 `/messages` 就行

所以这套链路最推荐的做法是：

```text
Catch-all -> Send to Worker
```

---

## 九、第五步：创建一个专门的邮件 Worker 项目

本地执行：

```bash
npm create cloudflare@latest tavily-mail-api
cd tavily-mail-api
npm i postal-mime
```

这里安装 `postal-mime` 的原因很简单：

Cloudflare 交给 Worker 的是原始邮件内容，我们需要把它解析成更容易读取的结构，比如：

- 主题
- 文本正文
- HTML 正文

这样我们的项目就能直接从 API 返回里拿 `subject / text / html`。

---

## 十、第六步：创建 D1 数据库

接下来创建数据库：

```bash
npx wrangler d1 create tavily_mail
```

创建完成后，Cloudflare 会返回一个 `database_id`。  
记下来，后面 `wrangler.toml` 要用。

然后新建一个 `schema.sql`：

```sql
CREATE TABLE IF NOT EXISTS messages (
  id TEXT PRIMARY KEY,
  address TEXT NOT NULL,
  subject TEXT,
  text TEXT,
  html TEXT,
  received_at INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_messages_address_time
ON messages(address, received_at DESC);
```

执行初始化：

```bash
npx wrangler d1 execute tavily_mail --file=schema.sql
```

这张表足够应付我们项目当前的用法。

字段解释：

- `id`：邮件唯一标识
- `address`：收件地址，也就是项目生成出来的那个随机邮箱
- `subject`：邮件主题
- `text`：纯文本正文
- `html`：HTML 正文
- `received_at`：收信时间，用于排序

---

## 十一、第七步：写 Worker 代码

在 `src/index.ts` 里放下面这份最小可用版本：

```ts
import PostalMime from "postal-mime";

export interface Env {
  DB: D1Database;
  API_TOKEN: string;
}

function unauthorized() {
  return new Response("Unauthorized", { status: 401 });
}

function checkAuth(req: Request, env: Env) {
  const auth = req.headers.get("Authorization") || "";
  return auth === `Bearer ${env.API_TOKEN}`;
}

export default {
  async email(message: ForwardableEmailMessage, env: Env) {
    const parsed = await new PostalMime().parse(message.raw);
    const id =
      parsed.messageId ||
      `${String(message.to).toLowerCase()}-${Date.now()}-${crypto.randomUUID()}`;

    await env.DB.prepare(
      `INSERT OR REPLACE INTO messages
       (id, address, subject, text, html, received_at)
       VALUES (?1, ?2, ?3, ?4, ?5, ?6)`
    )
      .bind(
        id,
        String(message.to).toLowerCase(),
        parsed.subject || "",
        parsed.text || "",
        typeof parsed.html === "string" ? parsed.html : "",
        Date.now()
      )
      .run();
  },

  async fetch(req: Request, env: Env) {
    if (!checkAuth(req, env)) {
      return unauthorized();
    }

    const url = new URL(req.url);

    if (req.method === "GET" && url.pathname === "/messages") {
      const address = (url.searchParams.get("address") || "").trim().toLowerCase();

      if (!address) {
        return Response.json(
          { error: "missing address" },
          { status: 400 }
        );
      }

      const result = await env.DB.prepare(
        `SELECT id, subject, text, html, received_at
         FROM messages
         WHERE address = ?1
         ORDER BY received_at DESC
         LIMIT 20`
      )
        .bind(address)
        .all();

      return Response.json({
        messages: (result.results || []).map((row: any) => ({
          id: row.id,
          subject: row.subject || "",
          text: row.text || "",
          html: row.html || "",
          received_at: row.received_at,
        })),
      });
    }

    return new Response("Not found", { status: 404 });
  },
};
```

你不需要一开始就把它想得多复杂。

这段代码就做两件事：

### `email()`

当 Cloudflare 收到邮件后，会自动调用这里。  
我们在这里把邮件解析出来，然后写入 D1。

### `fetch()`

当我们的项目去请求：

```text
/messages?address=某个随机邮箱
```

就从 D1 里把这个邮箱最近收到的邮件取出来。

---

## 十二、第八步：配置 `wrangler.toml`

写一个最小版本：

```toml
name = "tavily-mail-api"
main = "src/index.ts"
compatibility_date = "2026-03-16"

[[d1_databases]]
binding = "DB"
database_name = "tavily_mail"
database_id = "替换成你的 D1 database_id"
```

然后设置一个自己的 API Token：

```bash
npx wrangler secret put API_TOKEN
```

这里输入一个你自己定义的长字符串，比如：

```text
your-super-long-random-token
```

这一步要记住一个很重要的点：

**这个 `API_TOKEN` 才是后面要填到项目 `EMAIL_API_TOKEN` 里的值。**

不是下面这些：

- 不是 Cloudflare Global API Key
- 不是 Cloudflare API Token
- 不是 Zone Token
- 不是账户级密钥

就是你这个 Worker API 自己的 Bearer Token。

---

## 十三、第九步：部署 Worker

执行：

```bash
npx wrangler deploy
```

部署成功后，你会得到一个地址，通常类似：

```text
https://tavily-mail-api.xxx.workers.dev
```

这个地址就是你后面要填到项目里的：

```env
EMAIL_API_URL=...
```

---

## 十四、第十步：把 Catch-all 路由到这个 Worker

现在回到 Cloudflare Dashboard：

```text
Email
-> Email Routing
-> Routes
```

找到 `Catch-all address`，把动作设置为：

```text
Send to a Worker
```

然后选中你刚刚部署好的 Worker。

到这里，Cloudflare 这边的“收信链路”就齐了。

链路会变成：

```text
发到任意随机邮箱
-> Cloudflare Catch-all 命中
-> 交给 Worker
-> Worker 写入 D1
-> 我们的项目去查 API
```

---

## 十五、第十一步：在项目里填写 `.env`

如果你只准备用一个域名：

```env
EMAIL_PROVIDER=cloudflare
EMAIL_API_URL=https://tavily-mail-api.xxx.workers.dev
EMAIL_API_TOKEN=你刚刚设置的API_TOKEN
EMAIL_DOMAIN=tvmail.example.com
```

如果你想准备多个域名，让项目启动时自由选择：

```env
EMAIL_PROVIDER=cloudflare
EMAIL_API_URL=https://tavily-mail-api.xxx.workers.dev
EMAIL_API_TOKEN=你刚刚设置的API_TOKEN
EMAIL_DOMAINS=tvmail.example.com,keys.example.com
```

你可以这样理解这些变量：

### `EMAIL_PROVIDER`

固定填：

```text
cloudflare
```

表示当前走的是 Cloudflare 域名邮箱方案。

### `EMAIL_API_URL`

填你自己这个 Worker API 的地址。

注意：

**这里不是 Cloudflare 官方控制台 API 地址。**

### `EMAIL_API_TOKEN`

填你自己设置给 Worker 的 Bearer Token。

注意：

**这里不是 Cloudflare 平台 API Key。**

### `EMAIL_DOMAIN`

填邮箱后缀，比如：

```text
tvmail.example.com
```

项目会自动拼成：

```text
tavily-随机串@tvmail.example.com
```

### `EMAIL_DOMAINS`

如果你填多个域名，项目启动时会让你选本轮用哪一个。

---

## 十六、项目是怎么拼邮箱地址的

这一点很多人配置时会忽略，所以这里单独说清楚。

项目当前的逻辑不是读取一个固定邮箱账号，而是自动生成随机地址。

类似这样：

```text
tavily-abcdefgh@tvmail.example.com
```

所以：

- 你不需要提前创建具体的 `tavily-abcdefgh`
- 你也不需要一条条手工建邮箱账号
- 你只需要保证 `Catch-all` 能接住所有随机地址

这是为什么前面一直强调：

**Catch-all 一定要开。**

---

## 十七、第十二步：上线前先手工验证

建议不要一上来就直接跑项目。

先做 3 个小验证，成功率会高很多。

### 验证 1：人工发一封测试邮件

随便找一个不存在但会命中 Catch-all 的地址，比如：

```text
test-123@tvmail.example.com
```

从另一个邮箱给它发一封邮件。

### 验证 2：直接用 curl 查 API

```bash
curl -H "Authorization: Bearer 你的API_TOKEN" \
  "https://tavily-mail-api.xxx.workers.dev/messages?address=test-123@tvmail.example.com"
```

如果返回结构里能看到：

- `messages`
- `subject`
- `text`
- `html`

说明收信链路已经通了。

### 验证 3：再跑项目

```bash
python3 run.py
```

如果验证码阶段项目能自动继续，说明这套配置已经完全打通。

---

## 十八、最常见的错误

下面这些坑非常常见，建议你配置时顺手对照一遍。

### 1. 把 `EMAIL_API_TOKEN` 填成 Cloudflare 官方 API Key

这是错的。

这里要填的是你自己 Worker 的 Bearer Token。

### 2. 没开 Catch-all

你们项目用的是随机邮箱名。  
如果不打开 Catch-all，随机地址收不到邮件。

### 3. 把 `EMAIL_DOMAIN` 填成 Worker 地址

这是错的。

- `EMAIL_DOMAIN` 是邮箱后缀
- `EMAIL_API_URL` 才是 Worker 地址

### 4. 主域名上已经在跑正式邮箱，还直接开启 Email Routing

这很容易和现有邮件服务冲突。

最稳做法永远是：

**用一个单独子域名。**

### 5. API 能打开，但项目拿不到邮件

通常排查这几项：

1. `Authorization` token 对不对
2. 查询的 `address` 和实际收件地址是否一致
3. Worker 是否真的把邮件写进了 D1
4. Catch-all 是否真的已经启用
5. 路由动作是否真的是 `Send to Worker`

### 6. 邮件确实发了，但 Cloudflare 后台一直不稳定

优先检查：

- 是否有旧 MX 冲突
- 是否有多个 SPF 记录
- 是否误改了 Email Routing 自动生成的 DNS 记录

---

## 十九、最推荐的落地方式

如果你不想走弯路，最推荐直接照这个方案来：

1. 准备一个专用子域名，比如 `tvmail.example.com`
2. 在 Cloudflare 开启 Email Routing
3. 开启 Catch-all
4. Catch-all 动作设为 `Send to Worker`
5. Worker 解析邮件并写入 D1
6. Worker 暴露 `GET /messages?address=...`
7. 在项目 `.env` 里填：

```env
EMAIL_PROVIDER=cloudflare
EMAIL_API_URL=你的Worker地址
EMAIL_API_TOKEN=你的Worker Bearer Token
EMAIL_DOMAIN=你的收信域名
```

这套方案和项目当前实现是最贴合的，不需要额外魔改项目逻辑。

---

## 二十、给第一次配置的人一个最简单的理解方式

如果你还是觉得前面信息很多，可以把这件事理解成下面这句话：

### Cloudflare 是前台，Worker 是收件员，D1 是仓库，项目是取件的人。

它们分别做的事是：

- Cloudflare：把发到你域名上的邮件接住
- Worker：把邮件拆开，保存下来
- D1：存邮件内容
- 项目：按邮箱地址去查最新邮件

只要这个分工想明白了，后面整个配置就不难了。

---

## 二十一、项目配置示例汇总

### 单域名

```env
EMAIL_PROVIDER=cloudflare
EMAIL_API_URL=https://tavily-mail-api.xxx.workers.dev
EMAIL_API_TOKEN=your-super-long-random-token
EMAIL_DOMAIN=tvmail.example.com
```

### 多域名

```env
EMAIL_PROVIDER=cloudflare
EMAIL_API_URL=https://tavily-mail-api.xxx.workers.dev
EMAIL_API_TOKEN=your-super-long-random-token
EMAIL_DOMAINS=tvmail.example.com,keys.example.com
```

---

## 二十二、参考资料

如果你想进一步对照官方文档，可以看这些 Cloudflare 文档：

- [Enable Email Routing](https://developers.cloudflare.com/email-routing/get-started/enable-email-routing/)
- [Email Routing Addresses and Rules](https://developers.cloudflare.com/email-routing/setup/email-routing-addresses/)
- [Email Routing Subdomains](https://developers.cloudflare.com/email-routing/setup/subdomains/)
- [Email Routing DNS Records](https://developers.cloudflare.com/email-routing/setup/email-routing-dns-records/)
- [Enable Email Workers](https://developers.cloudflare.com/email-routing/email-workers/enable-email-workers/)
- [Email Workers Runtime API](https://developers.cloudflare.com/email-routing/email-workers/runtime-api/)
- [Troubleshooting SPF records](https://developers.cloudflare.com/email-routing/troubleshooting/email-routing-spf-records/)
- [Test Email Routing](https://developers.cloudflare.com/email-routing/get-started/test-email-routing/)

---

## 二十三、最后的建议

如果你只是想尽快把项目跑起来，不要一开始就做太多扩展。

先按最小可用方案完成下面这些事：

1. 一个子域名
2. 一个 Catch-all
3. 一个 Worker
4. 一个 D1
5. 一个 `/messages` 接口

先把这条链跑通，再考虑：

- 多域名轮换
- 邮件自动清理
- 旧邮件保留策略
- 控制台可视化
- 统计与监控

这样最稳，也最不容易把自己绕进去。
