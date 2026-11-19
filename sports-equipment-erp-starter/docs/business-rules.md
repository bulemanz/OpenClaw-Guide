# 生产管理领域规则与工程验收

## 1. 生产场景

小型体育器材：原材料与标准件采购，依据 BOM 配料，可能经过简单加工与组装，完工后成品入库，按销售订单发货。

## 2. BOM 与工单

- 物料主数据统一用 Item SKU；区分原料、配件、半成品、成品和耗材。
- BOM 以 `(productItemId, version)` 唯一化；激活同一产品的新版本时，应通过事务确保只能存在一个 ACTIVE 版本。
- 工单的 `finishedItemId` 必须与所选 BOM 的 `productItemId` 一致。
- 在工单从 DRAFT 下达到 RELEASED 时，将 BOM 行写入 ProductionOrderMaterial 并固定 `bomQuantitySnapshot`、`scrapRateSnapshot`、`requiredQuantity`。
- 工单下达后禁止静默更新快照。修改物料需求应使用显式变更单并写入审计日志。

## 3. 库存账本与余额

- InventoryTransaction 是不可随意更改的库存流水，正数为入库，负数为出库。
- 库存按 `(warehouseId, itemId, lotCode)` 唯一聚合；无批次物料使用 `NO_LOT`。
- 每次过账都在 **同一个 PostgreSQL 事务**中：验证来源单据与库存条件 → 使用 idempotencyKey 去重 → 写库存流水 → 原子增减库存余额 → 记录审计。
- 在数据库层增加 `CHECK (quantity_on_hand >= 0)` 的同等约束（字段实际名称与迁移脚本保持一致）；数量差额不能等于 0，入库类型必须是正数，出库类型必须是负数。
- 更新库存余额时使用原子 SQL 或条件更新 / 行锁控制并发，不能先读后在应用层做非原子的加减运算。
- `lotCode` 为简单批次标识，尚不是完整的供应商批次追溯 / 批次主数据模块。

## 4. 单据追踪

- 采购收货：InventoryTransaction.PURCHASE_RECEIPT → purchaseOrderLineId。
- 生产领料 / 退料：PRODUCTION_ISSUE / PRODUCTION_RETURN → productionOrderMaterialId + productionOrderId。
- 完工入库：PRODUCTION_RECEIPT → productionOrderId，入库物料必须是工单成品。
- 销售发货：SALES_SHIPMENT → salesOrderLineId。
- 所有操作必须使用可重复提交的业务幂等键 `idempotencyKey`。
- 不同凭证类型的来源字段合法性、SKU 一致性、状态门禁均由领域服务验证，并应补充数据库约束或触发器强化。

## 5. 最低验收用例

1. 同一幂等键重复提交入库请求：只过账一次。
2. 两个工单同时领用最后一件库存：只能有一个成功，不允许库存为负。
3. 工单下达后修改 BOM 版本：历史工单快照不被改写。
4. 采购 100 件，工单领用 30 件、退回 2 件，成品入库 10 件：原材料余额正确，且各自流水可追踪。
5. 销售出库不能超过对应仓库可用成品库存；批次库存不可串用。
6. 禁止删除已过账库存流水；冲销必须以新的反向流水实现。

## 6. 暂未交付

前端、后端 API、自动化测试、数据库 migration、生产部署、实时车间报工及复杂的 MRP、成本核算、财务总账均不在本原型交付范围。