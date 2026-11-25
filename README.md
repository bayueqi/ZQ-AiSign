# ZQ-AiSign

一个 Cloudflare Worker，同时管理四个平台的每日签到：Trae、WorkBuddy、Qoder、CodeArts。一套部署、一个控制台、一条定时任务，账号数据存同一个 KV，按平台前缀隔离。

> 作者：bayueqi

## 支持的平台

| 平台 | 授权方式 | 签到内容 |
|------|----------|----------|
| Trae | 抖音扫码授权 | 每日积分 |
| WorkBuddy | 微信扫码授权 | 每日签到 |
| Qoder | 设备授权（PKCE） | 每日 credits 领取 |
| CodeArts | 华为云 PKCE + DPoP 授权 | 每日积分领取（SDK-HMAC-SHA256 签名） |

## 部署

1. 在 Cloudflare 控制台新建一个 Worker，把 `worker.js` 的内容整个贴进去（或走 Wrangler 部署）。
2. 绑定一个 KV 命名空间，变量名填 `KV`。
3. 配置 Secret：
   - `MASTER_KEY`：32 字节随机值的 base64（四个平台共用）
   - `ADMIN_TOKEN`：控制台登录密码
   - `CRON_SECRET`：（可选）`/cron` 独立口令，缺省回落到 `ADMIN_TOKEN`
4. （可选）配置定时触发器 Cron，UTC 时间（比北京时间慢 8 小时），例如：
   - 北京 08:10 = UTC 00:10 → `10 0 * * *`

## 路由

- `/`：控制台（四个标签页）
- `/cron`：定时入口，四个平台依次签到
- `/api/ping`：密码校验
- `/api/run-all`：四个平台一起签
- `/trae/api/*`、`/wb/api/*`、`/qoder/api/*`、`/codearts/api/*`：各平台管理接口
- `/oauth/callback`：Trae 扫码回调

## 说明

- 签到幂等，今天签过会自动跳过。
- 各平台令牌用 AES-256-GCM 加密存 KV，密钥由 `MASTER_KEY` 经 HKDF 派生。
- CodeArts 授权走华为云官方 PKCE + DPoP 流程，设备私钥由 Worker 现场生成并加密保存，授权回调落在本机 `127.0.0.1`，采用「粘贴回调链接」方式换取令牌。
