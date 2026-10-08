代码模块 & 测试模块的规格表示

先说结论，再说为什么，最后给例子。

一、结论：用什么格式

模块 推荐格式 理由
代码模块 契约式（Contract）：意图 + 边界 + 不变量 能表达”我要什么、不要什么、什么永远成立“
测试模块 断言式（Assertion）：场景 → 期望 能表达”什么情况下必须是什么结果“

两者共用一个核心结构：Given（前提）→ 约束（边界）→ 期望（结果）

不要用伪代码，不要用自然语言散文，不要用 UML。用结构化清单。

二、为什么是这个格式

你的问题是 AI 建模偏离意图。偏离的本质是：

· 你说的词（”提醒“）和 AI 理解的词（”通知系统“）不同
· 你没说边界（”最多一个文件“），AI 就无限扩张
· 你没说”不要什么“，AI 就全都要

所以规格必须包含三样东西，缺一不可：

1. 意图：我到底要解决什么问题（钉死抽象词）
2. 边界：规模上限 + 明确不要什么（防膨胀）
3. 不变量：无论怎么实现，哪些事永远成立（防跑偏）

三、代码模块规格

```yaml
module: task-reminder
intent: |
  用户给任务设一个时间点，到点弹一次提醒。就这一件事。

scope:
  files_max: 1
  concepts_max: 2        # Task, Reminder
  lines_estimate: 50

vocabulary:
  reminder: ”到点弹一次窗，仅此一次“
  NOT:
    - 通知系统
    - 多渠道（邮件/短信/推送）
    - 重试机制
    - 模板系统
    - 用户偏好

invariants:
  - 一个任务最多一个提醒
  - 提醒只弹一次，弹完即失效
  - 不依赖任何外部服务

interface:
  - set_reminder(task_id, time) -> reminder_id
  - cancel_reminder(reminder_id) -> bool
```

四、测试模块规格

```yaml
module: task-reminder-test
covers: task-reminder

scenarios:
  - given: 任务存在，时间在未来
    when: set_reminder(task_id, future_time)
    then: 返回 reminder_id，提醒被记录

  - given: 提醒已设，时间到了
    when: 时间触发
    then: 弹一次窗，且仅一次

  - given: 提醒已触发过
    when: 时间再次到达
    then: 不再弹窗

  - given: 提醒已设
    when: cancel_reminder(reminder_id)
    then: 返回 true，提醒被删除

must_not:
  - 不测试邮件/短信/推送
  - 不测试重试
  - 不测试模板渲染
```

五、两者合起来看

```
┌─────────────────────┐         ┌─────────────────────┐
│  代码模块规格         │         │  测试模块规格         │
│                     │         │                     │
│  intent  我要什么    │────────►│  covers  覆盖谁      │
│  scope   多大        │         │  scenarios 场景期望   │
│  NOT     不要什么    │         │  must_not 不测什么   │
│  invariants 永远成立  │         │                     │
└─────────────────────┘         └─────────────────────┘
        │                                  │
        └──────────────┬───────────────────┘
                       ▼
              给 AI 的完整规格
        （它照着建，就不可能跑偏太远）
```

六、为什么这样能治”建模偏离“

偏离原因 规格里的对策
抽象词理解不同 vocabulary 里钉死 + NOT 列表
规模无限膨胀 scope.files_max / concepts_max
加了没要的东西 NOT 列表显式禁止
实现跑偏 invariants 永远成立
测了不该测的 must_not 显式禁止

七、和你反思程序的关系

规格是反思的产物，也是反思的输入。

```
反思 → 发现”上次AI把提醒做成了通知系统“
     → 固化成一条规则：”抽象词必须先钉NOT列表“
     → 下次写规格时，自动带上 vocabulary + NOT
     → AI 拿到带 NOT 的规格，就不会跑偏
```

也就是说：

· reflect commit 存的规则，最终会变成规格模板的一部分
· reflect check 在写规格前提醒你：”上次这里跑偏了，记得钉边界“

规格是”这次怎么跟AI说“，规则是”以后每次都怎么说“。

八、一句话

代码规格 = 意图 + 边界 + 不变量 + 不要什么；测试规格 = 场景 + 期望 + 不要测什么。两者共用”钉死词汇、划清边界“的结构，专门治 AI 建模偏离。

要不要我把这两个规格套进你前面的 reflect 命令流程，走一个”从偏离到规格“的完整例子？
