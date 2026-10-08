# 量潮认知工程

1. 业务需求

用结构化外化 + 计算审计，把零散经验提炼为可推理、可修改的自我认知模型。

流程五阶段：

阶段 动作 真实需求
① 沉淀 从 Markdown 提取 TTL 原始经验 → 可计算实例
② 聚类 按主题归拢记忆 散落实例 → 有边界的集合（情境）
③ 抽象 从情境提取共同模式 集合 → 可复用类型（图式）
④ 审计 算结构缺陷 可计算证据对抗通读印象
⑤ 修订 改 TTL 而非改镜子 分清数据缺口（真）vs 工具缺陷（伪）

一句话：

记忆 → 情境 → 图式，每步可计算、可审计、可修订。

—

2. 哲学意图

双重本体：

· 工程本体（RDFS）：类型系统
· 哲学本体（范畴）：自我 = 对象 + 态射

三层对应三种数学对象：

层 哲学身份 数学身份
记忆 现象界碎片 元素
情境 现象的分类 集合
图式 本体界模式 类型

两次跨层操作，性质不同：

\text{记忆} \xrightarrow{\ \text{聚类}\ } \text{情境} \xrightarrow{\ \text{抽象}\ } \text{图式}

图式的双重运动（皮亚杰式）：

```turtle
:schemaX     :derivedFrom  :situation1 .   # 顺应：情境修改图式
:situation1  :framedBy     :schemaX .       # 同化：图式套住情境
```

肯定 / 否定两条认识路径：

· 肯定（cataphatic）：serves 链
· 否定（apophatic）：excludes 链

核心洞察：

肯定句给你一个词，否定句给你一条边。
词是高自由度的，边是受约束的。
所以否定天然携带结构，结构可计算。

—

3. 数学形式化定义与建模

图模型：

G = (V, E), \quad E \subseteq V \times P \times V

P = \{\text{contains, derivedFrom, framedBy, evokes, responds\_to, serves, excludes, seeAlso}\}

三层对象：

· 记忆层 M \subseteq V
· 情境层 \mathcal{S} \subseteq 2^M
· 图式层 \mathcal{K}

两次操作：

\text{cluster}: M \to \mathcal{S}, \quad \text{abstract}: \mathcal{S} \to \mathcal{K}

核心度量：

\text{cohesion}(S) = \frac{\text{情境内相似记忆对数}}{\text{情境内记忆总对数}}

\text{overlap}(S_1, S_2) = \frac{|S_1 \cap S_2|}{|S_1 \cup S_2|}

\text{coverage}(K) = |\{S : K \xrightarrow{\text{derivedFrom}} S\}|

\text{orphan} = \{m \in S : S \text{ 无 derivedFrom 出边}\}

\rho(S) = \frac{|\text{excludes}|}{|\text{serves}|}

H = -\sum_i \frac{f_i}{F} \log_2 \frac{f_i}{F}

同源等价划分：

R = \text{seeAlso 在 Dilemma 上的传递闭包}, \quad \mathcal{P} = V_{\text{Dilemma}} / R

度量有效性定理：

度量必须在其真实定义域上计算：实例集 ∖ 模式集，边集 ∖ 对象属性边。定义域错了，结果是伪值。

—

4. 代码实现

核心结构：

```python
def localname(uri):
    return str(uri).split(’#‘)[-1].split(’:‘)[-1]

def analyze(g):
    instances = {s: localname(o) for (s, type, o)
                 if localname(o) not in SCHEMA}
    edges = [(s, localname(p), o) for (s, p, o)
             if localname(p) in MORPH]
    connected = {端点 for 边 in edges for 端点 in 边两端}
    dist = len(instances & connected) / len(instances)
    orphans = [n for n in instances if n not in connected]
    ...
    return ...

def entropy(counter):
    F = sum(counter.values())
    return -sum((v/F) * log2(v/F) for v in counter.values())
```

三层对应实现操作：

层 操作 代码职责
记忆 提取 Markdown → TTL 实例
情境 聚类 contains 构建集合
图式 抽象 derivedFrom 从集合提升类型
审计 度量 cohesion / overlap / coverage / orphan
修订 propose 对孤节点反向推荐候选连接

四条设计原则：

1. 镜子不编辑：audit.py 只度量，TTL 修订在人工标注会话完成。
2. 定义域优先：所有度量先确定实例集 ∖ 模式集。
3. 两次操作分离：聚类与抽象必须是两个函数。
4. 双向边并存：derivedFrom 与 framedBy 同时存在，才能表达软性图式。

下一步扩展：

· propose 模式：孤节点反向推荐候选连接
· cohesion / overlap：暴露虚情境与重复主题
· coverage：暴露过拟合图式
· orphan：暴露未消化经验



