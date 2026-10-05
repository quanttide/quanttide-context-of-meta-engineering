# pr4xis 引入评估

决策问题：pr4xis 是否进入标准库、进入哪一层。评估依据 2026-10-05 在 `examples/quanttide-meta-lab/src/cli` 的一次实测（pr4xis 0.29.1），实测代码与运行输出均可复跑。

## 结论与建议

- 建议引入，范围限定在有界状态机的前置条件检查：把 gallery 文档里的规则写成 `Precondition`，引擎拦下不合规的状态转换并指出具体是哪条公理失败。这一层已验证可用。
- 暂不引入的部分是推理与收敛：余极限计算、余锥选择、演绎归纳溯因都不在本次验证范围，简介提到的这些能力没有一条跑通过。
- 引入的两个前置条件：确认 CC-BY-NC-SA-4.0 非商业条款与实验室代码的分发方式不冲突；版本锁 0.29.1，该库仍处 0.x 迭代段。

## 支撑证据

API 以下载的 0.29.1 源码逐个核对，简介 `data/library/pr4xis.md` 只作线索。核对结果：`Engine::new(初始情境, 前置条件, apply)` 与 `.next()/.back()/.forward()` 属实；前置条件返回 `Verdict`（`Result<Box<dyn Proof>, Box<dyn Counterexample>>`），每个判定必须附带 `Provenance` 四字段。

拦截能力的实际输出，对应 `docs/gallery/ontology/entity/master-data.md` 的三层归属决议：

```text
2) 本体归数据工程 → 被拦下
   ✗ DeclaredInOntology: 本体未声明「本体」归 数据工程 —— 已声明的归属见 OwnedBy 边
```

收敛题的实际输出，对应 `docs/gallery/category/journal-vs-profile.md` 第七节：

```text
1) 直接 next(Converge) —— 被公理拦下：
   ✗ AlignmentComplete: 未对齐的日志事件：["10月3日调整预算", "10月5日暂停B渠道"]
   ✗ HumanConfirmed: 候选视图尚无人类确认，停在 pending
```

规则改动的成本在实测中为零：三层归属写在 `ontology!` 的三条 `OwnedBy` 边里，换归属改声明即可，引擎代码不动。两个示例 `cargo fmt --check`、`cargo clippy --all-targets -- -D warnings` 双绿，`cargo run --example` 可复跑。

## 证据的可信度边界

单次实测、两个示例，只覆盖状态机前置条件检查一条路径。`pr4xis-domains`、`.prx` 归档、跨领域函子、三种推理模式均未验证；对这些能力的判断目前仅来自简介，而简介已知有与源码不符之处，后续核对同样以源码为准。

## 待决事项

1. 是否引入及落点层级——标准库中承担规则检查，还是仅留在实验室试用。
2. 许可证合规确认——非商业条款与代码分发方式的冲突判断。
3. 是否投入验证推理与归档能力——这决定 pr4xis 能否承担标准库中的推理角色。
