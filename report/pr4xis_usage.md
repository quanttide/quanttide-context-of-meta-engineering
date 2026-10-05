# pr4xis 使用总结

2026-10-05，用 pr4xis 0.29.1 在 `examples/quanttide-meta-lab/src/cli` 写成两个示例，把 `docs/gallery` 的两篇文档变成机器可执行的检查：`journal_profile_convergence.rs` 对应日志与档案的收敛，`master_data_ownership.rs` 对应主数据三层归属。记录这次用到的东西和学到的教训。

## 简介可信但要下载源码核对

`data/library/pr4xis.md` 的简介方向正确，落到类型就有几处出入，照简介写编不过。

- `Engine::new(初始情境, 前置条件, apply)` 与 `.next()`、`.back()`、`.forward()` 属实；`EngineError::Violated` 把引擎和违反它的公理一并交还，所以被拦下之后能继续操作，也能打印出是哪条公理拦的。
- 前置条件 `Precondition::check` 返回 `Verdict`，类型是 `Result<Box<dyn Proof>, Box<dyn Counterexample>>`。库里没有从 bool 构造 Verdict 的助手，每次检查都要显式给出证明或反例。
- 证明和反例都带 `Provenance`（`name`、`description`、`citation`、`module_path` 四个字段）。示例先写 `axiom_meta()` 再写规则，公理出处填 gallery 文档路径。
- `ontology!` 宏可用的子句是 `name`、`source`、`concepts`、`labels`、`edges`，`labels` 接受中文，例如 `("zh", "本体", "实例数据的正本")`。宏生成 `{Name}Concept`、`{Name}Category`、`{Name}Ontology` 和关系种类枚举，引擎侧直接查 Category 的边。

## 归属规则写在本体里

`master_data_ownership.rs` 用三条 `OwnedBy` 边声明三层归属，前置条件只查 Category 里有没有对应边。本体声明的是归业务系统，所以「本体归数据工程」被拦下，实际输出：

```text
✗ DeclaredInOntology: 本体未声明「本体」归 数据工程 —— 已声明的归属见 OwnedBy 边
```

换归属就改那行边，引擎代码不动。规则与代码分离这一点在示例里成立。

## 收敛题的实际结果

`journal_profile_convergence.rs` 初始只有第一条事件对齐，直接 `next(Converge)` 被两条公理拦下：

```text
✗ AlignmentComplete: 未对齐的日志事件：["10月3日调整预算", "10月5日暂停B渠道"]
✗ HumanConfirmed: 候选视图尚无人类确认，停在 pending
```

补齐对齐、把档案侧无对应的事件显式标记、填 `approved_by` 之后收敛通过；`.back()` 回到 Pending，`.forward()` 回到 Converged，轨迹 5 条含被拦的那次。余极限怎么算、三个余锥选哪个仍在引擎之外，示例里对应 `Approve` 这个人类动作。

## 工程流程

- crates.io 的 API 对脚本返回 403，取源码改走 `index.crates.io` 查版本、`static.crates.io` 下载 `.crate`，解包读 `src/engine/` 和 `ontology/tests.rs` 才拿到真实语法。
- 包里只有 `examples/` 也能构建，不需要 `src/lib.rs`，`cargo check --example <name>` 可用。
- `cargo clippy --all-targets -- -D warnings` 拦下三类问题：变体名以 enum 名结尾（`Layer::OntologyLayer` 改成 `Layer::Ontology`）、Copy 类型上调 `clone()`、`Result` 上 `.ok().expect()`。fmt 与 clippy 双绿后再提交。
- 实验室子模块处于分离头指针，推送用 `git push origin HEAD:main`；这次远端多出一条改名提交，先 `git fetch` 加 `git rebase origin/main` 才推得上去。

## 未验证范围与许可证

`pr4xis-domains`、`.prx` 归档、跨领域函子、演绎归纳溯因三种推理模式这次都没碰，示例只覆盖状态机前置条件检查这一条路径。pr4xis 许可证为 CC-BY-NC-SA-4.0 含非商业条款，实验室代码若要对外分发需先确认。
