# 小型体育器材制造 ERP / MES 数据模型原型

这是用于真实项目讨论和工程启动的 **数据库模型原型**，不是完整的 ERP 成品，也不是已有可运行的生产系统。

## 技术范围

- PostgreSQL + Prisma ORM 7.x + TypeScript
- 第一阶段：采购、物料、BOM、生产工单、库存事务、销售出库、审计
- 14 个实体模型，7 个领域枚举；面向轻量级模块化单体，不先拆微服务

## 文件结构

```text
sports-equipment-erp-starter/
├── prisma/schema.prisma         # 真实 Prisma ORM 数据模型
├── prisma.config.ts             # Prisma 7 数据源配置
├── package.json                 # CLI 依赖及命令
├── .env.example                 # 本地 PostgreSQL 配置样例
├── docs/business-rules.md       # 领域业务规则与一致性设计
└── README.md
```

## 核心业务链路

采购订单 → 原材料入库（InventoryTransaction + InventoryBalance） → BOM 版本 → 生产工单 → BOM 快照 → 工单领料 → 组装/完工 → 成品入库 → 销售出库。

举例：某训练器材成品 SKU `SPORT-TRAIN-001` 的 BOM 可以包含钢管、握把、紧固件、包装材料；生产工单创建时关联 BOM，**下达时**将各项物料用量复制到 `ProductionOrderMaterial`，再根据真实库存流水记录领退料、完工及发货。

## 本地启动（需要 Node.js 与 PostgreSQL）

```bash
npm install
cp .env.example .env
# 编辑 .env，填入真实的本地数据库用户名/密码
npm run db:validate
npm run db:migrate -- --name init
npm run db:generate
```

上面的命令需要联网安装依赖及一个可连接的 PostgreSQL 数据库。本仓库只含数据模型与基础 CLI 配置；**未在此处运行 migration，也未提供前后端服务**。实际使用前还需要完成业务 API、事务一致性、权限、测试与部署。

## 推荐 Devin 第一阶段任务

1. 审查 Prisma 数据模型，完成首个 migration，并补齐约束（数量大于 0、库存不能为负、相同产品唯一活动 BOM 等）。
2. 开发 `InventoryPostingService`，在单个数据库事务中创建幂等库存流水并原子更新库存余额。
3. 开发 BOM 版本及工单下达逻辑，生成不可随 BOM 后续修改而变化的工单物料快照。
4. 按业务规则补充状态机、角色权限、并发扣库与重复请求自动化测试。
5. 在开发环境验证“采购入库 → 工单领料 → 完工入库 → 销售出库”端到端路径。

请不要用本原型声称代码已部署或系统已上线。