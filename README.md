# 演唱会座位登记 · Netlify 部署版

静态前端 + 单个 Serverless 接口函数。数据默认存 **Netlify Blobs**（Netlify 官方持久存储，绑定站点即可用，不需要额外注册服务）。

## 目录结构（这些文件必须放在仓库根目录）

```
netlify.toml              Netlify 配置：publish=public，/api/* → 函数
netlify/functions/api.js  Netlify 函数入口，调用 server.js 的 handleApi
server.js                 全部业务接口 + 静态文件服务（本地可直接 node server.js）
public/                   前端页面（index.html / app.js / style.css / seatdata.js）
package.json              依赖（@netlify/blobs）与启动脚本
vercel.json              （备用）Vercel 部署配置
api/index.js             （备用）Vercel 函数入口
```

## 部署步骤（Netlify）

1. **传代码**：把 `web/` 里所有文件放到 GitHub 仓库根目录（不要套一层 `web/`）。
2. **导入站点**：Netlify → Add new site → Import an existing project → 选 GitHub 仓库。
   Build command / Publish directory 会自动读 `netlify.toml`，无需手填。
3. **确认持久存储**：Site configuration → **Blobs**（部分账号在 Build & deploy 下）→ 确认 Blobs 已启用。
   Netlify 运行时会自动注入存储所需的上下文，代码直接用，不用配密钥。
4. **部署**：Deploy site。拿到 `xxx.netlify.app` 域名即可访问。
5. **管理员入口**：登录页右下角 👤。首次进入填昵称 + 秘钥（可自定义或沿用系统生成的）；
   想固定成自己的秘钥，就在 `server.js` 的 `blankData()` 里预置 `adminKey` 后再部署。

## 存储策略（按优先级自动选择）

| 模式 | 触发条件 | 适用 |
| --- | --- | --- |
| Netlify Blobs | 检测到 `NETLIFY` 环境变量，或手动设 `STORE=blob` | Netlify（默认） |
| Upstash Redis | 设了 `UPSTASH_REDIS_REST_URL` + `UPSTASH_REDIS_REST_TOKEN`（或 `KV_REST_API_*`） | Vercel / 其它平台 |
| 本地 data.json | 以上都没有 | `node server.js` 本地开发 |

一旦用了远程存储，**不要再提交 data.json**（已在 .gitignore 里）。

## 环境变量（Netlify）

| 变量 | 必填 | 说明 |
| --- | --- | --- |
| `STORE` | 可选 | 填 `blob` 强制用 Blobs，填 `file` 强制用本地文件 |
| `BLOB_STORE` | 可选 | Blobs 存储桶名，默认 `concert-seat` |
| `DATA_KEY` | 可选 | 数据键名，默认 `seat-data.json`（Redis 模式下默认 `seat:data`） |
| `UPSTASH_REDIS_REST_URL` / `TOKEN` | 可选 | 改用 Redis 时填 |

## 数据说明

- 全部数据（管理员秘钥、座位占用、会话、题库、反馈）是一个 JSON 对象，整体读写。
- 写请求会加分布式锁（Blobs 用条件写入，Redis 用 `SET NX PX`），避免并发重复占座。
- 想清空：管理员后台「清空全部数据」；或删掉 Blobs 里的数据键。
- Blobs 单值上限较大，但反馈截图是 base64，量大了会拖慢读写——建议定期清理已处理反馈。

## 本地运行

```bash
npm install      # 只为 @netlify/blobs，本地可选
node server.js   # 打开 http://localhost:3000
```

手机上访问：与电脑同一 Wi-Fi，打开启动日志里打印的局域网地址。

## 部署前自检

- [ ] 仓库根目录能直接看到 `netlify.toml`、`server.js`、`public/`
- [ ] `data.json` 没有被提交（里面有你的管理员秘钥）
- [ ] Netlify 构建日志里 `npm install` 成功装上了 `@netlify/blobs`
- [ ] 打开站点后，登录 → 答题 → 选座能走通
- [ ] 换一个浏览器/设备访问，刚才的座位仍是「已占用」（说明持久存储生效）
