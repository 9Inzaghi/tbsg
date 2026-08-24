# 哪些信息在 YAML 里，文件如何组织？

**YAML 承载了"定义性元数据"（Definition Metadata），即业务逻辑、模型结构、计算规则和安全策略；而"运行态元数据"（Runtime Metadata）和"血缘/日志元数据"通常存储在数据库或缓存中。**

关于**文件组织策略**，如果把所有内容塞进一个巨大的 `config.yaml`，那是灾难性的（难以维护、冲突频繁、加载慢）。现代语义平台（如 dbt, Cube.js, Transform）都遵循 **"高内聚、低耦合"** 的模块化原则来拆分文件。

以下是详细的**文件拆分指南**和**架构建议**：

---

### 一、哪些信息在 YAML 里？哪些不在？

#### ✅ 在 YAML 里的（定义性/静态元数据）

这些是"代码"，需要版本控制（Git）：
1. **数据源连接配置**（部分敏感信息除外，如密码用环境变量）。
2. **物理表映射**：表名、字段类型、主键定义。
3. **关联关系 (Joins)**：表与表如何连接。
4. **维度定义**：维度名称、类型、层级、同义词。
5. **指标定义**：计算公式、过滤条件、时间智能规则。
6. **业务描述**：用于 AI 理解的 Description、Aliases、Examples。
7. **安全策略**：行级权限规则、列级脱敏规则。
8. **测试用例**：数据质量断言。

#### ❌ 不在 YAML 里的（运行态/动态元数据）

这些是"状态"，由引擎运行时生成并存储在数据库/Redis 中：
1. **查询缓存**：热点查询的结果。
2. **实时血缘图谱**：虽然依赖关系在 YAML 里定义了，但解析后的完整 DAG 图通常存在图数据库或关系型数据库中，方便快速查询影响范围。
3. **使用统计/热度**：哪个指标被查了多少次（用于推荐或下线）。
4. **执行日志**：SQL 执行耗时、错误堆栈。
5. **物化视图状态**：MV 是否构建成功、最后刷新时间。
6. **用户会话信息**。

---

### 二、YAML 文件如何拆分？（最佳实践）

核心原则：**按"业务域 (Domain)"和"对象类型 (Object Type)"进行二维拆分。**

不要建一个 `all_metrics.yaml`，而要建立清晰的目录树。

#### 推荐目录结构示例

```text
semantic_layer/
├── project.yaml              # [全局配置] 项目名称、默认数据源、环境变量
├── packages.yaml             # [依赖管理] 引用的公共包（如通用时间智能库）
│
├── sources/                  # [数据源层] 定义物理表映射
│   ├── sales_source.yaml     # 销售域物理表
│   ├── user_source.yaml      # 用户域物理表
│   └── product_source.yaml   # 商品域物理表
│
├── models/                   # [模型层] 定义逻辑宽表/Join 关系
│   ├── sales_model.yaml      # 销售主题逻辑模型 (Fact + Dims)
│   ├── marketing_model.yaml  # 营销主题逻辑模型
│   └── finance_model.yaml    # 财务主题逻辑模型
│
├── metrics/                  # [指标层] 定义具体指标 (可按域细分)
│   ├── sales/                # 销售域指标
│   │   ├── gmv.yaml          # GMV 相关指标
│   │   ├── order_count.yaml  # 订单量相关指标
│   │   └── conversion.yaml   # 转化率相关指标
│   ├── user/                 # 用户域指标
│   │   ├── active_users.yaml
│   │   └── ltv.yaml
│   └── shared/               # 公共复用指标 (如汇率转换逻辑)
│       └── currency_rates.yaml
│
├── dimensions/               # [维度层] (可选，若未在 model 中定义)
│   ├── time_dim.yaml         # 时间维度特殊配置
│   └── geo_dim.yaml          # 地理维度层级
│
├── security/                 # [安全层] 权限策略
│   ├── rls_rules.yaml        # 行级权限规则
│   └── masking_rules.yaml    # 脱敏规则
│
├── tests/                    # [测试层] 数据质量测试
│   ├── critical_tests.yaml   # 核心指标阻断性测试
│   └── warning_tests.yaml    # 警告级测试
│
└── ai_context/               # [AI 增强层] 专门给 LLM 看的上下文
    ├── glossary.yaml         # 全局业务术语表
    └── few_shot_examples.yaml# 典型问答示例库

```

---

### 三、拆分的具体逻辑与规则

#### 1. 按"业务域"拆分 (Domain Separation)
- **规则**：不同业务线（销售、财务、供应链）的文件必须物理隔离。
- **原因**：
    - **权限隔离**：销售团队只关心 `sales/` 目录，财务团队只关心 `finance/`。
    - **减少冲突**：销售团队修改 `gmv.yaml` 不会导致财务团队的 PR 冲突。
    - **加载优化**：引擎可以按需加载特定域的元数据，无需全量解析。

#### 2. 按"对象粒度"拆分 (Granularity)
- **规则**：
    - **Sources/Models**：按主题域分文件（一个域一个大文件，因为表之间关联紧密）。
    - **Metrics**：**强烈建议按指标组或单个重要指标分文件**。
        - *为什么？* 指标数量最多，变更最频繁。如果把 1000 个指标写在一个文件里，每次改一个都要重新加载整个大文件，且 Git Diff 很难看。
        - *做法*：`gmv.yaml` 里只放 GMV、净销售额、退款率等强相关的 3-5 个指标。

#### 3. 公共 vs. 私有 (Shared vs. Private)
- **规则**：
    - **基础原子指标**（如 `sum(amount)`）放在各自域的文件里。
    - **跨域复用逻辑**（如 `exchange_rate` 汇率转换、`tax_rate` 税率）放在 `shared/` 或 `packages/` 目录下，供其他文件 `import` 或 `ref`。

#### 4. AI 上下文独立 (AI Context Isolation)
- **规则**：将通用的业务术语、同义词、Few-Shot 示例单独放在 `ai_context/`。
- **原因**：这些内容不参与 SQL 生成逻辑，只参与 Prompt 构建。独立存放方便 NLP 专家单独维护和优化，而不干扰数据工程师的逻辑开发。

---

### 四、文件内部结构示例

以 `metrics/sales/gmv.yaml` 为例，展示一个文件里应该包含什么：

```yaml
# metrics/sales/gmv.yaml
version: 1

metrics:
  - name: gmv_total
    description: "总交易额，包含所有已支付订单，不含运费"
    label: "GMV 总额"
    type: ratio # 或 sum, count
    type_params:
      measure: order_amount_sum
      numerator: order_amount_sum
      denominator: 1 # 简单聚合
    filter: |
      {{ Dimension('order.status') }} = 'paid'
    tags: ['core', 'sales', 'ai-high-priority']
    aliases: ["销售额", "成交金额", "营收"]
    
    # AI 专属：针对该指标的示例问题
    ai_examples:
      - question: "上个月的销售总额是多少？"
        expected_sql_snippet: "SUM(order_amount) WHERE ..."
      
  - name: gmv_yoy_growth
    description: "GMV 同比增长率"
    type: derived
    formula: "(gmv_total - gmv_total_prev_year) / gmv_total_prev_year"
    depends_on:
      - ref('gmv_total')
      - ref('gmv_total_prev_year') # 引用时间智能变体

```

---

### 五、总结：如何管理这么多文件？

既然拆成了几十甚至上百个 YAML 文件，如何管理？
1. **引用机制 (Ref System)**：
    - 文件之间通过 `ref('metric_name')` 或 `ref('model_name')` 相互引用，而不是硬编码名字。引擎在启动时会解析整个依赖图。
2. **CI/CD 流水线**：
    - **Lint 检查**：提交时自动检查 YAML 语法、命名规范。
    - **单元测试**：自动运行 `tests/` 下的断言，确保新指标逻辑正确。
    - **影响分析**：PR 提交时，自动分析改动影响了哪些下游报表，并通知相关负责人。
3. **模块化导入**：
    - 在 `project.yaml` 中配置 include 路径，例如 `include: ['models/*.yaml', 'metrics/**/*.yaml']`，支持通配符批量加载。

**核心结论**： **不要把 YAML 当作"配置文件"，要把它当作"源代码"来管理。** 像管理 Java/Python 项目一样管理语义层 YAML：**分模块、分目录、有依赖、有测试、有版本控制**。这样既能承载海量元信息，又能保持系统的可维护性和扩展性。
