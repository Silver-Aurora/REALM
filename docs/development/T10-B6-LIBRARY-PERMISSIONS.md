# T10-B6 · Library 非管理命令权限与 Runtime Pool 加固

> 批次 T10-B6（docs/development/EXPERIENCE-ITERATION.md T10 拆批；T8 §2.4 遗留「其余命令权限另案」+ 路由 owner pool 绕过受限角色问题）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=bcbcf2c；不改迁移 0001–0022、不新增迁移、不改 LM Studio/T6–T10-B5；T8 archive/delete owner-only 语义不动。

## 一、现状实锤（读代码 + 实查授权，勿重复调查）

1. **list 不过滤**：`createPostgresLibraryService.list()` 的 worlds WHERE 只有
   workspace——任何登录 principal 看见全部世界；membershipRole 缺行时回落
   'player'（library-service.ts:235-245）。
2. **写命令零 membership 校验**：world-style 直接 UPDATE；character/branch/
   story/record/attach 只查世界存在；player-stance 只更新本人行但不验存在性。
3. **路由用 owner pool**：`app/api/library/route.ts:249-263` 用
   `process.env.DATABASE_URL` + 每服务 `createLocalPostgresPool`，绕过
   realm_runtime 受限权限面。
4. **授权面实查**（realm_dev information_schema 当次实测）：realm_runtime
   有 INSERT 的表仅 records/stories/worldlines（0022）+ 治理/记忆/台账类；
   **worlds / character_definitions / participants / record_heads 无 INSERT**，
   player_world_memberships/participants 无 UPDATE。结论：list 与
   world-style/story/branch/archive/delete 可在受限角色下运行；
   world/character/record/attach-character/player-stance/createGenesis 在受限
   角色下**物理不可写**——在「不新增迁移」约束下这些命令只能留在 owner
   pool（见 §三的分池契约与理由）。

## 二、权限矩阵（本批定稿，测试锚定）

| 命令 | 要求 | 非成员 | observer 成员 | 非 owner 成员 |
|---|---|---|---|---|
| `GET list` | 只返回有 membership 的世界（EXISTS 过滤） | 不可见（不泄露） | 可见只读 | 可见 |
| `world` | 无目标；创建者同事务成 owner（既有） | — | — | — |
| `world-style` | **owner**（世界全局设置） | 403 WORLD_NOT_OWNED（与 T8 管理命令同口径，见下注） | 403 WORLD_NOT_OWNED | 403 WORLD_NOT_OWNED |
| `character` / `attach-character` / `branch` / `story` / `record` | membership role ∈ owner/player | 404 WORLD_NOT_FOUND | 403 WORLD_READ_ONLY（新稳定码） | 放行 |
| `player-stance` | 只能改本人 membership（body 不得指定 principal） | 404 WORLD_NOT_FOUND | 放行（observer↔player 是本人姿态） | 放行 |
| `world-archive` / `delete-world` | owner（T8 不动） | 404/403 沿用 T8 | 403 WORLD_NOT_OWNED | 403 WORLD_NOT_OWNED |
| `/api/library/import`（写内容等价物） | membership role ∈ owner/player | 404 形态 | 403 形态 | 放行 |

- 不泄露原则：内容命令（story/record/character/branch/attach）与 stance 的
  非成员一律 404（与 T9/T10-B1 同口径）；**owner-only 管理命令例外沿用 T8
  口径——非成员 403 WORLD_NOT_OWNED**（T8 已锁定该形态，本批不改动 T8 语义）。
- 归档世界叠加既有 WORLD_ARCHIVED 409（T8 语义优先于本矩阵的角色判定之外
  不变；判定顺序：存在性 → 角色 → 归档）。
- observer 403 与 404 的取舍：observer 是合法成员（知道世界存在），用显式
  403 WORLD_READ_ONLY 提供可纠正的信号；非成员 404 不泄露。

## 三、分池契约（授权缺口的工程裁决）

- **runtime pool（REALM_RUNTIME_DATABASE_URL + getSharedRuntimePool）**：
  list、world-style、story、branch、world-archive、delete-world——这些命令的
  全部读写在 realm_runtime 授权面内（实测见 §一.4）。
- **owner pool（DATABASE_URL，唯一例外）**：world、character、record、
  attach-character、player-stance、createGenesis、import——它们的事务需要
  worlds/character_definitions/participants/record_heads INSERT 或
  memberships/participants UPDATE，受限角色无授权；「不新增迁移」约束下
  无法下沉。每个例外命令在路由常量中枚举，静态契约锚定。
- 缺 REALM_RUNTIME_DATABASE_URL → 受限命令 503；缺 DATABASE_URL → owner
  命令 503；安全文案不泄露连接信息。
- 静态契约（api-core-wiring 扩展）：library 路由必须经
  `getSharedRuntimePool` 走 runtime URL；owner pool 只允许出现在枚举的
  例外命令路径。
- 后续批次（候选 T10-B7）：一份把上述例外命令所需最小授权补齐的迁移评审
  （worlds/character_definitions/participants/record_heads INSERT 列级 +
  memberships/participants 受限 UPDATE），评审通过前 owner 例外不扩大。

## 四、实现要点

- service：`createPostgresLibraryService(readDb, writeDb?)`——readDb 承载
  list 与受限命令事务；writeDb（缺省=readDb）承载 owner 例外命令；每个命令
  事务仍在单一池内原子完成。
- 共享守卫（同一事务内）：
  `resolveWorldRole(client, scope, worldId)` → 'owner'|'player'|'observer'|null；
  null→404、observer 写内容→403 WORLD_READ_ONLY、owner-only→403
  WORLD_NOT_OWNED；worldId/storyId/recordId 交叉校验沿用既有；
  body 永不含 workspace/principal（路由层不给）。
- list SQL：`AND EXISTS (SELECT 1 FROM player_world_memberships m
  WHERE m.workspace_id=world.workspace_id AND m.world_id=world.id
  AND m.principal_id=$2)`；membershipRole 列已有同键 join，不再回落假
  'player'（无 membership 的世界已被过滤）。
- import 路由同矩阵（内容写等价物）：member 且 role∈owner/player 放行，
  其余 404/403 形态。

## 五、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | 非成员读 list | 世界不出现（不泄露） |
| F2 | 非成员写任何目标世界 | 404 WORLD_NOT_FOUND |
| F3 | observer 写内容命令 | 403 WORLD_READ_ONLY |
| F4 | 非 owner 执行 world-style/archive/delete | 403 WORLD_NOT_OWNED |
| F5 | player-stance 无 membership | 404（不创建席位） |
| F6 | 缺 runtime/owner URL | 503 安全文案 |
| F7 | 归档世界内容写 | 409 WORLD_ARCHIVED（T8 不动） |

## 六、验收标准

1. 临时 PG（t.after 拆库）：list 成员可见/非成员不见；矩阵逐格（owner/
   player/observer/非成员 × style/content/stance/archive/delete）；归档门禁
   与 T8 语义不回归；import 非成员 404。
2. 受限角色实锤：runtime URL 池完成 list/style/story/branch/archive/delete；
   owner URL 不设置时例外命令 503；route 无 DATABASE_URL 时受限命令仍可用。
3. 既有 postgres-library-service 回归不删不改语义（owner 建库/runtime 执行
   拆分）；api-core-wiring 静态契约过。
4. typecheck/受影响 eslint/文档布局/git diff --check exit 0；不跑全量/GUI。
5. 开发库基线不变。

## 七、交付步骤

1. 本规范（单独 commit）；2. service/路由/围栏/测试（独立 commit）；
3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
