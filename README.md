# 亚马逊运营绩效考核方案（工程化版）

本 README 为运营绩效代码库的设计蓝图，涵盖考核逻辑与指标、参数配置、数据来源、实现路径、数据存储，以及后续调整维护方法，确保各模块可快速落地与扩展。

---

## 1. 方案目标
- 建立兼顾 **库存健康 / 销售增长 / 投入产出** 的量化评价体系。
- 通过自动化数据链路减少人工对账、提升考核透明度。
- 为人力激励、预算规划、补货排程提供客观依据，并支持多站点复制。

## 2. 考核逻辑 & 指标体系
### 2.1 维度与权重
| 维度 | 核心指标 | 权重 | 考核逻辑 |
| --- | --- | --- | --- |
| 库存规划 | 需求预测达成率 | 20% | 衡量预测模型准确性，鼓励“备得准、备得稳”。 |
| 库存健康 | FBA/I F C 库存健康度 | 35% | 关注 WOS、超龄库存、补货执行率，防止资金占压。 |
| 销售结果 | 父体/ASIN 销售额达成率 | 35% | 以父体结构 + 子体定位权重计算 GMV 表现。 |
| 投入产出 | 广告 ROAS & 结构质量 | 10% | 检查广告 ROI 与预算分层是否符合策略。 |

> 权重可在 `config/weights.yml` 中按事业部需求进行重载。

### 2.2 KPI 评分
- **需求预测达成率**：`实际销量 ÷ 预测销量`，区间 95%~105% 为满分；超出范围按误差梯度扣分，缺货产生额外惩罚 ×1.5。
- **库存健康**：在 WOS（2~9 周）与超龄库存（<50 件）条件下得 100 分；偏离阈值按 SKU 占比扣分。
- **销售额达成率**：`Σ(子 ASIN 达成率 × 类型权重) ÷ 父体目标`，再乘以运营负责父体的 GMV 占比；≥105% 触发奖励。
- **广告 ROAS**：`广告销售额 ÷ 广告花费`，同时检查品牌词/类目词/ASIN 定投占比；未达标时按 10% 和 20% 两档扣分。

评分脚本位于 `pipelines/scorecard.py`，可在 CI 中按月或按周运行。

## 3. 参数设置
- `config/weights.yml`：维度权重、SKU 定位权重（引流 30%、畅销 50%、利润 20%）。
- `config/targets.yml`：站点/品类/父体的 GMV 目标、预测阈值、ROAS 目标。
- `config/penalties.yml`：缺货惩罚倍数、超龄扣分曲线、ROAS 扣分梯度。
- 所有参数使用 **Git 版本化 + 环境变量覆盖** 的方式管理，允许针对特定站点在 `.env` 中注入临时阈值。

## 4. 数据来源与流向
1. **原始数据**  
   - OMS/WMS：订单、发货、取消、退货。  
   - FBA API：库存量、超龄、补货建议。  
   - ERP/Planning：预测、目标表、SKU 定位。  
   - Amazon Ads / DSP：广告消耗、销售额。  
2. **采集调度**  
   - 使用 Airflow/Prefect（见 `pipelines/jobs/`）按 UTC+8 06:00、12:00、18:00 三个时段拉取。  
   - 数据落地 S3/OSS “bronze” 层，带时间戳分区。  
3. **处理**  
   - `pipelines/transform/` 内含 dbt/Spark 脚本，将原始表聚合成 `silver`（标准化指标）与 `gold`（考核结果）。  
   - `notebooks/validation/` 提供抽样校验。

流程示意：源系统 → Ingestion Job → Bronze Storage → Transform（dbt/Spark）→ Gold Table → Scorecard & Dashboard。

## 5. 实现路径
1. **数据链路部署**  
   - 配置 Airflow DAG `dag_amz_perf.py`，拉取源数据。  
   - 通过 dbt `models/kpi_*` 计算 KPI 中间表。  
2. **指标计算**  
   - 运行 `python pipelines/scorecard.py --period 2025-10 --site US`，生成 `scorecard_{site}_{period}.parquet`。  
3. **可视化/报表**  
   - `dashboard/` 存放 Looker Studio/Power BI 模板，直接连接 `gold.scorecard`。  
4. **验收与回滚**  
   - 所有作业加 `tests/` 内单元测试。  
   - 若评分异常，可通过 `pipelines/recompute.py --period ...` 重算并写入版本号。

## 6. 数据存储设计
- **Bronze**：原始日志（Parquet/CSV），按 `source=date/hour` 分区；保留 90 天。  
- **Silver**：清洗后的订单、库存、广告事实表，按 `site`、`period` 分区；保留 18 个月。  
- **Gold**：`gold.scorecard`（月度）、`gold.weekly_alert`（周度红黄灯）。  
- 元数据登记在 `docs/data_dictionary.md`，并与 Data Catalog 连接以供血缘追踪。

## 7. 后续调整与维护
1. **参数调整**  
   - 修改配置文件后提交 PR，附带影响评估。  
   - 合并后 CI 自动触发 `config_smoke_test`，确保阈值合法。  
2. **指标更新**  
   - 若新增 KPI，需同步更新 dbt 模型、scorecard 脚本与看板字段。  
   - 变更记录写入 `docs/changelog.md`。  
3. **数据监控**  
   - 通过 `pipelines/monitor.py` 监控延迟、缺失、值域异常，输出到 Slack/Webhook。  
4. **权限与安全**  
   - 仅数据团队可访问 Bronze/Silver；运营同学通过 BI 工具读取 Gold。  
5. **灾备**  
   - 每日 23:00 进行快照备份，支持 30 天回滚。

## 8. 项目结构（建议）
```
amz-operation-performance-assessment/
├── README.md
├── config/                  # 权重、目标、惩罚等参数
├── data/                    # 本地样例数据（勿提交生产数据）
├── docs/                    # 数据字典、变更记录
├── notebooks/               # 验证与探索
├── pipelines/
│   ├── jobs/                # Airflow/Prefect 任务
│   ├── transform/           # dbt/Spark 模型
│   ├── scorecard.py         # 评分脚本
│   └── monitor.py           # 运行监控
├── dashboard/               # BI 模板
└── tests/                   # 单元与集成测试
```

---
如需扩展站点或对接新的 BI/AI 模块，可在 `docs/` 中增补实施方案，并通过配置中心无缝切换参数。欢迎提交 Issue/PR 与我们讨论更优的考核策略。*** End Patch
