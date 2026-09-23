# Schema Dowser · 表格探矿仪

<p align="center">
  <a href="https://github.com/CJX0712/schema-dowser/actions/workflows/ci.yml"><img src="https://github.com/CJX0712/schema-dowser/actions/workflows/ci.yml/badge.svg" alt="ci"></a>
  <a href="https://github.com/CJX0712/schema-dowser/releases"><img src="https://img.shields.io/github/v/release/CJX0712/schema-dowser?sort=semver" alt="release"></a>
  <a href="https://github.com/CJX0712/schema-dowser/blob/main/LICENSE"><img src="https://img.shields.io/github/license/CJX0712/schema-dowser" alt="license"></a>
  <img src="https://img.shields.io/badge/author-%E6%99%A8%E6%98%9F-1f6feb" alt="author">
</p>

把一堆 CSV 丢进去，探出**隐藏的键、依赖、脏数据，以及本该拆出去的表**。

单文件 HTML，零依赖，全部计算在浏览器内存里完成 —— 数据不联网、不上传、不落盘。

```
双击 index.html  →  点「载入示例」  →  看它把 18 列订单表拆成 customers / cities / coupons
```

---

## 它解决什么问题

你拿到一张几千行的 CSV，要把它落成数据库表。这时你其实在猜四件事：

1. 主键是什么？
2. 哪些列是冗余的（值完全由别的列决定）？
3. 哪些列该拆成独立的表？
4. 数据里有哪些坑（离群值、写法不统一、缺失模式）？

人工看头 20 行能猜个大概，但 2000 行里那条「`city` 其实能决定 `country`」的传递依赖，
或者「`total` 就是 `unit_price × quantity × (1 - discount)`」这种派生关系，肉眼基本看不出来。

Schema Dowser 就是把这四件事自动化，并且**给出理由**。

---

## 四层结论

### 1 · 列剖面

每列一张卡：推断类型（整型 / 浮点 / 布尔 / 日期 / 时间 / 邮箱 / URL / UUID / IP / 枚举 / 文本）、
基数、唯一率、缺失率、Top 取值、数值分位与离群、日期跨度。

### 2 · 函数依赖

形如 `customer_id → email`，读作「customer_id 一旦确定，email 就唯一确定」。

- **精确依赖**：零违反
- **近似依赖**：少量违反，通常是脏数据造成。若归一化（忽略大小写与首尾空格）后成立，
  会明确标出「写法差异」—— 数据意图上是依赖，只是写法不统一
- **平凡依赖**：决定因子本身是键，键决定一切，单独折叠
- **高基数巧合**：已自动识别并忽略，见下方「算法边界」

### 3 · 候选键

能唯一标识一行的最小列组合。找不到就说明数据里有完全重复的行，会直接提示。

### 4 · SQL DDL 与规范化建议

按数据反推出可直接执行的建表语句（PostgreSQL / MySQL / SQLite 三种方言）：

```sql
CREATE TABLE imported_data (
  order_id TEXT                   NOT NULL,
  customer_id TEXT                NOT NULL,  -- 由 email 完全决定
  total DOUBLE PRECISION          NOT NULL,  -- 可由 unit_price + quantity + discount 计算得出
  ...
  PRIMARY KEY (order_id),
  UNIQUE (order_date),
  CONSTRAINT chk_status CHECK (status IN ('paid','shipped','pending','refunded','cancelled'))
);

CREATE TABLE customers ( ... PRIMARY KEY (customer_id) );
```

同时指出部分依赖与传递依赖，建议该拆出哪些表，并给出外键语句。

---

## 算法说明

| 环节 | 做法 |
|---|---|
| CSV 解析 | 手写状态机，处理引号、转义双引号、字段内换行、BOM、CRLF、自动探测分隔符 |
| 类型推断 | 逐值正则投票（整型 / 浮点 / 布尔 / 日期 / 邮箱 / URL / UUID / IP），再按基数与重复率判定枚举 |
| 依赖发现 | 按决定因子取值分桶，桶内被决定列取值唯一则成立；逐层加维度（1→2→3），已成立的不再扩展 |
| 候选键 | 组合逐层搜索 + 超集剪枝 |
| 相关性 | 数值列取 \|Pearson r\|（跳过缺失），其它列取归一化互信息，统一归一到 0–1 |
| 异常检测 | IQR 1.5× / 3× 离群、z-score、负值、日期越界、首尾空白、大小写变体、长尾取值、格式混写 |

### 三处非显然的处理

**1 · 高基数剪枝**
若决定因子 X 的桶数超过行数一半（平均不到 2 行一个桶），X→Y 基本是过拟合的巧合而非结构。
这类依赖会被标记 `weak` 并从结论中剔除 —— 否则一个高基数列（如订单时间）能"决定"几乎所有其它列。

**2 · 归一化后的依赖**
`customer_id → customer_name` 若因大小写、首尾空格而违反，直接丢弃就丢掉了真实结构；
全盘接受又会污染结果。这里单独跑一遍归一化编码，只对"归一化后成立"的标为近似依赖并注明原因。

**3 · 等价列合并**
`customer_id`、`email`、`signup_date` 常常互相决定 —— 它们是同一个实体的三把钥匙。
不合并的话会被当成三个不同的实体，建议拆出三张重复的表。这里用并查集归并等价类，
再按「名字像不像 id」选代表列。

### 算法边界（诚实的说明）

- **函数依赖只能证明"这批数据里成立"**。换一批数据可能不成立，尤其是低频取值的列。
- 数值列当决定因子产生的依赖（如 `unit_price → status`）在数学上成立但多半是巧合，
  规范化建议会跳过，依赖列表里仍会列出 —— 请按业务常识判断。
- 维度 ≥3 时只在**数值列**里搜索（派生列几乎都是数值算出来的），否则组合数爆炸。
- 默认采样上限 4000 行。依赖发现是 O(列² × 行)，超过请调低维度或行数。

---

## 导出

- **Markdown 报告**：概览、列剖面表、依赖清单、候选键、质量告警、完整 DDL
- **SQL 文件**：`.sql`
- **依赖图**：SVG / PNG（2×）

---

## 使用

- 粘贴 CSV/TSV，或拖入 `.csv` 文件，或点「载入示例」
- 选项：分隔符（默认自动识别）、首行是否为表头、依赖维度（1–3）、采样上限
- 浏览器：Chrome / Edge / Firefox / Safari 近两年版本。无构建步骤，无外部请求

---

## License

MIT · 见 [LICENSE](LICENSE)
