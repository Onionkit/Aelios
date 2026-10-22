# 让 AI 帮你配 Aelios

这份说明是写给 AI 看的。你用 Claude Code、Codex、Cursor 这类能跑命令的 AI，把下面这段话发给它就行：

```text
帮我配好 Aelios（我的 AI 记忆网关）。先读 https://raw.githubusercontent.com/wusaki0723/Aelios/main/docs/setup-with-ai.md ，照着做。需要我在浏览器里点、或者需要我自己填 key 的时候，停下来告诉我去哪儿填。
```

配完你会拿到三样东西，填进你平时用的聊天软件就能用：

- **地址**（Base URL）
- **API Key**
- **模型名**

从此聊天经过 Aelios，每句话自动记、下次自动想起来，不用装任何上传、召回的脚本。

整个过程你自己要动手的只有两件事：在浏览器里点一次「允许」让 AI 登录 Cloudflare；把你在模型厂商那里的 key（比如 DeepSeek 的）存进 Cloudflare。key 由你自己填，不经过 AI。其余的，包括生成 Cloudflare 的令牌、配助手、导入以前的记忆，都可以交给 AI。

---

## 给 AI：先守这几条

- 用用户的语言跟用户说话。每一步先用一句话说你要做什么。
- **别让用户把凭证贴进对话**：模型厂商的 key、Cloudflare API Token、GitHub Token 都不行。厂商的 key 由用户自己存进 Cloudflare（第 3 步）。唯一的例外是 Aelios 自己的访问钥匙 `CHATBOX_API_KEY`：它本来就要填进用户的客户端，用户愿意给你就收。
- **你生成的 Cloudflare 令牌别打印、别写进文件**，用管道直接存进 Worker（第 4 步的命令就是这么写的）。
- **没经用户同意，别删、别换已有的东西**：AI Gateway、厂商 key、Worker 的 Secret、已配好的助手、D1 数据库、Vectorize 索引都不行。记忆就存在里面。
- 部署只用 `npm run deploy`。这个仓库是 wrangler 项目，**别跑 `cf deploy`、`cf dev`、`cf build`、`cf migrate`**。`cf` 只拿来配账号里的东西：AI Gateway、令牌、Worker 的 Secret。
- 卡住了别换着法子硬试。把报错原文和你的判断告诉用户，问下一步。

## 0. 先问用户

1. **Aelios 部署好了吗？** 没有就先做文末「附：还没部署的」，做完回来。部署好了就要 Worker 地址（形如 `https://companion-memory-proxy.<子域>.workers.dev`），跑一下 `curl -s <Worker 地址>/health`，看到 `"ok": true` 再往下。
2. **Aelios 的钥匙 `CHATBOX_API_KEY` 还记得吗？** 就是 MCP 连接器地址里 `token=` 后面那串。忘了也行，第 2 步能重设。
3. **模型从哪儿来？** 按回答选路线：

   | 用户有什么 | 走哪条 |
   |---|---|
   | 某家的 API key：DeepSeek、OpenAI、Gemini、OpenRouter、Anthropic 等，包括在网站上用账号登录（OAuth）换来的 key | **API 模式**，key 存在 Cloudflare AI Gateway 里（BYOK）。第 1 步到第 7 步全做 |
   | 一个中转站给的地址和 key，或者 AI Gateway 的下拉列表里没有的厂商 | **API 模式**，同样走 AI Gateway：先建一个自定义 provider，key 照样存在它下面。第 3 步多做「自定义 provider」那段 |
   | 只有 Claude 订阅（Pro、Max），没有 API | 订阅变不成 API。只能走 **MCP 连接器**，用 Claude Code 的话再加仓库里的 Hook（`integrations/claude-code/`）。直接跳到第 7 步的 MCP 部分 |

   告诉用户：**API 模式加 MCP 连接器是满血用法**。API 模式每句话自动记、自动召回；MCP 让助手能主动查、主动记、自己审记忆。只用 MCP 加 Hook 是权宜之计。

4. **在哪个软件里聊？** Chatbox、Cherry Studio、自己的 agent、Claude Code、Codex……决定第 6、7 步怎么填。
5. **助手叫什么？** 一个小写英文名，会出现在地址里，比如 `daily`。再问日记里怎么称呼用户和助手，可以不填。
6. **有没有以前的记忆、聊天记录要搬过来？** 有的话配完做「导入以前的记忆和聊天记录」。

## 1. 装 Cloudflare 命令行，登录

```bash
npm install --global cf
cf auth whoami
```

没登录就跑下面这条。先告诉用户：会给一个网址和一串码，在浏览器里打开网址、确认那串码、点允许。

```bash
cf auth login
```

你跑在远程机器、容器或云端沙盒里的话，加 `--no-browser`，把打印出来的网址交给用户。`cf` 的登录和 wrangler 的登录是分开的，各登各的。

`whoami` 列出不止一个账号时，问用户用哪个，之后每条 `cf` 命令前面带上 `CLOUDFLARE_ACCOUNT_ID=<那个 32 位 ID>`。账号 ID 不是密钥，但也别写进仓库里的文件。

下面的命令默认 Worker 叫 `companion-memory-proxy`。用户改过名就换成他的。

## 2. 确认 Aelios 的钥匙

```bash
curl -s <Worker 地址>/api/gateway/config -H "Authorization: Bearer <钥匙>"
```

返回 JSON（带 `"version":3`）就对了，把它留着，第 5 步要用。返回 401 就是钥匙不对。

用户忘了钥匙：先说清楚，重设以后所有已经在用这把钥匙的地方都要改，包括 MCP 连接器地址和别的客户端。用户同意了再跑：

```bash
KEY="sk-aelios-$(node -e "process.stdout.write(require('node:crypto').randomBytes(24).toString('base64url'))")" \
  && cf workers secrets update CHATBOX_API_KEY --worker companion-memory-proxy --type secret_text --text "$KEY" >/dev/null \
  && echo "$KEY"
```

把钥匙告诉用户，请用户存进密码管理器。别写进任何文件。

## 3. 用户自己存厂商的 key（BYOK）

### 先准备网关

AI Gateway 叫 `default`。先看有没有：

```bash
cf ai-gateway gateways get default
```

没有就建一个。BYOK 要求网关开着认证：

```bash
cf ai-gateway gateways create --id default --authentication true \
  --cache-invalidate-on-update true --cache-ttl 0 --collect-logs true \
  --rate-limiting-interval 0 --rate-limiting-limit 0
```

已经有了但 `authentication` 是 `false` 的：问用户能不能打开，同意了用 `cf ai-gateway gateways update default` 改。这条会整份覆盖网关设置，所以把 `get` 回来的其它字段原样带上，只把 `--authentication` 改成 `true`。

### 让用户存 key

中转站、下拉列表里没有的厂商，先跳到下面「自定义 provider」那段。

两种办法，让用户挑。这一步你别碰 key。

**A. 在 Cloudflare 后台点。** 把这个链接给用户：<https://dash.cloudflare.com/?to=/:account/ai/ai-gateway>。进 `default` → **Provider Keys** → **Add API Key**，选厂商、粘贴 key，别名填 `default`，保存。

**B. 在用户自己的终端里跑。** 把下面这条给用户，让用户自己粘贴、自己跑（`deepseek` 换成对应厂商）。key 输入时不显示，也不会留在命令历史里：

```bash
read -rs -p "粘贴 key 后回车：" K && echo \
  && cf ai-gateway gateways providers create default --provider-slug deepseek --alias default --default-config true --secret "$K" >/dev/null \
  && echo 已存; unset K
```

常见厂商的写法：`deepseek`、`openai`、`anthropic`、`google-ai-studio`、`openrouter`。别的厂商看后台的下拉列表。

用户说存好了，你确认一下（只看有没有，看不到 key 本身）：

```bash
cf ai-gateway gateways providers list --gateway-id default
```

### 中转站，或者下拉列表里没有的厂商：自定义 provider

AI Gateway 可以自定义 provider，中转站和没内置的厂商都这样接，照样存 key、照样有日志。

1. **建自定义 provider**（这一步没有秘密，你来做）。先问用户那家的 OpenAI 兼容地址。`base_url` 填 `/v1` 前面那一段，末尾的斜杠留着：比如 Kimi Coding 的接口是 `https://api.kimi.ai/coding/v1/chat/completions`，`base_url` 就填 `https://api.kimi.ai/coding/`。AI Gateway 会自己在后面接 `v1/chat/completions`。`slug` 用小写英文和连字符，比如 `kimi`：

   ```bash
   cf ai-gateway custom-providers create --name <显示名> --slug <slug> --base-url https://api.kimi.ai/coding/ --enable true
   ```

   `--enable true` 一定要带，不带建出来是停用的。

2. **用户存 key**：后台 `default` → **Provider Keys** → **Add API Key**，下拉里选刚建的这个 provider，别名填 `default`。在终端里存的话，`--provider-slug` 写 `custom-<slug>`；报错就改回后台存。

3. 模型名一律写成 `custom-<slug>/<那家的模型名>`，比如 `custom-kimi/kimi-for-coding`。

然后照常做第 4 步。

整个网关只有一个上游（第 5 步的 `upstream`），内置厂商和自定义 provider 都挂在同一个 AI Gateway 下面，可以混着用。

## 4. 生成 Cloudflare 令牌，存进 Worker

Aelios 要一把 Cloudflare 令牌去调 AI Gateway。这把由你生成，生成后直接存进 Worker，全程不显示。

先看 Worker 里是不是已经有了：

```bash
cf workers secrets list --worker companion-memory-proxy
```

已经有 `CLOUDFLARE_API_TOKEN` 的，问用户要不要换新的，不换就跳过这一步。

查这几个权限组的 ID（`--name` 的值要 URL 编码，空格写成 `%20`）：

```bash
cf accounts tokens permission-groups list --name "AI%20Gateway%20Run"
cf accounts tokens permission-groups list --name "AI%20Gateway%20Read"
cf accounts tokens permission-groups list --name "Workers%20AI%20Read"
cf accounts tokens permission-groups list --name "Vectorize%20Read"
```

前三个是聊天转发要的；`Vectorize Read` 给维护工具按 Vectorize 列记忆用，找不到可以不加。把 ID 写进一个临时的策略文件（这里面没有秘密）：

```json
[{
  "effect": "allow",
  "resources": { "com.cloudflare.api.account.<账号 ID>": "*" },
  "permission_groups": [ { "id": "<AI Gateway Run 的 ID>" }, { "id": "<AI Gateway Read 的 ID>" }, { "id": "<Workers AI Read 的 ID>" }, { "id": "<Vectorize Read 的 ID>" } ]
}]
```

存成 `/tmp/aelios-policy.json`，然后一条命令生成并存进 Worker。令牌只在管道里过一下，不打印：

```bash
V=$(cf accounts tokens create --name aelios-gateway --policies @/tmp/aelios-policy.json \
  | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{let j;try{j=JSON.parse(s)}catch{console.error("返回的不是 JSON");process.exit(1)}const r=j.result||j;if(!r.value){console.error("没拿到令牌，返回的字段：",Object.keys(r).join(", "));process.exit(1)}process.stdout.write(r.value)})') \
  && cf workers secrets update CLOUDFLARE_API_TOKEN --worker companion-memory-proxy --type secret_text --text "$V" >/dev/null \
  && echo 已存; unset V; rm -f /tmp/aelios-policy.json
```

建令牌被拒（比如登录的账号没有建令牌的权限）：别反复试。让用户在后台建一把，再自己填进 Worker：AI Gateway → `default` → **Settings** → **Authenticated Gateway** 那里建令牌；然后 **Workers & Pages** → `companion-memory-proxy` → **Settings → Variables and Secrets** → **Add**，类型选 **Secret**，名字 `CLOUDFLARE_API_TOKEN`，值是那把令牌。

## 5. 配助手

先看现在的记忆存在哪个空间：

```bash
curl -s <Worker 地址>/api/gateway/spaces -H "Authorization: Bearer <钥匙>"
```

新助手的 `namespace` 填记忆最多的那个，一般是 `default`。MCP 连接器写的也是 `default`，这样 API 和 MCP 用的是同一份记忆。

模型名：BYOK 一律写成 `厂商/模型`，比如 `deepseek/deepseek-chat`、`openai/gpt-5`、`google-ai-studio/gemini-2.5-pro`。拿不准就让用户在厂商文档里确认模型名，前面加上第 3 步的厂商写法。自定义 provider 写成 `custom-<slug>/<模型名>`。

然后 PATCH。**`identities` 是整份替换的**：第 2 步拿到的配置里已经有助手的，把它们原样放进列表，再加新的这个。

```bash
curl -sS -X PATCH <Worker 地址>/api/gateway/config \
  -H "Authorization: Bearer <钥匙>" -H "Content-Type: application/json" \
  -d '{
    "upstream": { "address": "<账号 ID>" },
    "identities": [
      { "slug": "daily", "namespace": "default", "keys": ["CHATBOX_API_KEY"],
        "models": ["deepseek/deepseek-chat"],
        "userName": "<用户的称呼>", "assistantName": "<助手的名字>" }
    ]
  }'
```

- `models` 只放用户真正拿来聊天的主模型。只有它们会记、会召回，别的模型路过不记。
- 用户没给称呼，就把 `userName`、`assistantName` 整项删掉。
- `upstream` 已经是对的就整项不写；写了会覆盖。

## 6. 试一句

```bash
curl -sS <Worker 地址>/daily/v1/chat/completions \
  -H "Authorization: Bearer <钥匙>" -H "Content-Type: application/json" \
  -d '{"model":"deepseek/deepseek-chat","messages":[{"role":"user","content":"你好，回我一个字就行"}]}'
```

有回复就通了。常见报错：

- 提到 **Unified billing**：Cloudflare 没找到用户存的 key，在用它自己的计费。多半是 key 存错了厂商、别名不是 `default`，或者模型名前缀和厂商对不上。
- **401、403**：令牌权限不够，或者网关认证没开。回第 3、4 步看。
- **Unknown gateway provider**：前缀写错了。自定义 provider 要写 `custom-<slug>/`，并且建的时候带了 `--enable true`。
- **404**：多半是自定义 provider 的 `base_url` 多写了 `/v1`（AI Gateway 会自己接），或者少了那家地址里的一段路径。
- **Upstream not configured**：第 5 步的 `upstream` 没填上。

Claude Code 走 `/daily/v1/messages`，Codex 走 `/daily/v1/responses`。这两条是转到厂商的原生接口，不是每家都有：`anthropic/` 的模型能走 messages，`openai/` 的能走 responses，别家多半只有上面那条 chat。用户要在这两个软件里用，先用 curl 试对应的路径，通了再改配置。

## 7. 交给用户

最后用一小块话交给用户，照这个样子：

```text
地址（Base URL）：<Worker 地址>/daily/v1
API Key：<钥匙>
模型：deepseek/deepseek-chat
```

再补几句：

- 模型名要一字不差，前缀也要。软件里拉模型列表的话，挑这个名字。
- 有的软件会自己在地址后面补 `/v1`，那就只填到 `/daily`。
- Claude Code：`ANTHROPIC_BASE_URL=<Worker 地址>/daily`，`ANTHROPIC_AUTH_TOKEN=<钥匙>`。
- Codex：`base_url = "<Worker 地址>/daily/v1"`，并设 `wire_api = "responses"`。
- 改用户的客户端配置文件前先问。别改你自己正在用的那份，改了你会断线。

**再接上 MCP 连接器，就是满血。** 连接器地址：

```text
<Worker 地址>/mcp?token=<钥匙>
```

- Claude 官方 App（网页、手机）：请用户打开 claude.ai/customize/connectors，添加自定义连接器，填上面的地址。工具权限里把只读那一组设成「总是允许」。再把 README「在 Claude 官方 App……里用 MCP 记忆」那段提示语贴进个人偏好，不然它不会主动用。
- Claude Code：征得同意后跑 `claude mcp add --transport http aelios "<Worker 地址>/mcp?token=<钥匙>"`。
- Codex：征得同意后跑 `codex mcp add aelios --url "<Worker 地址>/mcp?token=<钥匙>"`。
- 别的软件：支持远程 MCP（Streamable HTTP）的，填同一个地址。

让用户试：「请记住：我的测试暗号是苹果星星-0428。」过一会儿换个新对话问「我的测试暗号是什么？」，答出来就全通了。

## 导入以前的记忆和聊天记录

整理好的条目直接写进库；聊天记录要先让模型从原话里抽候选，抽出来的进审核页「导入的」那一栏，用户在 `<Worker 地址>/admin` 里点了才算数。先看导入设置，里面有限额、抽取模型和给旧 AI 的提示词：

```bash
curl -s "<Worker 地址>/v1/import/info?namespace=default" -H "Authorization: Bearer <钥匙>"
```

`namespace` 用第 5 步那个。

### 整理好的记忆条目

格式是一行一条，日期、类型、标签都能省：

```text
- 2025-12-15 [偏好] 不吃香菜 #饮食
```

1. **拿到条目。** 用户手上有导出文件（md、jsonl、json）就直接用。记忆在别的 AI 那里的，把 `info` 返回的 `data.prompt` 给用户，让用户贴给那个 AI，把它的回答存成一个文件交给你。
2. **预览，不写库：**

   ```bash
   curl -sS -X POST <Worker 地址>/v1/import/entries/preview \
     -H "Authorization: Bearer <钥匙>" -H "Content-Type: application/json" \
     --data-binary @<(node -e 'process.stdout.write(JSON.stringify({text:require("fs").readFileSync(process.argv[1],"utf8")}))' <文件>)
   ```

   把条数、各类型多少、`warnings` 和几条例子给用户看，用户点头再写。
3. **写入：** 把预览返回的 `data.entries` 每 25 条一批 POST 到 `/v1/import/entries`：

   ```json
   {
     "namespace": "default",
     "fingerprint": "<整个文件的 sha256>",
     "name": "<文件名>",
     "mode": "direct",
     "start_index": 0,
     "total": 120,
     "entries": [ "...这一批的 25 条，原样用预览返回的对象..." ]
   }
   ```

   - `"direct"` 直接写进库：跟库里一字不差的跳过，跟库里某条很像的不硬写、转去审核页「导入的」等用户定，其余新建；带 `factKey` 的按 key 改写，旧版留着。用户说想一条条过目，才改成 `"review"`，全部先进审核。
   - 写完把返回的 `counts` 告诉用户：新建 `created`、改写 `updated`、跳过 `unchanged` 和 `duplicate`、转审核 `queued`、失败 `failed`。有 `queued` 的，请用户去 `/admin` → **审核队列** → **导入的** 看一眼。
   - 每批 `start_index` 加 25，`fingerprint`、`total` 不变。中途断了，用同一个 `fingerprint` 从断的地方接着发。

### 聊天记录

聊天记录会一天天交给夜里整理记忆的那个模型（`DREAM_MODEL`）读，抽出候选记忆。要花 Workers AI 的用量，量大时先导两三天的看看效果，再全导。

**最省事：让用户在网页上导。** `<Worker 地址>/admin` → **更多** → **导入导出** → **导入聊天记录**，选文件就行，页面会拆、会切段、显示进度，关了下次选同一个文件接着做。它认：

- ChatGPT、Claude 导出的 `conversations.json`
- 酒馆（SillyTavern）的 `.jsonl`
- 通用 JSON / JSONL：消息数组，每条有 `role`（或 `sender`、`author`）和 `content`（或 `text`）
- 纯文本：「名字：内容」一行一句，「[2025-12-15 23:01] 名字: 内容」，日期单独一行当小标题，QQ、微信那种「时间 名字」一行、正文下一行

别的格式：你把它转成上面的通用 JSONL，每行一条 `{"role":"user","content":"……","time":"2025-12-15T23:01:00+08:00"}`（助手说的 `role` 是 `assistant`），交给用户在网页上导。

**用户要你直接导的话**，用接口：

1. 开一批（同一份文件用同一个 `fingerprint`，就能接着上次做）：

   ```json
   POST /v1/import/batches
   { "namespace": "default", "fingerprint": "<文件的 sha256>", "name": "<文件名>",
     "total_chunks": 42, "total_messages": 2300, "first_date": "2025-06-01", "last_date": "2025-12-31" }
   ```

   返回里的 `data.batch.id` 是这批的 ID，`data.done_keys` 是已经做完的段，跳过它们。
2. 按天切段，一段一段发。一段默认最多 60 条、3 万字，硬上限 80 条、4 万字；一天太多就切成几段：

   ```json
   POST /v1/import/chat/chunk
   { "namespace": "default", "import_id": "<批 ID>",
     "chunk": { "key": "2025-12-15#0", "day": "2025-12-15",
       "messages": [ { "seq": 0, "role": "user", "content": "……", "time": "2025-12-15T23:01:00+08:00", "speaker": "<名字>" } ] } }
   ```

   `key` 每段不同、不超过 64 个字符；`seq` 在一段里不重复。一段一段串着发，别并发。
3. 做完告诉用户去 `/admin` → **审核队列** → **导入的** 里挑。

## 附：还没部署的

不熟命令行的用户，也可以照 README「1. 部署」在 Cloudflare 后台点着装，装好再回第 0 步。下面是你替用户在命令行里装：

```bash
node -v   # 要 22 或更高
gh repo fork wusaki0723/Aelios --clone --default-branch-only   # 没有 gh：请用户在网页上点 Fork，再 git clone 用户的 fork
cd Aelios
npm ci
npx wrangler login   # 远程机器、容器、云端沙盒里用：npx wrangler@latest login --device
npm run deploy
```

`npm run deploy` 会建 D1 数据库、Vectorize 索引、队列，再部署 Worker，同名的已经有了就直接用。输出最后的 `https://companion-memory-proxy.<子域>.workers.dev` 就是 Worker 地址。

常见卡点：

- **账号没有 workers.dev 子域**：请用户在后台打开一次 **Workers & Pages**，按提示选一个子域，再重跑 `npm run deploy`。
- **Vectorize 索引维度不对、里面有数据**：脚本会停下，不会自己删。把原文给用户看，让用户决定。
- **多个账号选不出来**：命令前带 `CLOUDFLARE_ACCOUNT_ID=<32 位 ID>`。

部署完：

1. 部署脚本把数据库 ID 和账号 ID 写进了 `wrangler.toml`，别提交：`git checkout -- wrangler.toml`。
2. 用第 2 步那条命令生成钥匙（新装的不用问「要不要换」）。
3. 推荐让 Cloudflare 改从用户的 fork 构建，以后在 GitHub 上点 **Sync fork** 就自动更新。只能用户在后台点：**Workers & Pages** → `companion-memory-proxy` → **Settings → Build** → **Connect**，选 fork（第一次会让装 Cloudflare 的 GitHub App），构建命令 `npm ci`，部署命令 `npm run deploy`。连好以后，有 gh 的话可以替用户同步：`gh repo sync <用户的 fork>`。

## 附：其它

- **查日志**：`npx wrangler tail companion-memory-proxy`。
- **别的设置**（召回条数、夜间整理用的模型等）：在 `<Worker 地址>/admin` 的「设置」里改，或者用同一个 `PATCH /api/gateway/config` 改 `settings`（按项合并，不是整份替换）。
