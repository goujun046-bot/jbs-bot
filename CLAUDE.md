# TG Bot 通用开发规则
# 适用所有 Telegram Bot 项目
# 每次新开对话，把此文件喂给 AI

---

## 🏗️ 技术栈

- **Bot 框架**：grammy
- **数据库**：mysql2/promise（连接池模式）
- **部署**：Railway
- **Node 版本**：>=18

---

## 🔴 绝对禁止

### 不得随意重构已有逻辑
修改某个函数前必须说明：① 为什么要改 ② 会影响哪些地方。不能静默重写已有功能。

### 不得改动 A 时影响 B
每次修改范围最小化。改完必须说明：「这个改动只影响了 xxx，不影响其他模块。」

### 不得硬编码任何密钥或敏感信息
BOT_TOKEN、API Key、数据库密码等必须全部从环境变量读取，不得出现在代码里。

---

## 🟡 数据库约定

### 用 LONGTEXT 存 JSON，不用 JSON 类型列
mysql2 对 JSON 类型列会自动解析，写入时对象可能变成 [object Object]。

```sql
-- ❌ 有坑
data JSON NOT NULL

-- ✅ 安全
data LONGTEXT NOT NULL
```

### 存取统一走 JSON.parse / JSON.stringify
```js
// 存
await db.execute('INSERT ... VALUES (?, ?)', [id, JSON.stringify(obj)]);
// 取
return JSON.parse(rows[0].data);
```

### Railway 数据库地址区分内网/公网
```
# 本地开发（.env）：用公网地址
DB_HOST=xxx.proxy.rlwy.net
DB_PORT=xxxxx

# Railway 服务内部：用内网地址
DB_HOST=mysql.railway.internal
DB_PORT=3306
```
两个不能混用，本地用公网，Railway服务用内网。

### bot服务必须和MySQL在同一个Railway project里
跨project访问会ETIMEDOUT，不要把bot部署在project A、MySQL放在project B。

---

## 🟡 callbackQuery 里获取 chatId

grammy 的 callbackQuery 里 ctx.chat 有时是 undefined。

```js
// ❌ 错误
const chatId = ctx.chat.id;

// ✅ 正确
const chatId = ctx.callbackQuery.message?.chat?.id ?? ctx.chat?.id;
if (!chatId) return;
```

所有 bot.callbackQuery(...) 里都用这个方式取 chatId。

---

## 🟡 存库前清理不可序列化的字段

setTimeout 返回的 timer 不能被 JSON.stringify，存库前删掉：

```js
async function saveRecord(record) {
  const plain = { ...record };
  delete plain.anyTimeout;
  await repo.set(record.id, JSON.stringify(plain));
}
```

---

## 🟡 .env 文件安全

### 新项目初始化顺序（不能反）
```
1. git init
2. 建 .gitignore，写入 .env
3. 才能建 .env 填入真实 key
```

.env 一旦被 git add 过，gitignore 就失效了。
如果已经泄露：立刻轮换所有 key，删仓库重建。

### push 前必须确认
```powershell
git ls-files | Select-String ".env"
```
有输出就不能 push，先执行 git rm --cached .env。

---

## 🟡 第三方 API 模型 ID

OpenRouter 的模型 ID 会过期或改名，使用前先确认：
- `google/gemini-2.0-flash-exp` 已下线
- 当前可用：`google/gemini-2.0-flash-001` 或 `google/gemini-2.5-flash`

新项目开始前去 openrouter.ai/models 确认模型 ID 还有效。

---

## 🟡 Node 环境变量加载

Node 不会自动读取 .env，必须用 dotenv：

```bash
npm install dotenv
```

```js
// bot.js 第一行
import 'dotenv/config';
```

---

## 🟢 部署约定

### Railway 环境变量（必须设置）
```
BOT_TOKEN=
DB_HOST=        # Railway内网：mysql.railway.internal
DB_PORT=        # Railway内网：3306
DB_NAME=
DB_USER=root
DB_PASSWORD=
```

### 只能有一个 bot 实例运行
本地开发时不要运行 node bot.js，会和 Railway 实例冲突报 409 Conflict。

检查本地（Windows）：
```powershell
Get-Process node | ForEach-Object { (Get-WmiObject Win32_Process -Filter "ProcessId=$($_.Id)").CommandLine }
```
Cursor 自己的 tsserver 进程不用管。

### 群里触发命令
群里命令必须带 bot 用户名，否则无响应：
```
/newgame@YourBotUsername
```
私聊直接 /newgame 即可。

---

## 🟢 文件结构

```
your-bot/
├── .cursorrules          ← 项目专用规则
├── CLAUDE.md             ← 本文件（通用规则）
├── .env                  ← 本地环境变量（不提交git）
├── .gitignore            ← 必须包含 .env
├── package.json
├── bot.js
├── src/
│   ├── handlers/         ← 命令和回调处理
│   ├── core/             ← 业务核心逻辑
│   ├── db/               ← 数据库层
│   └── messages/         ← 消息文本构建
└── docs/
    ├── prd.md
    ├── tech-design.md
    ├── task.md
    └── changelog.md
```

---

## 🟢 调试原则

1. 先加日志，再猜原因：在关键路径加 console.log，看实际值
2. 每次只改一处：同时改多处出了 bug 不知道是哪里导致的
3. commit 前问 AI：「这个改动有没有影响其他模块？」

---

## 🟢 Git 规范

```
feat: 新功能
fix: bug 修复
refactor: 重构（不改功能）
debug: 临时调试（上线前必须删掉）
```

每个 task 完成测试通过后立即 commit。新功能用新分支，出问题直接回主分支。

---

## 📋 每次新对话的上下文模板

```
我在开发一个 Telegram bot，技术栈：grammy + mysql2 + Railway。
通用规则：[粘贴本文件]
项目专用规则：[粘贴 .cursorrules]
当前 task：[描述]
相关代码：[粘贴]
```
