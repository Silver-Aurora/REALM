# 酒馆格式角色卡与世界书导入实施规范

> 文档性质：实施规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。

## 1. 支持格式清单

| 输入 | 识别方式 | 说明 |
|---|---|---|
| PNG 角色卡（chara 块） | PNG 签名 + 逐 chunk 扫描，`tEXt` keyword=`chara`，base64→UTF-8 JSON | 卡本体即头像，存入 world_files |
| PNG 角色卡（ccv3 块） | 同上，keyword=`ccv3`（chara 缺失时回退） | V3 结构按 V2 data 读取 |
| JSON 角色卡 V2 | `spec = "chara_card_v2"`，读 `data.*` | — |
| JSON 角色卡 V1（legacy） | 无 spec 但有 `name` + `description` | 平铺字段 |
| 独立世界书导出 | JSON 含 `entries`（索引对象或数组） | — |
| V2 卡内嵌世界书 | `data.character_book.entries` | 随卡同一事务导入 |

## 2. 字段映射

| 酒馆字段 | 落点 |
|---|---|
| name | character_definitions.display_name / profile.name |
| description / personality / scenario / first_mes / mes_example / creator_notes / system_prompt / post_history_instructions / alternate_greetings / tags / creator / character_version | profile 同名键（完整归一化保留） |
| PNG 本体 | world_files（kind=character_avatar），profile.avatar_file_id 引用 |
| 世界书条目 name/keys[0]/序号 | world_articles.title（name → 首个 key → 「条目 N」） |
| 世界书条目 content | world_articles.body |
| 世界书条目 enabled=false | 跳过并列入报告 skippedDisabled |

## 3. 已知损耗（v1 丢弃）

`priority` / `insertion_order` / `case_sensitive` / `keys`（除用于标题外）/ `secondary_keys` / 深度与位置指令（@D 等）/ 卡内图片以外的扩展资产。角色不会被自动装配进既有记录阵容——导入只落定义，阵容在新建记录时按自定义角色装配规则进入。

## 4. 错误处理（fail-closed）

无法识别的格式、损坏的 base64、非图片字节、JSON 解析失败、必填字段缺失、超 10MB——一律返回带原因的类型化错误（`TAVERN_IMPORT_*`），不写任何数据。导入事务：角色 + 头像文件 + 世界书条目原子提交，任一失败整体回滚。允许重复导入（每次生成新 id，不去重——用户已拍板）。

## 5. 文件校验

magic bytes 验证 PNG（`89 50 4E 47`）/ JPEG（`FF D8 FF`）/ WebP（`RIFF....WEBP`），不信扩展名与声明的 content-type；单文件上限 10MB。

## 6. 存储层（world_files，迁移 0016）

Postgres `bytea`：与运行时契约一致（单库单事务）、导入与元数据同事务提交、`pg_dump` 即备份、无文件系统孤儿文件风险。列：workspace_id/world_id/id/kind/content_type/filename/sha256/size_bytes/data/created_at；realm_runtime 授 SELECT, INSERT；RLS 沿用 workspace 隔离策略。

## 7. API 契约

- `POST /api/library/import`（multipart）：`file` + `worldId` → `{ character?: {id,name,avatarFileId}, articlesCreated: string[], skippedDisabled: string[], warnings: string[] }`；错误 `{ ok:false, error:{code,message} }`。
- `GET /api/files/[id]`：会话鉴权 + workspace 校验；按 content_type 输出，`Cache-Control: public, max-age=31536000, immutable`；跨 workspace/无权限 403，不存在 404。

## 8. 前端

世界库面板加「导入」入口：文件选择（.png/.json）→ 上传 → 报告（角色+头像预览、条目数、跳过名单、警告）。角色行有 avatar_file_id 时经 /api/files/[id] 渲染头像。固定文案走批次 Q 的 i18n key（三语对齐）；导入内容原样展示，不模板化不翻译。
