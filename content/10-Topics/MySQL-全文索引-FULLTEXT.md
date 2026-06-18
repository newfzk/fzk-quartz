---
title: MySQL 全文索引 — FULLTEXT
aliases:
  - FULLTEXT Index
  - 全文索引
  - 倒排索引
  - MySQL Full-Text Search
tags:
  - language/sql
  - topic/MySQL
  - topic/数据库/索引
status: to-review
---

# MySQL 全文索引 — FULLTEXT

FULLTEXT 索引是 MySQL 中专用于**全文检索（Full-Text Search）**的索引类型。它通过**倒排索引（Inverted Index）**实现对文本内容的快速关键词搜索，是替代 `LIKE '%keyword%'` 的高效方案。

---

## 与其他索引的核心区别

| 维度 | B+Tree 索引 | FULLTEXT 索引 |
|------|------------|--------------|
| **底层结构** | B+Tree（有序平衡树） | 倒排索引（词→文档列表映射） |
| **匹配方式** | 前缀匹配（`LIKE 'abc%'`） | 分词匹配（关键词/短语搜索） |
| **适用场景** | 精确匹配、范围查询、排序 | 文本内容搜索、文章检索 |
| **查询语法** | `WHERE col LIKE '...'` | `MATCH(col) AGAINST('...')` |
| **排序能力** | 需额外 ORDER BY | 内置相关性评分排序 |
| **大数据量性能** | 全表 `LIKE` 无法走索引 | 可高效并行搜索大文本 |

> [!tip] LIKE '%keyword%' 为什么慢
> `LIKE '%keyword%'` 前缀通配导致 B+Tree 无法从中间导航，必须**全表扫描 + 逐行匹配**。而 FULLTEXT 通过倒排索引直接定位包含该关键词的行，性能数量级优于 `LIKE`。详见 [[MySQL索引创建原则#五、面试高频场景：什么情况下索引会失效]]

---

## 工作原理

### 1. 分词（Tokenization）

建索引时，MySQL 将文本按照**分词规则**拆分为独立的词（Token）：

```sql
-- 原始文本
"The quick brown fox jumps over the lazy dog"

-- 分词结果（去除停用词后）
['quick', 'brown', 'fox', 'jumps', 'over', 'lazy', 'dog']
```

### 2. 倒排索引结构

```
┌─────────────┬──────────────────────────┐
│    word     │    document_list         │
├─────────────┼──────────────────────────┤
│ quick       │ row_id_1, row_id_5       │
│ brown       │ row_id_1                 │
│ fox         │ row_id_1, row_id_3       │
│ jumps       │ row_id_1, row_id_7       │
│ lazy        │ row_id_1, row_id_3       │
│ dog         │ row_id_1                 │
│ database    │ row_id_2, row_id_4       │
│ index       │ row_id_2, row_id_6       │
└─────────────┴──────────────────────────┘
```

- **词（Word）**：分词后的关键词
- **文档列表（Document List）**：包含该词的行记录 ID 列表
- 每个词还附加**位置信息**和**词频**，用于相关性评分

> [!info] 对比 B+Tree
> B+Tree 是"从记录到列值"的正向映射（给定范围，找到记录），而倒排索引是"从词到记录"的反向映射（给定词，找到所有包含它的记录）。这正是"倒排"名称的由来。

### 3. 相关性评分

MySQL 默认使用 **TF/IDF 算法** 对搜索结果排序（MySQL 8.0 起支持 [[#BM25 算法（MySQL 8.0+）|BM25]]）：

$$score = \sum_{term} \frac{1}{1 + \log(df_d)} \cdot \log\left(\frac{N}{df_t}\right)$$

- **TF（Term Frequency）**：词在文档中出现的频率 → 出现越多越相关
- **IDF（Inverse Document Frequency）**：词在整个文档集合中的稀缺度 → 越稀缺权重越高
- **N**：文档总数
- **df_t**：包含该词的文档数

> [!info] TF-IDF 的本质
> 结果按**相关性降序**返回，相关性高的结果靠前。如果一个词在很多文档中都出现（如 "the"），IDF 很低 → 对评分贡献小。如果一个词很少出现（如 "MySQL"），IDF 很高 → 匹配到的文档排名靠前。

---

## 语法与使用

### 创建 FULLTEXT 索引

```sql
-- 建表时定义
CREATE TABLE article (
    id INT PRIMARY KEY,
    title VARCHAR(200),
    body TEXT,
    FULLTEXT idx_fulltext (title, body)   -- 可联合多列
) ENGINE=InnoDB;

-- 已存在的表添加
CREATE FULLTEXT INDEX idx_fulltext ON article(title, body);

-- 或使用 ALTER
ALTER TABLE article ADD FULLTEXT INDEX idx_fulltext (title, body);
```

> [!info] 多列全文索引
> FULLTEXT 索引可以包含多个列（如上例的 `title, body`），查询时 `MATCH` 必须指定相同的列集合，否则索引无法使用。

### 查询语法

```sql
SELECT * FROM article
WHERE MATCH(title, body) AGAINST('database index' IN NATURAL LANGUAGE MODE);
```

`MATCH` 的列列表必须与 FULLTEXT 索引定义完全一致。

---

## 三种搜索模式

### 1️⃣ 自然语言模式（NATURAL LANGUAGE MODE）**默认**

```sql
-- 默认就是自然语言模式
SELECT id, title, MATCH(body) AGAINST('database index') AS relevance
FROM article
WHERE MATCH(body) AGAINST('database index');

-- 等价于
WHERE MATCH(body) AGAINST('database index' IN NATURAL LANGUAGE MODE);
```

**特点：**
- 返回按相关性评分降序排列的结果
- MySQL 自动过滤**停用词（Stopwords）**
- 长度低于 `innodb_ft_min_token_size` 的词被忽略（默认 3）
- 表中行数少于一定数量（约 50% 行匹配）时可能返回空结果（即**50% 阈值**，InnoDB 已取消此限制）

### 2️⃣ 布尔模式（BOOLEAN MODE）

```sql
SELECT * FROM article
WHERE MATCH(body) AGAINST('+database -nosql' IN BOOLEAN MODE);
```

**操作符：**

| 操作符 | 含义 | 示例 | 说明 |
|--------|------|------|------|
| `+` | 必须包含 | `+database` | 结果必须包含 database |
| `-` | 必须排除 | `-nosql` | 结果不能包含 nosql |
| `(空)` | 可选，但出现则排名更高 | `database index` | 包含任意词即返回，含多个词排名更高 |
| `>` | 提高该词权重 | `>database` | 包含该词会显著提高排名 |
| `<` | 降低该词权重 | `<nosql` | 包含该词会降低排名 |
| `*` | 通配符（后缀） | `data*` | 匹配以 data 开头的词（如 database, data） |
| `""` | 精确短语 | `"database index"` | 精确匹配整个短语 |
| `()` | 子表达式分组 | `+(database index)` | 括号内操作符优先 |

**组合示例：**

```sql
-- 必须包含 "MySQL"，可以包含 "index"（有则加分），不能包含 "Oracle"
SELECT * FROM article
WHERE MATCH(title, body)
AGAINST('+MySQL index -Oracle' IN BOOLEAN MODE);

-- 必须包含 "database"，同时匹配以 "index" 开头的词
SELECT * FROM article
WHERE MATCH(title, body)
AGAINST('+database index*' IN BOOLEAN MODE);

-- 精确短语匹配
SELECT * FROM article
WHERE MATCH(title, body)
AGAINST('"inverted index"' IN BOOLEAN MODE);
```

### 3️⃣ 查询扩展模式（QUERY EXTENSION）

```sql
SELECT * FROM article
WHERE MATCH(body) AGAINST('database' WITH QUERY EXPANSION);
```

**执行流程（两阶段搜索）：**

1. **第一阶段**：使用原始关键词（`database`）进行全文搜索，找出相关文档
2. **第二阶段**：从第一阶段结果中提取高频且相关的词，组成新查询再搜索一次

> [!warning] 谨慎使用查询扩展
> 查询扩展会**显著增加结果数量**，可能引入大量不相关内容。适用于"找相关文档"的场景但噪声较大。建议仅在搜索结果太少时使用。

---

## BM25 算法（MySQL 8.0+）

MySQL 8.0 起，InnoDB 全文索引引入 **BM25（Best Matching 25）** 算法作为相关性评分选项，相比传统的 TF/IDF 更为精准：

```sql
-- 查看当前相关性算法（默认或变量）
SHOW VARIABLES LIKE 'innodb_ft_use_bm25';
```

BM25 引入了两个可调参数：
- **k1**（默认 0.5）：控制词频饱和程度，越大词频影响越大
- **b**（默认 0.6）：控制文档长度归一化，b=1 时完全归一化，b=0 时忽略文档长度

BM25 相比 TF/IDF 的优势：
- 避免长文档天然获得更高词频的不公平
- 词频非线性增长（饱和函数），防止高频词的过度影响
- 在长文档上的表现更优

---

## 中文全文检索（n-gram 解析器）

MySQL 默认采用**空格/标点分词**，不适用于中文。MySQL 5.7.6+ 提供了内置的 **n-gram 全文解析器（ngram parser）** 来支持中、日、韩（CJK）语言。

### 创建 n-gram 全文索引

```sql
-- 使用 ngram 解析器创建全文索引
CREATE TABLE article (
    id INT PRIMARY KEY,
    title VARCHAR(200),
    body TEXT,
    FULLTEXT INDEX idx_body (body) WITH PARSER ngram
) ENGINE=InnoDB;

-- 或对已有表
CREATE FULLTEXT INDEX idx_body ON article(body) WITH PARSER ngram;
```

### n-gram 原理

n-gram 将文本按**连续 n 个字符**为单位进行切分：

```sql
-- ngram_token_size = 2（默认值）
-- 原文："数据库索引"
-- 分词结果：['数据', '据库', '库索', '索引']

-- ngram_token_size = 3
-- 原文："数据库索引"
-- 分词结果：['数据库', '据库索', '库索引']
```

> [!tip] ngram_token_size 选择
> - `ngram_token_size=2` → 更细粒度，召回率更高，但可能产生噪声
> - `ngram_token_size=3` → 更精准，但可能漏掉 2 字词
> - 默认值为 **2**，在大多数场景下表现均衡
> - 可在配置文件或启动参数中设置：`ngram_token_size=2`

### 查询 n-gram 全文索引

```sql
SELECT * FROM article
WHERE MATCH(body) AGAINST('数据库' IN BOOLEAN MODE);
```

MySQL 会将查询词 `数据库` 按同样的 n-gram 规则拆分为 `'数据 据库 库索 索引'`（n=2）后进行匹配。

> [!warning] n-gram 的限制
> - ngram 解析器**不识别语义**，纯按字符滑动窗口切分，会产生无意义的 Token（如 `库索`）
> - 对于专业术语或生僻词，可能需要搭配布尔模式精确短语搜索
> - 真正的中文分词需要专业搜索引擎（如 [[Elasticsearch ES]] + IK 分词器）

---

## 配置参数

### InnoDB 全文索引相关参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `innodb_ft_min_token_size` | 3 | 索引的最小词长度（InnoDB），可设为 1 支持单字母搜索 |
| `innodb_ft_max_token_size` | 84 | 索引的最大词长度 |
| `innodb_ft_server_stopword_table` | 内置停用词表 | 自定义停用词表（InnoDB），指向一个包含 `value` 列的表 |
| `innodb_ft_user_stopword_table` | — | 当前会话级别的停用词表 |
| `innodb_ft_enable_stopword` | ON | 是否启用停用词过滤 |
| `innodb_ft_use_bm25` | OFF | 是否使用 BM25 相关性算法（MySQL 8.0+） |
| `ngram_token_size` | 2 | n-gram 分词窗口大小（使用 ngram parser 时） |

### MyISAM 全文索引相关参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `ft_min_word_len` | 4 | 索引的最小词长度（MyISAM） |
| `ft_max_word_len` | 84 | 索引的最大词长度 |
| `ft_stopword_file` | 内置 | 停用词文件路径 |
| `ft_boolean_syntax` | `+ -><()~*:""&|` | 布尔搜索操作符语法 |

> [!tip] 修改参数后需要重建索引
> 修改 `innodb_ft_min_token_size` 等参数后，需 `REPAIR TABLE` 或 `OPTIMIZE TABLE` 重建 FULLTEXT 索引才会生效。

---

## 性能考量与局限

### 适用场景

- 博客文章、新闻内容的站内搜索
- 产品名称／商品描述的关键词匹配
- 小型 CMS 系统的文本检索
- 数据量在**千万级以下**的文本搜索

### 不适用场景

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 高并发实时搜索（毫秒级） | [[Elasticsearch ES]] / Solr | MySQL FULLTEXT 在并发高、数据量大时性能下降明显 |
| 全文搜索是核心功能 | 专业搜索引擎 | 分词精度、聚合分析、分布式能力远不如 ES |
| 海量数据 (>1 亿行) | Elasticsearch | 倒排索引在分布式引擎中更高效，支持水平扩展 |
| 高级中文分词需求 | ES + IK 分词器 | n-gram 纯字符滑动，无语义理解 |
| 多维度组合搜索 | Elasticsearch | ES 的 DSL 支持结构化 + 全文搜索的组合查询 |

### 性能注意事项

```sql
-- ✅ 良好实践：使用覆盖索引避免回表
-- 如果 FULLTEXT 在 body 列，查询仅返回 id
-- (id 在主键中，无需回表)
SELECT id FROM article
WHERE MATCH(body) AGAINST('database');

-- ⚠️ 回表查询：需要额外数据时
SELECT id, title, body FROM article   -- body 长文本，回表拉取
WHERE MATCH(body) AGAINST('database');
```

- **回表查询**：FULLTEXT 索引找到匹配行的主键后，如果 SELECT 需要更多列，需回聚簇索引获取完整数据
- **DML 性能**：FULLTEXT 索引维护成本高，高频 INSERT/UPDATE 场景写入会变慢
- **内存占用**：倒排索引在内存中的缓存开销较大，需关注 `innodb_buffer_pool_size`

---

## 常见问题

### Q1：为什么 MATCH 匹配不到结果？

```sql
-- 可能原因排查清单
-- 1️⃣ 检查最小词长限制（默认 3，短词 'is'、'go' 被忽略）
SHOW VARIABLES LIKE 'innodb_ft_min_token_size';

-- 2️⃣ 检查是否有停用词
-- 'the'、'a'、'an' 等常见词被过滤
SHOW VARIABLES LIKE 'innodb_ft_enable_stopword';

-- 3️⃣ 检查索引是否已建好
SHOW INDEX FROM article;

-- 4️⃣ 检查索引是否包含对应列
-- MATCH 列必须与 FULLTEXT 索引定义完全一致
```

### Q2：FULLTEXT 与 `LIKE '%keyword%'` 能否互相替代？

> 不能完全替代。`LIKE '%keyword%'` 的子串匹配能力更强（可以匹配词中的任意部分），但性能极差。FULLTEXT 基于分词匹配，性能好且支持排序，但无法做子串匹配。各有适用场景。

### Q3：InnoDB vs MyISAM 哪个 FULLTEXT 更好？

> MySQL 5.6+ 优先选择 InnoDB，因为 InnoDB 支持事务、行级锁、外键，FULLTEXT 功能已与 MyISAM 持平（甚至更多）。MyISAM 的 FULLTEXT 只在 5.6 之前的系统中存在遗留使用。

### Q4：FULLTEXT 索引占多少空间？

> 倒排索引通常占文本数据的 **50%~100%**，比 B+Tree 索引更"重"。如果全文索引列是 `TEXT` 长字段，索引体积可能超过数据本身，需规划好磁盘空间。

---

## 相关笔记

- [[MySQL索引类型]] — 索引分类总览，FULLTEXT 在其中与其他索引类型的对比
- [[MySQL索引创建原则]] — FULLTEXT 的适用场景澄清，`LIKE '%'` 索引失效的替代方案
- [[MySQL联合索引]] — 对比 B+Tree 联合索引与 FULLTEXT 多列索引的区别
- [[MVCC-多版本并发控制]] — InnoDB 事务特性与索引的关联
