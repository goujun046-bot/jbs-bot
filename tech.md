# 剧本杀 Telegram Bot - Tech Design

**版本**: v2.1  
**日期**: 2026-03-27  
**关联文档**: `PRD-v2.md`  
**状态**: 可开发草案

---

## 1. 技术选型及理由

| 维度 | 选型 | 理由 |
|---|---|---|
| 运行时 | Node.js >= 18 | 与部署平台兼容，生态成熟 |
| Bot 框架 | `grammy` | Telegram Bot 场景成熟，Middleware 组织清晰 |
| 数据库 | MySQL 8 | 关系数据 + 事务，适合状态机推进 |
| DB SDK | `mysql2/promise`（连接池） | 官方推荐，简单稳定 |
| 配置 | `dotenv` | 显式加载 `.env`，避免环境变量缺失 |
| 部署 | Railway | 与项目约定一致，运维成本低 |

关键约束:
- 剧本与局内结构化数据统一存 `LONGTEXT`，手动 `JSON.stringify/parse`。  
- 不引入 Redis，MVP 并发规模下直接使用 MySQL + 内存定时器。  
- 不在代码硬编码密钥，全部读取环境变量。

---

## 2. 项目文件结构

```text
jbs-bot/
├── bot.js
├── src/
│   ├── handlers/           # 指令与 callback 入口
│   ├── core/               # 状态机与业务规则
│   ├── db/                 # 数据访问与 SQL
│   ├── messages/           # 文案模板
│   ├── utils/              # 校验、时间、日志工具
│   └── types/              # JSDoc / TS 类型声明（可选）
├── docs/
│   ├── prd.md
│   ├── tech-design.md
│   ├── task.md
│   └── changelog.md
├── .env
├── .gitignore
└── package.json
```

分层职责:
- `handlers`: 只做入参提取与权限拦截，不写复杂业务。  
- `core`: 统一管理开局、发身份、投票、结算状态推进。  
- `db`: 提供最小数据接口，避免业务层直接写 SQL。

---

## 3. 数据模型（数据库表结构）

### 3.1 状态模型

- 房间状态: `WAITING -> RUNNING -> VOTING -> ENDED`  
- 不允许跨状态跳转；每次跳转先校验阶段和权限。

### 3.2 DDL（MVP）

```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  tg_user_id BIGINT NOT NULL UNIQUE,
  tg_username VARCHAR(64) NULL,
  display_name VARCHAR(128) NOT NULL,
  private_chat_id BIGINT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE scripts (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  title VARCHAR(128) NOT NULL,
  summary TEXT NOT NULL,
  player_count TINYINT NOT NULL,
  duration_min SMALLINT NOT NULL,
  data LONGTEXT NOT NULL, -- JSON string
  uploader_tg_user_id BIGINT NOT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE rooms (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  invite_code CHAR(6) NOT NULL UNIQUE,
  owner_tg_user_id BIGINT NOT NULL,
  group_chat_id BIGINT NOT NULL,
  script_id BIGINT NOT NULL,
  status VARCHAR(16) NOT NULL, -- WAITING/RUNNING/VOTING/ENDED
  discuss_end_at DATETIME NULL,
  vote_end_at DATETIME NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  ended_at DATETIME NULL
);

CREATE TABLE room_players (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  room_id BIGINT NOT NULL,
  tg_user_id BIGINT NOT NULL,
  role_id VARCHAR(64) NULL,
  role_name VARCHAR(128) NULL,
  joined_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uq_room_user (room_id, tg_user_id)
);

CREATE TABLE votes (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  room_id BIGINT NOT NULL,
  voter_tg_user_id BIGINT NOT NULL,
  target_role_id VARCHAR(64) NOT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uq_room_voter (room_id, voter_tg_user_id)
);
```

### 3.3 `scripts.data` JSON 约定

```json
{
  "roles": [
    {
      "id": "r1",
      "name": "角色A",
      "background": "背景",
      "goal": "目标",
      "isMurderer": false,
      "clues": [{"name": "线索1", "content": "内容"}]
    }
  ],
  "truth": {
    "murdererRoleId": "r2",
    "story": "真相全文"
  }
}
```

---

## 4. API 接口规范（路径/请求/响应/错误码）

> 说明: Telegram Bot 主要通过命令与 callback 触发；以下为内部服务接口与可扩展 HTTP 接口约定。

### 4.1 Bot 命令契约

- `/create` -> 创建房间（返回邀请码）  
- `/join <code>` -> 加入房间  
- `/start_game` -> 校验并开局  
- `/end_discuss` -> 结束讨论进入投票  
- `callback:vote:<roleId>` -> 投票

### 4.2 HTTP 预留（Mini App）

- `GET /api/scripts`  
- `GET /api/rooms/:inviteCode`  
- `POST /api/rooms/:id/vote`

### 4.3 统一响应格式

```json
{
  "ok": false,
  "code": "PHASE_INVALID",
  "message": "当前阶段不可执行该操作",
  "data": null
}
```

错误码最小集合:
- `ROOM_NOT_FOUND`
- `PERMISSION_DENIED`
- `PHASE_INVALID`
- `PLAYER_NOT_READY_PM`
- `DUPLICATE_VOTE`
- `INTERNAL_ERROR`

---

## 5. 校验流程

### 5.1 开局校验

1. 请求者是否房主。  
2. 房间状态是否 `WAITING`。  
3. 当前人数是否等于剧本人数。  
4. 全员是否具备 `private_chat_id`。  
5. 校验通过后事务内更新状态到 `RUNNING`。

### 5.2 投票校验

1. 房间状态必须是 `VOTING`。  
2. 投票人必须是本局玩家。  
3. `votes` 唯一键保证一人一票（幂等）。  
4. 每次写票后检查是否全员投完，满足则触发结算。

### 5.3 callbackQuery 取 `chatId` 规范

所有 `bot.callbackQuery(...)` 内统一:

```js
const chatId = ctx.callbackQuery.message?.chat?.id ?? ctx.chat?.id;
if (!chatId) return;
```

---

## 6. 第三方 API 调用规范

### 6.1 Telegram Bot API

- 所有 callback 必须 `answerCallbackQuery`，避免客户端 loading 卡住。  
- 私密内容只发私聊，群消息禁止带身份信息。  
- 失败重试最多 2 次，仍失败则告警并回滚状态。

### 6.2 OpenRouter（如后续启用）

- 模型 ID 仅使用在线可用型号（如 `google/gemini-2.5-flash`）。  
- 超时与失败必须可降级（返回默认文案，不阻塞主流程）。  
- 请求日志脱敏，禁止输出 token 与用户隐私内容。

---

## 7. 环境变量清单

```bash
BOT_TOKEN=
DB_HOST=
DB_PORT=
DB_NAME=
DB_USER=root
DB_PASSWORD=
NODE_ENV=production
LOG_LEVEL=info
```

说明:
- 本地开发使用 Railway 公网数据库地址。  
- Railway 内部部署使用 `mysql.railway.internal:3306`。

---

## 8. 部署配置

### 8.1 Railway 部署原则

- Bot 与 MySQL 必须在同一个 Railway Project。  
- 只允许一个 Bot 实例运行，避免 Telegram 409 Conflict。  
- 发布前检查 `.env` 未被跟踪:

```powershell
git ls-files | Select-String ".env"
```

### 8.2 启动检查清单

1. 环境变量是否完整。  
2. 数据库连通性是否正常。  
3. Bot 是否成功拉起并收到 webhook/polling 更新。  
4. 可否完整跑通一局（开局 -> 讨论 -> 投票 -> 结算）。

---

## 9. 风险预审（这个技术组合的已知坑）

必问问题:
- Railway 数据库内网/公网地址如何区分，是否按环境正确配置？  
- Mini App 的 `initData` 签名验证在哪里做、失败如何拒绝？  
- CORS 如何配置（允许域名、方法、凭证策略）？  
- 并发场景下如何避免重复开局、重复投票、重复结算？  
- 第三方 API 超时或限流时的降级策略是什么？

风险与对策:
- **重复提交**: 通过唯一键 + 事务 + 幂等返回兜底。  
- **状态错乱**: 所有状态切换走同一服务函数，禁止分散更新。  
- **私聊失败**: 进入阻塞态并给房主明确恢复路径。  
- **进程重启**: 启动后扫描未结束房间并恢复定时任务。  
- **密钥泄露**: `.gitignore` 严格拦截 `.env`，推送前固定检查。