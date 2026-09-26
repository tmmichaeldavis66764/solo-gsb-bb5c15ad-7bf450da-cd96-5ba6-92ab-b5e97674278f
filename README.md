# 实验室混样批次追溯 API

基于 **FastAPI + SQLite** 的实验室混样（pooling）批次追溯服务：登记原始样本、按配方创建混样批次，并对任一批次递归追溯其中各原始样本的精确占比（约分后的分子/分母，无浮点误差）。

## 功能特性

- 登记**原始样本**、按配方创建**混样批次**（配方 = 已有批次 + 各自的正整数投入份数）
- **原始样本禁配规则**：登记一对不同的原始样本（无序对，重复登记幂等）；创建混样时沿配方图递归展开**嵌套混样与多路径汇合**后的全部可达原始样本，命中任一禁配对即返回 **409**（不创建批次或配方）；规则只约束**随后创建**的批次，已有批次及其占比接口不受影响
- **禁配补录影响预演**：给定两个不同的原始样本（允许尚未登记为禁配对），沿现有批次配方 DAG 计算可达性，返回**已同时含两样本的全部混样批次**；直接原料均不同时含两样本的批次标记为**首次合流**，并为每个受影响批次列出沿配方可追溯的首次合流批次 id（嵌套与多路径汇合不漏报、不重复）；**只读，不登记规则、不改已有配方**
- 创建校验：拒绝**空配方**（422）、**未知批次**（400）、**非正整数份数**（422，含 0、负数、小数、字符串）、**命中禁配规则**（409）
- 新批次只能引用**已创建**的批次，引用关系天然构成有向无环图，不可能成环
- 占比追溯：按每层“投入份数 / 该层总份数”递归分摊；同一原始样本经多条路径到达时**自动合并**；基于 `fractions.Fraction` 精确有理数运算，返回**约分后**的分子与分母
- **目标占比规划**：给定两个不同的已有批次、一个原始样本与目标占比，反算使新混样精确达标的**正整数投入份数**，只返回总份数最小的整数解（两原料占比都等于目标时取 1:1），配方可直接提交 `/pools`；**只规划、不创建批次**
- **配方变更影响预演**：提交完整替代配方，在内存中演算目标批次及全部下游批次各原始样本的原占比、预演占比与有符号差值，**不修改数据库**
- **份数诊断**：假设混样配方仅一项投入份数记错，按实测原始样本占比用有理数方程反推该项的正整数份数，并用全部样本占比复核；份数变化不改变组成且与实测吻合的项列为**无法判定**；**不修改数据库**
- **原始样本库存与批次制备**：可为已有原始样本补入**正整数份**库存并查询余额；`POST /batches/{id}/preparations` 按**正整数次数**制备批次，每制备一次混样按其配方消耗相应份数的上游批次，沿配方图逐层乘算、多路径需求合并为各原始样本的**整数消耗**（原始样本目标直接消耗自身）；核算与扣减在**同一 SQLite 写事务**内完成，任一样本库存不足即返回 **409** 与各样缺口、库存与制备记录均不变化，足量时一次扣减并保存目标批次、次数与消耗明细，可按记录 id 查询
- **整单制备**：`POST /preparation-jobs` 一次接收**非空制备清单**（每项为已有批次 id + 正整数制备次数），同一批次重复出现时先**合并次数**；先沿配方图计算每项对原始样本的整数需求，再**按样本汇总整单需求**统一核算（共享样本不会被各项分别判为足量）；读取余额、核算缺口、扣减库存与写入各批次制备记录在**同一写事务**完成，任一样本不足返回包含该样本**总需求、余额与缺口**的 **409**、整单不留记录且余额不变；成功时返回各批次制备记录 id 与整单消耗，每条记录仍可由 `GET /preparations/{id}` 读取
- **只读排产预演**：`POST /preparation-jobs/selection-preview` 接收 **1～12 条候选制备单**（每条为已有批次 id + 正整数次数，按请求位置标识订单，**允许重复批次**、不合并次数），复用配方图整数消耗展开计算每条订单对原始样本的需求，依据**同一库存快照**选出各样本总需求均不超过余额的子集——先**最大化入选订单数**，再**最大化入选制备次数之和**，仍并列时取**入选位置序列字典序最小**者；返回入选/未入选位置、按样本汇总的消耗与剩余库存，库存全不足时返回空集；**不扣减库存、不创建制备记录**
- **制备记录冲正**：`POST /preparations/{id}/reversal` 对**存在且尚未冲正**的制备记录，按记录**已保存的消耗明细逐样本返还**原始样本库存，并保存与该记录**一一对应**的冲正时间；读取记录、判重、返还与保存在**同一 SQLite 写事务**内完成；**重复冲正幂等**返回同一冲正记录（200）且不再次返还，未知制备记录返回 **404**。`GET /preparations/{id}/reversal` 查询冲正时间、各样本返还份数与当前库存（明细按样本 id 升序），未冲正返回 **404**。单批制备与整单制备生成的记录均可冲正；冲正**不删改**原制备记录与消耗明细，`GET /preparations/{id}` 照常可查
- **原始样本质量封存**：可对已有**原始样本**封存（`POST /batches/{id}/seal`）与解除封存（`POST /batches/{id}/unseal`），每次状态变更都必须给出**原因**并记录**时间**，`GET /batches/{id}/seal` 查询当前状态与**全部状态变更历史**；仅原始样本可封存（混样批次返回 **400**），**同一状态重复操作返回 409**（不新增流水）。单批制备与整单制备在同一 SQLite 写事务内沿**现有配方图**（含嵌套混样、多路径汇合）识别本次会消耗的全部上游原始样本，任一处于封存中即返回 **409** 与按样本 id 升序的**封存样本清单**，**整次请求不扣库存、不写制备记录**（封存检查先于库存缺口检查）；**解除封存后可再次制备**。封存状态变更与制备核算共用 `BEGIN IMMEDIATE` 写事务串行化，并发封存不会被制备漏拦；封存不影响**历史制备记录**的查询与冲正
- **原始样本库存盘点**：`POST /inventory-counts/preview` 接收**非空且样本 id 不重复**的盘点项（样本 id + 实盘**非负整数**份数），按**同一库存快照**返回各项账面数（**无库存行按 0**）与差额，**只读不写库**；`POST /inventory-counts` 接收盘点项、预演账面数与**非空白原因**，在**同一 SQLite 写事务**内核对全部样本**仍为原始样本**且余额**与预演账面数一致**——任一变化返回 **409** 与冲突项、**整单不改库存、不留记录**，全部一致时按实盘数更新余额（无库存行的样本新建库存行）并保存各项**调整前后数、差额、原因与时间**；`GET /inventory-counts/{id}` 查询已提交的盘点明细
- 依赖安装与数据库建表全部纳入启动流程，无需手工初始化

## 快速开始（Docker）

唯一需要的一条命令：

```bash
docker compose up --build
```

启动后访问：

| 地址 | 说明 |
| --- | --- |
| <http://localhost:8000> | API 服务 |
| <http://localhost:8000/docs> | Swagger 交互式文档（可直接试调用） |
| <http://localhost:8000/redoc> | ReDoc 文档 |

数据保存在 Compose 命名卷 `lab-data`（容器内 `/data/lab.db`）中，删除重建容器数据不丢失；`docker compose down -v` 可连同数据一起清除。

## 配置

通过环境变量配置（`docker-compose.yml` 中已设置，可按需修改）：

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `DATABASE_PATH` | 容器内 `/data/lab.db`，本地运行默认 `data/lab.db` | SQLite 数据库文件路径，启动时自动建目录、建表 |

端口映射在 `docker-compose.yml` 的 `ports` 中调整（默认 `8000:8000`）。

## API 一览

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/samples` | 登记原始样本 |
| `POST` | `/exclusion-rules` | 登记原始样本禁配对（无序对，重复登记幂等） |
| `GET` | `/exclusion-rules` | 列出全部禁配规则（按样本对字典序） |
| `POST` | `/exclusion-rules/impact-preview` | 禁配补录影响预演：两样本首次合流与受影响批次（只读，不登记规则） |
| `POST` | `/pools` | 创建混样批次（命中禁配返回 409） |
| `POST` | `/pool-plans` | 按目标占比反算整数份数配方（只规划，不写库） |
| `GET` | `/batches` | 列出全部批次 |
| `GET` | `/batches/{id}` | 批次详情（混样含配方） |
| `GET` | `/batches/{id}/composition` | 递归追溯原始样本占比 |
| `POST` | `/batches/{id}/recipe-preview` | 配方变更影响预演（不写库） |
| `POST` | `/batches/{id}/portion-diagnosis` | 份数诊断：假设仅一项份数记错，反推正整数份数（不写库） |
| `POST` | `/batches/{id}/inventory` | 为原始样本补入正整数份库存（累加） |
| `GET` | `/batches/{id}/inventory` | 查询原始样本库存余额（未补入为 0） |
| `POST` | `/batches/{id}/seal` | 封存原始样本（记录原因与时间，重复封存 409） |
| `POST` | `/batches/{id}/unseal` | 解除封存原始样本（记录原因与时间，未封存时 409） |
| `GET` | `/batches/{id}/seal` | 查询封存状态与全部状态变更历史 |
| `POST` | `/batches/{id}/preparations` | 批次制备：按整数次数消耗原始样本库存（封存或库存不足均 409 且整体回滚） |
| `POST` | `/preparation-jobs` | 整单制备：多批次清单合并次数、按样本汇总需求一次扣减（封存或库存不足均 409 且整单回滚） |
| `POST` | `/preparation-jobs/selection-preview` | 只读排产预演：1～12 条候选制备单按同一库存快照选型（不扣库存、不写记录） |
| `GET` | `/preparations/{id}` | 按记录 id 查询制备记录（消耗明细 + 当前剩余库存） |
| `POST` | `/preparations/{id}/reversal` | 制备记录冲正：按已保存消耗明细逐样本返还库存（重复冲正幂等，未知记录 404） |
| `GET` | `/preparations/{id}/reversal` | 查询冲正时间、各样本返还份数及当前库存（未冲正 404） |
| `POST` | `/inventory-counts/preview` | 库存盘点预演：同一库存快照返回账面数与实盘差额（只读，不写库） |
| `POST` | `/inventory-counts` | 提交盘点：核对账面数一致后按实盘数更新余额（任一变化 409 且整单回滚） |
| `GET` | `/inventory-counts/{id}` | 查询已提交的盘点明细（调整前后数、差额、原因与时间） |

## 示例请求

### 1. 登记原始样本

```bash
curl -X POST http://localhost:8000/samples \
  -H 'Content-Type: application/json' \
  -d '{"name": "S1"}'
# {"id":1,"name":"S1","kind":"sample","created_at":"...","recipe":null}

curl -X POST http://localhost:8000/samples \
  -H 'Content-Type: application/json' \
  -d '{"name": "S2"}'
# {"id":2,"name":"S2","kind":"sample",...}
```

### 2. 创建混样批次 P1 = 1×S1 + 1×S2

```bash
curl -X POST http://localhost:8000/pools \
  -H 'Content-Type: application/json' \
  -d '{"name": "P1", "recipe": [{"batch_id": 1, "portions": 1}, {"batch_id": 2, "portions": 1}]}'
```

```json
{
  "id": 3,
  "name": "P1",
  "kind": "pool",
  "created_at": "2026-09-24T12:00:00.000Z",
  "recipe": [
    {"batch_id": 1, "batch_name": "S1", "portions": 1},
    {"batch_id": 2, "batch_name": "S2", "portions": 1}
  ]
}
```

### 3. 创建二级混样 P2 = 1×P1 + 1×S1

```bash
curl -X POST http://localhost:8000/pools \
  -H 'Content-Type: application/json' \
  -d '{"name": "P2", "recipe": [{"batch_id": 3, "portions": 1}, {"batch_id": 1, "portions": 1}]}'
```

### 4. 追溯 P2 的原始样本占比

```bash
curl http://localhost:8000/batches/4/composition
```

```json
{
  "batch_id": 4,
  "batch_name": "P2",
  "kind": "pool",
  "components": [
    {"sample_id": 1, "sample_name": "S1", "numerator": 3, "denominator": 4},
    {"sample_id": 2, "sample_name": "S2", "numerator": 1, "denominator": 4}
  ]
}
```

S1 经两条路径进入 P2（直接投入 1/2，加上经 P1 的 1/2×1/2），合并为 3/4；各占比之和恒为 1。

### 5. 预演 P1 配方变更的影响

提交完整替代配方（沿用创建时的校验：严格正整数份数、重复原料合并、未知批次拒绝），在内存中把 P1 的配方替换为 `3×S1 + 1×S2`，返回 P1 及其全部下游批次（此处为 P2）各原始样本的原占比、预演占比与有符号差值（`delta = preview − original`），均为约分后的分子/分母：

```bash
curl -X POST http://localhost:8000/batches/3/recipe-preview \
  -H 'Content-Type: application/json' \
  -d '{"recipe": [{"batch_id": 1, "portions": 3}, {"batch_id": 2, "portions": 1}]}'
```

```json
{
  "batch_id": 3,
  "batch_name": "P1",
  "recipe": [
    {"batch_id": 1, "batch_name": "S1", "portions": 3},
    {"batch_id": 2, "batch_name": "S2", "portions": 1}
  ],
  "affected_batches": [
    {
      "batch_id": 3,
      "batch_name": "P1",
      "kind": "pool",
      "components": [
        {"sample_id": 1, "sample_name": "S1", "original": {"numerator": 1, "denominator": 2}, "preview": {"numerator": 3, "denominator": 4}, "delta": {"numerator": 1, "denominator": 4}},
        {"sample_id": 2, "sample_name": "S2", "original": {"numerator": 1, "denominator": 2}, "preview": {"numerator": 1, "denominator": 4}, "delta": {"numerator": -1, "denominator": 4}}
      ]
    },
    {
      "batch_id": 4,
      "batch_name": "P2",
      "kind": "pool",
      "components": [
        {"sample_id": 1, "sample_name": "S1", "original": {"numerator": 3, "denominator": 4}, "preview": {"numerator": 7, "denominator": 8}, "delta": {"numerator": 1, "denominator": 8}},
        {"sample_id": 2, "sample_name": "S2", "original": {"numerator": 1, "denominator": 4}, "preview": {"numerator": 1, "denominator": 8}, "delta": {"numerator": -1, "denominator": 8}}
      ]
    }
  ]
}
```

说明：

- 预演原料只能引用**比目标批次更早创建**的批次（含目标自身在内的引用都会被 400 拒绝），预演不会成环；
- 多路径汇合的下游批次精确合并、只返回一次；未受影响的批次不出现在结果中；
- 某样本在新配方下占比降为 0 时，以 `0/1` 返回，差值为负；
- 预演**不修改数据库**：调用前后 `/batches/{id}` 与 `/batches/{id}/composition` 的结果保持原样。

### 6. 按目标占比规划混样配方

`POST /pool-plans` 接收两个**不同的已有批次 id**（`ingredient_a_batch_id`、`ingredient_b_batch_id`）、一个已有**原始样本** id（`sample_id`）、目标占比（`numerator` / `denominator`，要求 `0 ≤ 分子 ≤ 分母` 且分母 > 0）以及正整数总份数上限 `max_total_portions`。

接口沿用上面的精确占比计算：设两原料中该样本的占比分别为 `a`、`b`，新混样按正整数份数 `x:y` 混合后占比为 `(a·x + b·y)/(x + y)`。接口反算使该占比**恰好等于**目标的正整数份数，只返回**总份数最小**的整数解；当两原料占比都等于目标时任意份数都满足，按约定取 `1:1`。返回的 `recipe` 可**直接作为 `/pools` 的配方提交**，`components` 给出新混样的全部原始样本精确占比。接口只做规划，**不创建任何批次**。

以第 4 节的 P2（S1 占 3/4）和 S2（S1 占 0）为原料，规划 S1 恰好占 1/2、总份数上限 10：

```bash
curl -X POST http://localhost:8000/pool-plans \
  -H 'Content-Type: application/json' \
  -d '{"ingredient_a_batch_id": 4, "ingredient_b_batch_id": 2, "sample_id": 1, "numerator": 1, "denominator": 2, "max_total_portions": 10}'
```

```json
{
  "sample_id": 1,
  "target": {"numerator": 1, "denominator": 2},
  "total_portions": 3,
  "recipe": [
    {"batch_id": 4, "portions": 2},
    {"batch_id": 2, "portions": 1}
  ],
  "components": [
    {"sample_id": 1, "sample_name": "S1", "numerator": 1, "denominator": 2},
    {"sample_id": 2, "sample_name": "S2", "numerator": 1, "denominator": 2}
  ]
}
```

校验：`2×P2 + 1×S2` 中 S1 = 2/3×3/4 = 1/2。把 `recipe` 原样提交即可创建该混样：

```bash
curl -X POST http://localhost:8000/pools \
  -H 'Content-Type: application/json' \
  -d '{"name": "P3", "recipe": [{"batch_id": 4, "portions": 2}, {"batch_id": 2, "portions": 1}]}'
```

说明：

- 目标占比**无法由正整数份数达到**时返回 **422**：包括目标落在两原料占比区间之外，或目标只等于其中一端（会迫使另一原料投入 0 份）；
- 最小整数解的**总份数超过上限**时返回 **422**，错误信息中给出最小总份数与上限；恰好等于上限时正常返回；
- 两原料 id 相同、分子 > 分母、分母 ≤ 0、份数上限非正整数等请求格式问题返回 **422**；引用不存在的批次返回 **400**；`sample_id` 指向混样批次（非原始样本）返回 **400**。

### 7. 原始样本禁配规则

`POST /exclusion-rules` 登记一对**不同且已存在的原始样本** id（`sample_a_id`、`sample_b_id`）。禁配对是**无序对**：服务端按 id 归一为 `(low_sample_id, high_sample_id)` 存储并返回；请求中两个 id 的先后顺序无关。重复登记同一无序对（含交换顺序）**不会生成第二条规则**：首次返回 **201**，重复登记幂等返回 **200** 且 `created_at` 保持首次登记时间。两个 id 相同、引用不存在的批次或引用混样批次（非原始样本）返回 **400**。

```bash
curl -X POST http://localhost:8000/exclusion-rules \
  -H 'Content-Type: application/json' \
  -d '{"sample_a_id": 2, "sample_b_id": 1}'
# 201（参数顺序无关，响应按 id 升序归一）
# {"low_sample_id":1,"high_sample_id":2,"created_at":"2026-09-25T09:00:00.000Z"}

curl -X POST http://localhost:8000/exclusion-rules \
  -H 'Content-Type: application/json' \
  -d '{"sample_a_id": 1, "sample_b_id": 2}'
# 200，重复登记幂等，不生成第二条
```

`GET /exclusion-rules` 列出全部规则，按样本对字典序（先 `low_sample_id` 后 `high_sample_id`）升序：

```bash
curl http://localhost:8000/exclusion-rules
```

```json
[
  {"low_sample_id": 1, "high_sample_id": 2, "created_at": "2026-09-25T09:00:00.000Z"}
]
```

**创建混样时的禁配检查**：`POST /pools` 会沿配方图递归求新混样可达的**全部原始样本**——无论样本直接出现在配方中，还是经嵌套混样、多路径汇合间接可达——只要其中任意两个命中已登记的禁配对，即返回 **409**，且**不创建批次或配方**（失败的批次名称不会被占用）。响应 `detail.pair` 给出按 id 升序的冲突样本对；同时命中多条规则时，按样本对字典序返回**第一对**：

```bash
# P1 = S1 + S3（创建于规则登记之前的批次不受影响）
# 登记禁配 (S1, S2) 后，尝试创建 P2 = 2×P1 + 3×S2：
# P2 经 P1 嵌套可达 S1，与直接投入的 S2 构成禁配对
curl -X POST http://localhost:8000/pools \
  -H 'Content-Type: application/json' \
  -d '{"name": "P2", "recipe": [{"batch_id": 3, "portions": 2}, {"batch_id": 2, "portions": 3}]}'
```

```json
{
  "detail": {
    "message": "混样命中原始样本禁配规则",
    "pair": [
      {"sample_id": 1, "sample_name": "S1"},
      {"sample_id": 2, "sample_name": "S2"}
    ]
  }
}
```

说明：

- 规则**只约束随后创建的批次**：登记之前已经存在的批次（即使本身同时含有这两个样本）不会被改动或删除，其详情与 `/batches/{id}/composition` 占比接口照常返回；随后创建的新批次若引用旧批次、经其可达样本构成禁配对，仍会被 409 拒绝；
- `/pool-plans` 只做配方规划、不创建批次，`/batches/{id}/recipe-preview` 只做内存预演，二者均不做禁配检查；
- 规则登记与混样创建使用同一个 SQLite 写事务模型（`BEGIN IMMEDIATE` 立即获取写锁），两类写操作相互串行：并发登记的规则一旦提交，随后的混样创建必然能读到该规则，不会漏检。

### 8. 禁配补录影响预演（只读）

`POST /exclusion-rules/impact-preview` 接收两个**不同的原始样本 id**（`sample_a_id`、`sample_b_id`，顺序无关，响应按 id 归一为 `low_sample_id` / `high_sample_id`），回答“**如果现在补录这条禁配规则，已经存在的哪些批次会受影响**”：

- 沿现有批次**配方 DAG** 递归计算两个样本的可达性（含嵌套混样、多路径汇合），返回已**同时含有两个样本的全部混样批次**，按批次 id 升序、各出现一次；两个样本从未汇合时返回空列表；
- `is_first_confluence`（**首次合流**）：该批次同时含两样本，但它的**每个直接原料都不同时含两样本**——即两样本沿配方边向上第一次在此汇合；
- `first_confluence_batch_ids`：该受影响批次沿配方边**可追溯到的全部首次合流批次 id**（批次自身是首次合流时含自身），按 id 升序；嵌套汇合与多路径汇合都不会漏报，同一首次合流批次只列出一次；
- **允许该无序对尚未登记为禁配对**：预演不读、不写规则，也**不修改任何已有批次或配方**；该对已登记时结果相同；
- 两个 id 相同、引用不存在的批次、或引用混样批次（非原始样本）返回 **400**；请求字段缺失或非正整数返回 **422**。

沿用上一节的数据（S1=1、S2=2、S3=3，P1=S1+S2 建于规则登记之前）。假设另有 P2 = P1+S3，以及独立的 P3 = S1+S2+S3，P4 同时引用 P2 与 P3：

```bash
curl -X POST http://localhost:8000/exclusion-rules/impact-preview \
  -H 'Content-Type: application/json' \
  -d '{"sample_a_id": 2, "sample_b_id": 1}'
```

```json
{
  "low_sample_id": 1,
  "high_sample_id": 2,
  "affected_batches": [
    {"batch_id": 4, "batch_name": "P1", "is_first_confluence": true,  "first_confluence_batch_ids": [4]},
    {"batch_id": 5, "batch_name": "P2", "is_first_confluence": false, "first_confluence_batch_ids": [4]},
    {"batch_id": 6, "batch_name": "P3", "is_first_confluence": true,  "first_confluence_batch_ids": [6]},
    {"batch_id": 7, "batch_name": "P4", "is_first_confluence": false, "first_confluence_batch_ids": [4, 6]}
  ]
}
```

说明：

- P1 与 P3 各自独立构成首次合流；P2 本身也含 S1、S2，但其直接原料 P1 已同时含两者，故不是首次合流，只可追溯到 P1；
- P4 经 P2、P3 两条路径汇合，可追溯的首次合流为 P1 与 P3，两个都列出且不重复；
- 份数不影响可达性，只影响占比，因此预演只沿配方图结构判断“可达/不可达”；
- 该端点是纯只读查询，调用后 `/exclusion-rules`、`/batches` 与各占比接口结果保持原样。

### 9. 份数诊断（反推记错的投入份数）

`POST /batches/{id}/portion-diagnosis`（服务地址 `http://localhost:8000`）接收目标混样批次的**实测原始样本占比列表**与**最大份数** `max_portions`：

- `measurements`：`sample_id` + `numerator`/`denominator`（分子 ≥ 0、分母 > 0 的严格整数）。样本 id **不得重复**且必须都是**原始样本**（400）；各占比**之和必须为 1**（422）；**未列出的样本占比视为 0**；
- 目标批次必须存在（404）且为**混样批次**（400）。

假设现有配方中**仅一项投入份数记错、其余不变**：接口沿嵌套配方精确展开各原料的原始样本占比，设被记错项为 `j`（记录份数 `p_j`，实际份数 `x`），其余项份数之和 `R` 与其贡献 `B_s` 固定，则实际组成满足 `(B_s + x·A_s)/(R + x) = m_s`。接口对每个样本建立有理数方程解出唯一候选 `x`，再用**全部样本占比**逐一复核，只返回**与原值不同且不超过 `max_portions`** 的正整数解，按原料 id 升序。若某项的份数变化**不改变组成且与实测吻合**（任意正整数份数都成立），列入 `undetermined`、**不枚举份数**；无解时 `solutions` 为**空列表**。接口只做内存演算，**不修改数据库**。

沿用前面的例子：P2（id=4）记录为 `1×P1 + 1×S1`（S1 应占 3/4），实测却是 S1=5/8、S2=3/8——恰好对应 `3×P1 + 1×S1`：

```bash
curl -X POST http://localhost:8000/batches/4/portion-diagnosis \
  -H 'Content-Type: application/json' \
  -d '{"measurements": [{"sample_id": 1, "numerator": 5, "denominator": 8}, {"sample_id": 2, "numerator": 3, "denominator": 8}], "max_portions": 10}'
```

```json
{
  "batch_id": 4,
  "batch_name": "P2",
  "solutions": [
    {"ingredient_batch_id": 3, "ingredient_name": "P1", "original_portions": 1, "corrected_portions": 3}
  ],
  "undetermined": []
}
```

校验：`3×P1 + 1×S1` 中 S1 = 3/4×1/2 + 1/4×1 = 5/8，S2 = 3/4×1/2 = 3/8，与实测一致；把 S1 的份数改成任何值都无法得到该组成，故 P1 是唯一可疑项。

### 10. 原始样本库存与批次制备

**补入库存 / 查询余额**：原始样本以“份”为单位持有库存。`POST /batches/{id}/inventory` 为**已存在的原始样本**补入正整数份（`portions`，累加到当前余额），返回补入后的余额与更新时间；`GET /batches/{id}/inventory` 查询余额，**从未补入过的样本余额为 0**（`updated_at` 为 `null`）。未知批次返回 **400**；混样批次不能持有库存，返回 **400**；非正整数份数返回 **422**。

```bash
# S1=1、S2=2，补入 10 份 S1 与 5 份 S2
curl -X POST http://localhost:8000/batches/1/inventory \
  -H 'Content-Type: application/json' -d '{"portions": 10}'
# {"sample_id":1,"sample_name":"S1","balance":10,"updated_at":"2026-09-25T10:00:00.000Z"}

curl http://localhost:8000/batches/2/inventory
# {"sample_id":2,"sample_name":"S2","balance":0,"updated_at":null}
```

**批次制备**：`POST /batches/{id}/preparations` 接收正整数**制备次数** `times`。每制备一次：

- 目标是**原始样本**时直接消耗**自身 1 份**；
- 目标是**混样**时按其配方消耗相应份数的上游批次，再沿配方图**逐层乘算**，同一原始样本经多条路径到达时需求**相加合并**，最终得到各原始样本的**整数消耗**（份数均为整数，全程不产生分数）。

**核算与扣减在同一个 SQLite 写事务内完成**（`BEGIN IMMEDIATE` 立即取写锁）：

- 任一样本**库存不足**：整体回滚返回 **409**，`detail.shortages` 按样本 id 升序列出每个缺口样本的需求份数、可用份数与缺口，**库存与制备记录均不变化**，可补足后原样重试；
- **全部足量**：一次扣减各样本库存，写入制备记录（目标批次、次数）与消耗明细，返回 **201**：记录内容与扣减后的剩余库存。

沿用前面的批次（P1=`1×S1+1×S2` id=3，P2=`1×P1+1×S1` id=4）。制备 P2 一次时，S1 经 P1（1 份）与直接投入（1 份）两条路径合并为 2 份、S2 为 1 份；制备 3 次即 `6×S1 + 3×S2`：

```bash
curl -X POST http://localhost:8000/batches/4/preparations \
  -H 'Content-Type: application/json' -d '{"times": 3}'
```

```json
{
  "id": 1,
  "batch_id": 4,
  "batch_name": "P2",
  "times": 3,
  "created_at": "2026-09-25T10:05:00.000Z",
  "consumptions": [
    {"sample_id": 1, "sample_name": "S1", "portions": 6},
    {"sample_id": 2, "sample_name": "S2", "portions": 3}
  ],
  "remaining_inventory": [
    {"sample_id": 1, "sample_name": "S1", "balance": 4, "updated_at": "2026-09-25T10:00:00.000Z"},
    {"sample_id": 2, "sample_name": "S2", "balance": 2, "updated_at": "2026-09-25T10:00:00.000Z"}
  ]
}
```

**按记录 id 查询**：`GET /preparations/{id}` 返回同结构的制备记录；`consumptions` 是该次制备的历史快照，`remaining_inventory` 为**查询时**的实时余额（后续补库/制备会反映其中）。记录不存在返回 **404**。

```bash
curl http://localhost:8000/preparations/1
```

库存不足的例子（S1 余 4 份，再制备 P2 三次需 6 份；S2 余 2 份，需 3 份）：

```bash
curl -X POST http://localhost:8000/batches/4/preparations \
  -H 'Content-Type: application/json' -d '{"times": 3}'
```

```json
{
  "detail": {
    "message": "原始样本库存不足",
    "shortages": [
      {"sample_id": 1, "sample_name": "S1", "required": 6, "available": 4, "shortage": 2},
      {"sample_id": 2, "sample_name": "S2", "required": 3, "available": 2, "shortage": 1}
    ]
  }
}
```

说明：

- 恰好耗尽（需求 = 余额）视为足量，制备成功且对应余额变为 0；从未补入库存的样本可用份数为 0；
- 制备次数必须为严格正整数（0、负数、小数、字符串、布尔均 **422**）；目标批次不存在返回 **400**；
- 并发制备由同一写事务串行：不会出现库存超扣（余额恒非负），库存只够部分请求时其余请求得到 409 且不留任何记录。

### 11. 整单制备（多批次合并核算）

`POST /preparation-jobs` 一次接收**非空制备清单** `items`，每项为 `batch_id`（已有批次，原始样本或混样均可）+ `times`（正整数制备次数）：

- 同一批次在清单中**重复出现时先合并次数**，每个批次只生成**一条**制备记录（次数为合并后的值）；
- 先沿配方图计算**每项**对原始样本的整数需求（与单批次制备同一规则：逐层乘算、多路径合并），再**按样本汇总整单需求**统一核算——共享样本按汇总后的总需求判定，**不会被各项分别判为足量**；
- 读取余额、核算缺口、扣减库存与写入各批次制备记录在**同一 SQLite 写事务**内完成：任一样本不足返回 **409**（`detail.shortages` 给出该样本的整单**总需求**、余额与缺口），**整单不留任何制备记录、余额不变**，可补足后原样重试；
- 成功返回 **201**：`preparations` 按批次 id 升序给出各批次的制备记录 id、合并后次数与该批次消耗明细，`consumptions` 为整单按样本汇总的总消耗；每条记录与单批次制备的记录同表存储，可由 `GET /preparations/{id}` 照常读取。

沿用前面的批次（P1=`1×S1+1×S2` id=3，P2=`1×P1+1×S1` id=4）。先补足库存，再提交整单：P2 在清单中出现两次（1+1 合并为 2 次）、P1 一次——P2 每次消耗 `2×S1 + 1×S2`、P1 每次消耗 `1×S1 + 1×S2`，整单合计 `5×S1 + 3×S2`：

```bash
curl -X POST http://localhost:8000/batches/1/inventory \
  -H 'Content-Type: application/json' -d '{"portions": 10}'
curl -X POST http://localhost:8000/batches/2/inventory \
  -H 'Content-Type: application/json' -d '{"portions": 10}'

curl -X POST http://localhost:8000/preparation-jobs \
  -H 'Content-Type: application/json' \
  -d '{"items": [{"batch_id": 4, "times": 1}, {"batch_id": 3, "times": 1}, {"batch_id": 4, "times": 1}]}'
```

```json
{
  "preparations": [
    {
      "batch_id": 3,
      "batch_name": "P1",
      "preparation_id": 2,
      "times": 1,
      "consumptions": [
        {"sample_id": 1, "sample_name": "S1", "portions": 1},
        {"sample_id": 2, "sample_name": "S2", "portions": 1}
      ]
    },
    {
      "batch_id": 4,
      "batch_name": "P2",
      "preparation_id": 3,
      "times": 2,
      "consumptions": [
        {"sample_id": 1, "sample_name": "S1", "portions": 4},
        {"sample_id": 2, "sample_name": "S2", "portions": 2}
      ]
    }
  ],
  "consumptions": [
    {"sample_id": 1, "sample_name": "S1", "portions": 5},
    {"sample_id": 2, "sample_name": "S2", "portions": 3}
  ]
}
```

（`preparation_id` 接续上一节已产生的记录编号；响应按批次 id 升序，与清单中的顺序无关。）

共享样本按整单汇总核算的例子：假设 S1 只余 3 份，整单为 P2×1（需 2×S1）+ P1×2（需 2×S1）——单项分别判断都“够”，但汇总需求 4 > 3，整单被拒绝且不留任何记录：

```bash
curl -X POST http://localhost:8000/preparation-jobs \
  -H 'Content-Type: application/json' \
  -d '{"items": [{"batch_id": 4, "times": 1}, {"batch_id": 3, "times": 2}]}'
```

```json
{
  "detail": {
    "message": "原始样本库存不足",
    "shortages": [
      {"sample_id": 1, "sample_name": "S1", "required": 4, "available": 3, "shortage": 1}
    ]
  }
}
```

说明：

- 清单为空返回 **422**；引用不存在的批次返回 **400**；制备次数非严格正整数（0、负数、小数、字符串、布尔）返回 **422**；
- 从未补入库存的样本可用份数为 0；汇总需求恰好等于余额视为足量，对应余额变为 0；
- 与单批次制备共用同一写事务模型：并发的整单与单批次制备相互串行，不会超扣，失败方不留任何记录；
- 409 时 `shortages` 中的 `required` 是该样本的**整单总需求**（各项合并后的值），不是单项需求。

### 12. 只读排产预演（候选制备单选型）

`POST /preparation-jobs/selection-preview`（服务地址 `http://localhost:8000`）接收 **1～12 条候选制备单** `items`，每条为 `batch_id`（已有批次，原始样本或混样均可）+ `times`（正整数制备次数）：

- 订单按**请求位置**（从 1 开始）标识：**允许同一批次重复出现**，与整单制备不同，重复批次**不合并次数**，各自作为互相独立的候选订单参与选型；
- 每条订单的需求复用配方图**整数消耗展开**（与单批次/整单制备同一规则：逐层乘算、多路径合并），得到该订单对各原始样本的整数需求；
- 依据**同一库存快照**（显式只读事务，并发补库/制备不会混入预演中途）选出可行子集：入选订单对**每一样本的总需求都不超过余额**（从未补入的样本余额为 0；空集恒可行，库存全不足时返回空集）；
- 选型优先级：先**最大化入选订单数**，再**最大化入选制备次数之和**，仍并列时取**入选位置序列字典序最小**者；
- 返回入选/未入选位置（均按位置升序）、入选订单按样本汇总的消耗、以及同一快照下满足消耗后的剩余库存（只列被消耗的样本）；
- **只读**：不扣减库存、不创建制备记录、不改动任何数据；清单为空或超过 12 条、次数非严格正整数返回 **422**，引用未知批次返回 **400**（沿用现有错误约定）。

沿用前面的批次（P1=`1×S1+1×S2` id=3，P2=`1×P1+1×S1` id=4，每次需 `2×S1+1×S2`）。假设当前库存 S1=5、S2=3，候选清单为：位置 1 制备 P2 两次（需 `4×S1 + 2×S2`），位置 2～4 各制备 S1 两次（各需 `2×S1`）。位置 1 与位置 2 不可兼得（合计 `6×S1` 超过余额 5），两两组合中订单数最多的是任选两个 S1 订单（2 条、共 4 次），位置序列字典序最小者为 `{2, 3}`：

```bash
curl -X POST http://localhost:8000/preparation-jobs/selection-preview \
  -H 'Content-Type: application/json' \
  -d '{"items": [{"batch_id": 4, "times": 2}, {"batch_id": 1, "times": 2}, {"batch_id": 1, "times": 2}, {"batch_id": 1, "times": 2}]}'
```

```json
{
  "selected_positions": [2, 3],
  "unselected_positions": [1, 4],
  "consumptions": [
    {"sample_id": 1, "sample_name": "S1", "portions": 4}
  ],
  "remaining_inventory": [
    {"sample_id": 1, "sample_name": "S1", "balance": 1, "updated_at": "2026-09-25T10:00:00.000Z"}
  ]
}
```

说明：

- 选型只比较**可行性**与上述优先级，不模拟扣减顺序；`remaining_inventory` 是“若按入选结果制备”在同一快照下的推算余额，接口本身**不写库**；
- 没有任何订单可行时返回空集：`selected_positions`、`consumptions`、`remaining_inventory` 均为空列表，`unselected_positions` 为全部位置；
- 预演结果可直接作为 `/preparation-jobs` 的 `items` 提交执行（整单制备按汇总需求核算，与预演同一口径）。

### 13. 制备记录冲正（逐样本返还库存）

对一条**已经存在**的制备记录发起冲正：服务端按该记录**已保存的消耗明细**（不重新展开当前配方）把各原始样本当初消耗的份数**逐样本加回**库存，并保存与该记录**一一对应**的冲正时间。**单批制备**（`POST /batches/{id}/preparations`）与**整单制备**（`POST /preparation-jobs`）生成的记录结构相同，都可以冲正。

- `POST /preparations/{id}/reversal`：首次冲正返回 **201**；记录读取、冲正判重、库存返还与冲正时间保存在**同一 SQLite 写事务**（`BEGIN IMMEDIATE`）内完成；
- **重复冲正幂等**：对同一记录再次冲正不返还任何库存，返回 **200** 与**同一个冲正记录**（冲正时间不变）；
- **未知制备记录**（不存在的 id）返回 **404**，不改动任何库存；
- 冲正**不删改**原制备记录与消耗明细：`GET /preparations/{id}` 与批次接口照常可查，其中的剩余库存反映返还后的实时余额；冲正后也可以正常补库、再次制备。

沿用前面的例子（P2=`1×P1+1×S1` id=4，每次消耗 `2×S1 + 1×S2`）。S1 补入 10 份、S2 补入 5 份，制备 P2 一次后余额为 S1=8、S2=4：

```bash
curl -X POST http://localhost:8000/batches/4/preparations \
  -H 'Content-Type: application/json' -d '{"times": 1}'
# 制备记录 id=1，consumptions：S1×2、S2×1

curl -X POST http://localhost:8000/preparations/1/reversal
```

```json
{
  "preparation_id": 1,
  "reversed_at": "2026-09-25T11:00:00.000Z",
  "items": [
    {"sample_id": 1, "sample_name": "S1", "returned_portions": 2, "balance": 10, "updated_at": "2026-09-25T10:00:00.000Z"},
    {"sample_id": 2, "sample_name": "S2", "returned_portions": 1, "balance": 5,  "updated_at": "2026-09-25T10:00:00.000Z"}
  ]
}
```

- `returned_portions` 即制备记录保存的各样本消耗份数；`balance` 为返还后的当前库存；`items` 按**样本 id 升序**；
- `updated_at` 是样本库存的**最近一次补入时间**，冲正返还不改写该时间（从未补入过时为 `null`）。

**重复冲正**返回同一冲正记录（`reversed_at` 与首次一致）、状态码 **200**，库存不会被二次返还：

```bash
curl -X POST http://localhost:8000/preparations/1/reversal
# 200，响应体与首次完全相同；S1、S2 余额仍为 10、5
```

`GET /preparations/{id}/reversal` 查询冲正时间、各样本返还份数与**查询时**的当前库存（后续补库/制备会反映在 `balance` 中，`returned_portions` 始终是历史明细）。制备记录**尚未冲正**与**记录不存在**均返回 **404**：

```bash
curl http://localhost:8000/preparations/1/reversal
# 200：{"preparation_id":1,"reversed_at":"...","items":[...]}

curl http://localhost:8000/preparations/2/reversal
# 404 {"detail":"制备记录尚未冲正: 2"}

curl http://localhost:8000/preparations/999/reversal
# 404 {"detail":"制备记录不存在: 999"}
```

整单制备生成的多条记录可分别冲正：每条只按**自己的**消耗明细返还；整单中两条记录若消耗了同一个样本，两条各自冲正后该样本的两份返还都会加回。冲正不影响新的制备——库存返还后可用同一批次再次制备，新制备生成新的记录。

### 14. 原始样本质量封存（封存 / 解除 / 查询与制备拦截）

**封存与解除封存**：仅**原始样本**可封存。`POST /batches/{id}/seal` 请求体必须给出非空的状态变更原因 `reason`（1～500 字符，纯空白同样拒绝），服务端记录原因与封存时间，返回 **200** 与当前状态；`POST /batches/{id}/unseal` 以同样方式解除封存。**同一状态重复操作返回 409**：已封存再次封存、未封存却解除封存都不会新增流水。混样批次或未知批次对封存/解除/查询均返回 **400**（沿用库存接口约定）。

```bash
# S1=1、S2=2，P1=1×S1+1×S2 id=3，均已补足库存
curl -X POST http://localhost:8000/batches/1/seal \
  -H 'Content-Type: application/json' -d '{"reason": "质检异常，等待复核"}'
```

```json
{
  "sample_id": 1,
  "sample_name": "S1",
  "sealed": true,
  "reason": "质检异常，等待复核",
  "sealed_at": "2026-09-26T09:00:00.000Z",
  "history": [
    {"action": "sealed", "reason": "质检异常，等待复核", "created_at": "2026-09-26T09:00:00.000Z"}
  ]
}
```

```bash
# 已封存再次封存 → 409，不新增流水
curl -X POST http://localhost:8000/batches/1/seal \
  -H 'Content-Type: application/json' -d '{"reason": "重复封存"}'
# {"detail":{"message":"原始样本已处于封存状态","sealed_samples":[1]}}

# 解除封存 → 200，追加第二条历史
curl -X POST http://localhost:8000/batches/1/unseal \
  -H 'Content-Type: application/json' -d '{"reason": "复检合格"}'
```

```json
{
  "sample_id": 1,
  "sample_name": "S1",
  "sealed": false,
  "reason": null,
  "sealed_at": null,
  "history": [
    {"action": "sealed",   "reason": "质检异常，等待复核", "created_at": "2026-09-26T09:00:00.000Z"},
    {"action": "unsealed", "reason": "复检合格",         "created_at": "2026-09-26T09:05:00.000Z"}
  ]
}
```

`GET /batches/{id}/seal` 查询当前状态与**全部状态变更历史**（按时间升序，封存/解除交替追加，解除不删除历史；从未封存过的样本 `sealed=false`、`history=[]`）。解除后再次封存会得到第三条历史，`sealed_at` 取最近一次封存时间。

**对制备的拦截**：`POST /batches/{id}/preparations`（单批制备）与 `POST /preparation-jobs`（整单制备）都会在与扣减核算**同一个 SQLite 写事务**内，沿**现有配方图**（含嵌套混样、多路径汇合）识别本次会消耗的全部上游原始样本——目标是原始样本时即其自身。其中任一处于封存中，即返回 **409**，`detail.sealed_samples` 按样本 id 升序给出**全部命中封存的样本**（整单按整单汇总去重），**整次请求不扣库存、不写制备记录**；封存检查先于库存缺口检查，同时命中时只返回封存冲突。解除封存后可原样重试：

```bash
# P2 = 1×P1 + 1×S1 id=4，S1 被封存：P2 经 P1 嵌套可达 S1
curl -X POST http://localhost:8000/batches/4/preparations \
  -H 'Content-Type: application/json' -d '{"times": 1}'
```

```json
{
  "detail": {
    "message": "上游原始样本已封存，禁止制备",
    "sealed_samples": [
      {"sample_id": 1, "sample_name": "S1"}
    ]
  }
}
```

说明：

- 封存只拦截**新的制备**：封存之前已经生成的历史制备记录仍可由 `GET /preparations/{id}` 查询、由 `POST /preparations/{id}/reversal` 冲正（冲正按已保存明细返还，不重新展开配方、不受封存影响）；批次详情、占比、配方预演等只读接口也均不受影响；
- 封存不影响库存数据本身：封存期间仍可补入库存（`POST /batches/{id}/inventory`），余额照常累加，解除封存后这些库存立即可用于制备；
- 只读排产预演 `/preparation-jobs/selection-preview` 不做封存检查（纯选型预演，不扣库存、不写记录）；
- **并发安全**：封存/解除与制备核算都以 `BEGIN IMMEDIATE` 立即获取 SQLite 写锁，状态变更与制备相互串行——封存事务一旦提交，随后开始的制备必然读到封存状态并返回 409，不会出现“封存已生效却制备成功”的漏拦；
- 原因缺失、为空或纯空白返回 **422**；未知批次或混样批次返回 **400**；同一状态重复操作返回 **409**。

### 15. 原始样本库存盘点（预演 → 提交 → 查询）

**盘点预演**：`POST /inventory-counts/preview` 接收**非空且样本 id 不重复**的盘点项列表 `items`（每项为 `sample_id` + 实盘**非负整数**份数 `counted`），在**同一库存快照**（显式只读事务，并发补库/制备/盘点不会混入预演中途）下返回各项的账面数 `book_balance`（**无库存行按 0**）、实盘数与差额 `delta = counted − book_balance`，按样本 id 升序。预演**不写库**：不改库存、不留盘点记录。未知批次或混样批次返回 **400**；空清单、重复样本 id、负数/非整数份数返回 **422**。

```bash
# S1=1 账面 10 份，S2=2 从未补入（账面按 0）
curl -X POST http://localhost:8000/inventory-counts/preview \
  -H 'Content-Type: application/json' \
  -d '{"items": [{"sample_id": 1, "counted": 7}, {"sample_id": 2, "counted": 2}]}'
```

```json
{
  "items": [
    {"sample_id": 1, "sample_name": "S1", "book_balance": 10, "counted": 7, "delta": -3},
    {"sample_id": 2, "sample_name": "S2", "book_balance": 0,  "counted": 2, "delta": 2}
  ]
}
```

**提交盘点**：`POST /inventory-counts` 接收盘点项（`sample_id` + `counted` + 预演返回的账面数 `book_balance`）与**非空白原因** `reason`。服务端在**同一 SQLite 写事务**（`BEGIN IMMEDIATE`）内核对全部样本**仍为原始样本**且当前余额**与预演账面数一致**（无库存行按 0）：

- **任一变化**（样本不存在或已不是原始样本、余额与预演账面数不符）→ **409**，`detail.conflicts` 按样本 id 升序给出冲突项的预演账面数与当前余额，**整单不改库存、不留盘点记录**，可按新余额重新预演后再提交；
- **全部一致** → 按实盘数更新各样本余额（无库存行的样本新建库存行），保存盘点单（原因、提交时间）与各项的**调整前账面数、调整后实盘数、差额**，返回 **201**。

```bash
curl -X POST http://localhost:8000/inventory-counts \
  -H 'Content-Type: application/json' \
  -d '{"items": [{"sample_id": 1, "counted": 7, "book_balance": 10}, {"sample_id": 2, "counted": 2, "book_balance": 0}], "reason": "月末盘点"}'
```

```json
{
  "id": 1,
  "reason": "月末盘点",
  "created_at": "2026-09-26T10:00:00.000Z",
  "items": [
    {"sample_id": 1, "sample_name": "S1", "book_balance": 10, "counted": 7, "delta": -3},
    {"sample_id": 2, "sample_name": "S2", "book_balance": 0,  "counted": 2, "delta": 2}
  ]
}
```

预演到提交之间余额被并发改动（如另一终端制备扣减）时，基于旧账面数的提交被拒绝：

```json
{
  "detail": {
    "message": "盘点预演已过期，请重新预演",
    "conflicts": [
      {"sample_id": 1, "sample_name": "S1", "expected_book_balance": 10, "current_balance": 7}
    ]
  }
}
```

**查询盘点**：`GET /inventory-counts/{id}` 返回已提交盘点的明细（单头原因、时间与各项调整前后数、差额，按样本 id 升序）；记录不存在返回 **404**。

```bash
curl http://localhost:8000/inventory-counts/1
```

说明：

- 账实相符（差额为 0）与实盘清零（`counted` 为 0）都是合法盘点项；盘点会刷新样本库存的 `updated_at`；
- 提交时混样批次或未知 id 与余额不一致一样按 **409** 冲突项返回（`current_balance` 为 `null`），整单回滚；
- 盘点只调整**原始样本**库存余额，不影响批次、配方、制备记录与封存状态；盘点记录一旦提交不可改，如需修正请按新账面数再盘点一次。

### 错误示例

```bash
# 空配方 → 422
curl -X POST http://localhost:8000/pools -H 'Content-Type: application/json' \
  -d '{"name": "P", "recipe": []}'

# 未知批次 → 400 {"detail":"未知批次 id: [999]"}
curl -X POST http://localhost:8000/pools -H 'Content-Type: application/json' \
  -d '{"name": "P", "recipe": [{"batch_id": 999, "portions": 1}]}'

# 非正份数（0 / 负数 / 小数 / 字符串）→ 422
curl -X POST http://localhost:8000/pools -H 'Content-Type: application/json' \
  -d '{"name": "P", "recipe": [{"batch_id": 1, "portions": 0}]}'

# 查询不存在的批次 → 404
curl http://localhost:8000/batches/999/composition

# 预演原料引用了比目标批次更晚创建的批次（或目标自身）→ 400
curl -X POST http://localhost:8000/batches/3/recipe-preview \
  -H 'Content-Type: application/json' \
  -d '{"recipe": [{"batch_id": 4, "portions": 1}]}'

# 规划目标无法由正整数份数达到（P2 与 S1 中 S1 占比为 3/4、1，目标 1/4 在区间之外）→ 422
curl -X POST http://localhost:8000/pool-plans -H 'Content-Type: application/json' \
  -d '{"ingredient_a_batch_id": 4, "ingredient_b_batch_id": 1, "sample_id": 1, "numerator": 1, "denominator": 4, "max_total_portions": 10}'

# 规划的最小整数解超过总份数上限（2/3 需要 2+1=3 份）→ 422
curl -X POST http://localhost:8000/pool-plans -H 'Content-Type: application/json' \
  -d '{"ingredient_a_batch_id": 1, "ingredient_b_batch_id": 2, "sample_id": 1, "numerator": 2, "denominator": 3, "max_total_portions": 2}'

# 登记禁配对时两个样本 id 相同 → 400
curl -X POST http://localhost:8000/exclusion-rules -H 'Content-Type: application/json' \
  -d '{"sample_a_id": 1, "sample_b_id": 1}'

# 登记禁配对引用不存在的批次 → 400 {"detail":"未知批次 id: [999]"}
curl -X POST http://localhost:8000/exclusion-rules -H 'Content-Type: application/json' \
  -d '{"sample_a_id": 1, "sample_b_id": 999}'

# 已登记禁配对 (1,2)，创建混样可达样本同时含 1 和 2（含经嵌套混样间接可达）→ 409
curl -X POST http://localhost:8000/pools -H 'Content-Type: application/json' \
  -d '{"name": "PBAD", "recipe": [{"batch_id": 1, "portions": 1}, {"batch_id": 2, "portions": 1}]}'
# {"detail":{"message":"混样命中原始样本禁配规则","pair":[{"sample_id":1,"sample_name":"S1"},{"sample_id":2,"sample_name":"S2"}]}}

# 禁配补录影响预演：两个样本 id 相同 → 400
curl -X POST http://localhost:8000/exclusion-rules/impact-preview -H 'Content-Type: application/json' \
  -d '{"sample_a_id": 1, "sample_b_id": 1}'

# 禁配补录影响预演：引用不存在的批次 / 混样批次 → 400
curl -X POST http://localhost:8000/exclusion-rules/impact-preview -H 'Content-Type: application/json' \
  -d '{"sample_a_id": 1, "sample_b_id": 999}'
curl -X POST http://localhost:8000/exclusion-rules/impact-preview -H 'Content-Type: application/json' \
  -d '{"sample_a_id": 1, "sample_b_id": 3}'   # 3 为混样批次 id 时同样 400

# 份数诊断：实测占比之和不为 1（未列样本视为 0）→ 422
curl -X POST http://localhost:8000/batches/3/portion-diagnosis -H 'Content-Type: application/json' \
  -d '{"measurements": [{"sample_id": 1, "numerator": 1, "denominator": 3}], "max_portions": 10}'

# 份数诊断：目标不是混样批次 → 400
curl -X POST http://localhost:8000/batches/1/portion-diagnosis -H 'Content-Type: application/json' \
  -d '{"measurements": [{"sample_id": 1, "numerator": 1, "denominator": 1}], "max_portions": 10}'

# 为不存在的批次补入/查询库存 → 400 {"detail":"未知批次 id: 999"}
curl -X POST http://localhost:8000/batches/999/inventory -H 'Content-Type: application/json' \
  -d '{"portions": 5}'
curl http://localhost:8000/batches/999/inventory

# 为混样批次补入库存 → 400（只有原始样本可持有库存）
curl -X POST http://localhost:8000/batches/3/inventory -H 'Content-Type: application/json' \
  -d '{"portions": 5}'

# 补入非正整数份（0 / 负数 / 小数 / 字符串 / 布尔）→ 422
curl -X POST http://localhost:8000/batches/1/inventory -H 'Content-Type: application/json' \
  -d '{"portions": 0}'

# 制备不存在的批次 → 400 {"detail":"未知批次 id: 999"}
curl -X POST http://localhost:8000/batches/999/preparations -H 'Content-Type: application/json' \
  -d '{"times": 1}'

# 制备次数非正整数 → 422
curl -X POST http://localhost:8000/batches/1/preparations -H 'Content-Type: application/json' \
  -d '{"times": 0}'

# 库存不足 → 409（缺口明细，库存与制备记录均不变化）
curl -X POST http://localhost:8000/batches/4/preparations -H 'Content-Type: application/json' \
  -d '{"times": 3}'
# {"detail":{"message":"原始样本库存不足","shortages":[{"sample_id":1,...}]}}

# 整单制备：空清单 → 422
curl -X POST http://localhost:8000/preparation-jobs -H 'Content-Type: application/json' \
  -d '{"items": []}'

# 整单制备：引用未知批次 → 400 {"detail":"未知批次 id: [999]"}
curl -X POST http://localhost:8000/preparation-jobs -H 'Content-Type: application/json' \
  -d '{"items": [{"batch_id": 999, "times": 1}]}'

# 整单制备：共享样本按汇总需求核算不足 → 409（整单不留记录、余额不变）
curl -X POST http://localhost:8000/preparation-jobs -H 'Content-Type: application/json' \
  -d '{"items": [{"batch_id": 4, "times": 1}, {"batch_id": 3, "times": 2}]}'
# {"detail":{"message":"原始样本库存不足","shortages":[{"sample_id":1,"required":4,...}]}}

# 只读排产预演：清单为空或超过 12 条 → 422
curl -X POST http://localhost:8000/preparation-jobs/selection-preview \
  -H 'Content-Type: application/json' -d '{"items": []}'

# 只读排产预演：引用未知批次 → 400 {"detail":"未知批次 id: [999]"}
curl -X POST http://localhost:8000/preparation-jobs/selection-preview \
  -H 'Content-Type: application/json' -d '{"items": [{"batch_id": 999, "times": 1}]}'

# 查询不存在的制备记录 → 404
curl http://localhost:8000/preparations/999

# 冲正不存在的制备记录 → 404 {"detail":"制备记录不存在: 999"}
curl -X POST http://localhost:8000/preparations/999/reversal

# 查询未冲正制备记录的冲正信息 → 404 {"detail":"制备记录尚未冲正: 1"}
curl http://localhost:8000/preparations/1/reversal

# 重复冲正不报错：首次 201，再次冲正幂等返回 200 且不再返还库存
curl -X POST http://localhost:8000/preparations/1/reversal

# 封存未知批次 / 混样批次 → 400（只有原始样本可封存）
curl -X POST http://localhost:8000/batches/999/seal -H 'Content-Type: application/json' \
  -d '{"reason": "原因"}'
curl -X POST http://localhost:8000/batches/3/seal -H 'Content-Type: application/json' \
  -d '{"reason": "原因"}'

# 封存原因为空 / 纯空白 / 缺失 → 422
curl -X POST http://localhost:8000/batches/1/seal -H 'Content-Type: application/json' \
  -d '{"reason": ""}'
curl -X POST http://localhost:8000/batches/1/seal -H 'Content-Type: application/json' \
  -d '{"reason": "   "}'

# 已封存再次封存 / 未封存却解除 → 409（不新增状态变更流水）
curl -X POST http://localhost:8000/batches/1/seal -H 'Content-Type: application/json' \
  -d '{"reason": "重复封存"}'
curl -X POST http://localhost:8000/batches/2/unseal -H 'Content-Type:application/json' \
  -d '{"reason": "重复解除"}'
# {"detail":{"message":"原始样本已处于封存状态","sealed_samples":[1]}}

# 制备命中封存上游（含经嵌套混样间接可达）→ 409：
# 不扣库存、不写制备记录；detail.sealed_samples 按样本 id 升序列出全部命中样本
curl -X POST http://localhost:8000/batches/4/preparations -H 'Content-Type: application/json' \
  -d '{"times": 1}'
curl -X POST http://localhost:8000/preparation-jobs -H 'Content-Type: application/json' \
  -d '{"items": [{"batch_id": 4, "times": 1}]}'
# {"detail":{"message":"上游原始样本已封存，禁止制备","sealed_samples":[{"sample_id":1,"sample_name":"S1"}]}}

# 盘点预演：空清单 / 样本 id 重复 / 实盘为负数或非整数 → 422
curl -X POST http://localhost:8000/inventory-counts/preview -H 'Content-Type: application/json' \
  -d '{"items": []}'
curl -X POST http://localhost:8000/inventory-counts/preview -H 'Content-Type: application/json' \
  -d '{"items": [{"sample_id": 1, "counted": 1}, {"sample_id": 1, "counted": 2}]}'
curl -X POST http://localhost:8000/inventory-counts/preview -H 'Content-Type: application/json' \
  -d '{"items": [{"sample_id": 1, "counted": -1}]}'

# 盘点预演：未知批次 / 混样批次 → 400（只有原始样本可盘点）
curl -X POST http://localhost:8000/inventory-counts/preview -H 'Content-Type: application/json' \
  -d '{"items": [{"sample_id": 999, "counted": 1}]}'

# 提交盘点：原因为空或纯空白 → 422
curl -X POST http://localhost:8000/inventory-counts -H 'Content-Type: application/json' \
  -d '{"items": [{"sample_id": 1, "counted": 7, "book_balance": 10}], "reason": "   "}'

# 提交盘点：预演后余额已变化（账面数过期）→ 409 冲突项，整单不改库存、不留记录
curl -X POST http://localhost:8000/inventory-counts -H 'Content-Type: application/json' \
  -d '{"items": [{"sample_id": 1, "counted": 7, "book_balance": 10}], "reason": "月末盘点"}'
# {"detail":{"message":"盘点预演已过期，请重新预演","conflicts":[{"sample_id":1,"sample_name":"S1","expected_book_balance":10,"current_balance":7}]}}

# 查询不存在的盘点记录 → 404
curl http://localhost:8000/inventory-counts/999
```

## 占比计算说明

- 每个原始样本对自身的占比为 `1/1`；
- 混样批次中，某成分的占比 = 该成分投入份数 ÷ 本层总份数，再乘以下游展开结果，逐层递归；
- 同一原始样本经多条路径到达时，各路径占比相加合并；
- 全程使用 `fractions.Fraction` 精确有理数运算，结果自动约分，直接返回分子/分母，无浮点误差；
- 由于新批次只能引用已创建的批次，配方图无环，递归必然终止。

## 本地开发（不用 Docker）

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
uvicorn app.main:app --reload          # http://localhost:8000
pytest                                 # 运行测试（使用临时数据库，互不影响）
```

## 项目结构

```
app/
  main.py       # FastAPI 路由与请求处理（禁配规则、目标占比规划、配方预演、份数诊断、库存补入、原始样本封存/解除/查询、批次制备、整单制备、只读排产预演、制备记录冲正与库存盘点端点）
  db.py         # SQLite 连接与建表（启动时自动执行，含禁配规则、库存、制备记录与冲正记录、封存状态与变更流水、盘点单与盘点明细表及写事务设置）
  schemas.py    # Pydantic 请求/响应模型（含严格正整数校验、封存/盘点原因非空白校验、盘点项非空不重复校验）
  service.py    # 递归占比计算、配方图可达原始样本、禁配补录影响预演、目标占比反算、配方预演、份数诊断、制备整数消耗与排产子集选型（Fraction 精确分数）
tests/
  test_api.py   # 端到端测试（含多级合并、约分、占比规划、配方预演、份数诊断、禁配规则与补录影响预演、库存与制备扣减、整单制备、只读排产预演、制备冲正幂等与返还、封存状态机与制备封存拦截、库存盘点预演/提交/查询与并发冲突、并发与各类非法输入）
Dockerfile
docker-compose.yml
requirements.txt
requirements-dev.txt
```
