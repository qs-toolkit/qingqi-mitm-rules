# 轻启 MITM 规则仓库协议 v1

GitHub 是规则、身份公钥、评论和评分的唯一权威。VPS 的 HTTPS API 和本地镜像对应同一个已校验的提交；上传完成会写回 GitHub，外部渠道提交也可通过签名 webhook 同步。webhook 未送达时每 60 秒核对仓库 HEAD。同步失败保留上一有效快照，`/health` 与 `/catalog` 返回同步状态。

默认入口：`https://176.122.145.69:8443/mitm-store/v1`。

## 身份与版本

- `identityId`：P-256 公钥 DER SPKI 字节的 SHA-256；无需注册或登录。私钥在客户端安全存储，绝不发送到服务器或仓库。
- `listingId`：商店条目 ID。作者更新时保持不变。作者由签名密钥证明，不依赖昵称或规则名称。
- `revisionId`：规则包 JSON 原始 UTF-8 字节的 SHA-256。内容文件不可变，更新追加版本。
- 本机安装记录另有稳定 `installationId`，保存编辑后直接替换当前版本，不并装原版与修改版。

## 文件布局

```text
catalog/index.json                     条目摘要数组，不含 revisions 历史数组
listings/<listingId>.json               完整条目与 revisions 历史数组
rules/<revisionId>.json                 规则包 JSON，文件名必须与正文摘要一致
identities/<identityId>.json            publicKey（SPKI base64）、nickname、identityId
reviews/<listingId>/<identityId>.json    此身份的当前评分与评论
reviews/<listingId>/index.json          当前评论记录数组，与单份记录一致
```

一个条目含 `listingId`、`ownerIdentityId`、`revisionId`、`revisions`、`name`、`description`、`scope`、`author`、`updatedAt`、`ratingSum`、`ratingCount`、`parentListingId`。`scope: []` 代表全局应用，非空数组为应用包名列表。评分为 1–5 星，同一身份的新评价覆盖自己的旧评分。

规则包的基本格式：

```json
{
  "schema": 1,
  "name": "规则名称",
  "description": "规则说明",
  "scope": ["com.example.app"],
  "text": "[Rule]\nDOMAIN,ads.example.invalid,REJECT",
  "scriptResources": [],
  "dataResources": [],
  "scriptsEnabled": false,
  "scriptScope": "*",
  "argumentsJson": "{}",
  "jsonPackage": ""
}
```

脚本资源含 `url` 和实际 `code`，数据资源含 `url`、`kind`、实际 `content`；可携带 `revision`、`updatedAt`。只读缓存地址不能替代完整正文。`localScript` 是旧格式兼容字段，含 host、bundleName、pattern、code、revision、url。`jsonPackage` 保留旧 JSON 包与任务配置；不因界面重构丢弃已有任务。

## HTTPS API

| 方法 | 路径 | 作用 |
| --- | --- | --- |
| GET | `/health` | 服务与镜像状态 |
| GET | `/catalog` | 当前提交、目录、同步状态 |
| GET | `/rules/<listingId>` | 当前条目及历史版本 |
| GET | `/content/<revisionId>` | `revisionId` 与原始 `contentJson`；客户端必须复核摘要 |
| GET | `/rules/<listingId>/reviews?offset=0` | 评价分页，每页 20 条 |
| GET | `/challenge?identity=<identityId>` | 获取限时、一次性挑战 |
| POST | `/rules` | 首次上传或作者更新 |
| POST | `/reviews` | 新建或修改本人评价 |
| POST | `/hooks/github` | GitHub push / ping，验证独立 webhook HMAC |

`POST /rules` 正文含 `nickname`、`listingId`、`expectedRevision`、`parentListingId`、`contentJson`。首次上传 `expectedRevision` 为空；更新必须为仓库当前版本。作者不匹配返回 403，版本竞争返回 409。重复上传同一版本不会增加条目。其他身份修改后需创建自己的条目并保留 `parentListingId`。

`POST /reviews` 正文含 `nickname`、`listingId`、`revisionId`、`rating`、`comment`。评价版本必须存在于该条目的版本历史中。

## 客户端请求签名

获取挑战后，将下面数组用紧凑 JSON（无额外空格）编码为 UTF-8，再用 ECDSA P-256 + SHA-256 签名，签名格式为 DER：

```text
[1, "POST", "/mitm-store/v1/rules", nonce, timestampMilliseconds, sha256(rawRequestBody)]
```

`X-QS-Auth` 请求头为以下对象的 JSON UTF-8 再 base64：

```text
{publicKey: SPKI_base64, nonce: challenge, timestamp: milliseconds, signature: DER_signature_base64}
```

路径必须包含完整 API 前缀。VPS 验证签名、时效和一次性挑战；不相信客户端单独提交的作者 ID。昵称可修改，不影响身份。客户端可以提供密码加密的密钥备份与恢复；遗失密钥且无备份时，不能仅凭昵称找回身份。

## 外部渠道写仓

可使用同一签名 API，也可由有权写仓的维护者通过 Git 提交文件。直接写仓时，正文、条目、目录、身份、评价记录和评分汇总必须在同一次提交中一致更新；不可覆盖已有 revision 文件。写仓权限属于仓库信任边界，外部发布工具应保持作者身份和派生关系。

GitHub push webhook 启用 SSL 校验和 HMAC-SHA256 验证，匹配目标仓库与分支。VPS 收到事件后重新核对 GitHub HEAD，不直接信任请求里提供的提交值。新镜像逐项验证摘要、目录对应关系、作者公钥和评分汇总后，原子切换当前快照，避免目录/正文混用版本。

仓库不应包含凭据、用户私钥、应用源码或构建符号。VPS 的 GitHub 写入凭据与 webhook secret 由独立服务凭据管理，API 请求路径不记录签名头或正文。
