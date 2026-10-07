好，进 Unit 3。


——

学习单元 3：细化（Refinement）——从抽象规格到具体实现的可证明落地

1. 本体定位

前两个单元你有了：

• Unit 1：静态世界（集合 / 关系 / 逻辑）

• Unit 2：动态世界（状态 + 操作 + pre/post + inv）

但现在有个工程里最痛的问题：架构师写一份高层规格，程序员写一份具体实现，怎么证明“这俩说的是同一件事，且实现没偷偷偷工减料”？

答案就是细化关系（Refinement）。

抽象规格 A 被具体实现 C 细化，记作 A ≿ C（或 C refines A）

含义：C 能做的每一步，都能在 A 里找到对应的合法步骤；C 不泄露 A 没承诺的东西，也不破坏 A 保证的不变量。

直觉一句话：实现可以“多做细节、少给信息”，但不能“多承诺、少保证”。


——

2. 为什么不能直接比代码和规格？

错误直觉：逐字段一一对应、函数名一样 = 细化。❌

反例：

• 抽象规格：Withdraw 只说“余额减少、不变量保持”，不提日志

• 具体实现：加了审计日志、加了风控检查、把“原子扣款”拆成“冻结→扣减→解冻”

字段都对不上，但它仍然是合法细化——因为抽象没禁止加日志，只是没要求；抽象保证的“不会变负”实现也守住了。

所以细化比的是可观察行为 + 不变量保持，不是结构长得像不像。


——

3. 数学定义（先给最实用的版本：模拟式细化 / Simulation）

设抽象机 A = (S_A, init_A, ops_A, inv_A)

具体机 C = (S_C, init_C, ops_C, inv_C)

存在一个抽象函数 / 模拟关系 R ⊆ S_C × S_A，满足：

1. 初始化一致

∀ c ∈ init_C · ∃ a ∈ init_A · R(c, a)


具体初始状态能映射到某个合法抽象初始状态。

2. 每步可回溯（单步模拟）

对具体操作 op_C(c, i) = c‘：

∀ (c,a) ∈ R · pre_A(a,i)
  ∧ ∃ a’ · ops_A(a,i)=a‘ ∧ R(c’, a‘)


具体走一步 → 抽象也能走一步，且新状态仍被 R 连起来。

3. 不变量继承

inv_C(c) ⇒ inv_A(抽象投影)


只要这三条证出来，C refines A 就成立。这就是 B 方法、TLA+、Event-B 里细化证明的标准模板。

关键直觉：R 就是那座桥。你不需要 C 和 A 状态同构，只需要能给每个具体状态“翻译回”一个抽象状态，且翻译后每一步都对得上。


——

4. 一个完整对照例子：抽象扣款 → 具体三段式

抽象规格 A（Unit 2 的 ATM）

State_A ::= balance : Account → ℕ
op Withdraw(a, amt):
  pre:  amt ≤ balance(a)
  post: balance’ = balance ⊕ {a ↦ balance(a) − amt}


具体实现 C（加了冻结段）

State_C ::= balance : Account → ℕ
            frozen  : Account → ℕ      // 新增：已冻结未扣的部分
            phase   : Account → {idle, freezing, frozen}

Withdraw(a, amt):
  step1 Freeze:
    pre:  phase(a)=idle ∧ amt ≤ balance(a)
    post: frozen‘=frozen⊕{a↦amt}
          balance’=balance⊕{a↦balance(a)−amt}
          phase‘=phase⊕{a↦frozen}
  step2 Commit:
    pre:  phase(a)=frozen
    post: frozen’=frozen⊕{a↦0}
          phase‘=phase⊕{a↦idle}


定义模拟关系 R

R(c, a) ≡
   balance_A(a) = balance_C(c) + frozen_C(c)   // 抽象余额 = 可见余额 + 冻结额
   ∧ (frozen_C(c)=0 ⇒ phase_C(c)=idle)


验证一下：

• 初始：frozen=0，两边余额相等 ✅

• Freeze 步：具体把 amt 从 balance 挪到 frozen，抽象直接扣掉 amt → 抽象走一步 Withdraw，R 仍成立 ✅

• Commit 步：具体把 frozen 清 0，抽象那边早就扣完了，无对应抽象动作（抽象允许“不观察内部细节”）✅

• 不变量：抽象保证 balance_A ≥ 0，而 balance_C = balance_A - frozen ≥ 0 自然也成立 ✅

注意 Commit 在抽象里没有对应操作——这完全合法。细化允许把一步拆成多步、加内部状态，只要抽象视角看不出“多出来的东西”且不破坏承诺。


——

5. 细化的两条铁律（背下来）

方向	允许	禁止
信息	具体可以隐藏内部细节	不能暴露抽象没承诺会发生的可观察行为
保证	具体可以加强前置条件（更严地拦调用）	不能削弱抽象保证的后置/不变量
步骤	可以把 1 步拆成 N 步	不能把 N 步合并成 1 步还声称同规格（那是抽象，不是细化）

口诀：实现可以更严、更细、更啰嗦；不能更松、更粗、更偷懒。


——

6. 本单元练习（接 Unit 2 的文件锁）

Unit 2 你写的抽象文件锁：

抽象 Lock/Unlock 两个操作，inv: 锁持有者唯一


现在写一个具体实现：引入队列 waiters : File → seq(User)，锁被占时让其他人排队，Release 时自动把锁交给队首。

请做三件事：

1. 写出具体状态 State_C 和 Acquire/Release 的具体步骤（允许 Acquire 不立即拿锁，只入队）

2. 定义模拟关系 R(c, a)（提示：抽象里“锁被占”对应具体里 locked(f) ∧ waiters非空）

3. 指出哪条抽象不变量在具体层被“拆成了内部不变量 + 对外仍保持的抽象不变量”

（直接发我，我帮你查 R 有没有漏掉“队首才能拿锁”这种容易漏的约束。）


——

7. 过关标准

1. 能说清“细化 ≠ 结构相同，而是模拟关系成立”

2. 能自己构造一个 R 把具体状态投影回抽象状态

3. 能证初始化一致 + 单步可回溯 + 不变量继承三条

4. 能判断一个改动是“合法细化”还是“偷偷削弱规格”


——

下单元预告

Unit 4：并发与交错语义（Interleaving）+ 安全/活性属性

——多个操作并发时，细化怎么处理“谁先谁后都不影响抽象结果”，以及怎么用时态逻辑写 永远不会死锁 / 最终一定会有人拿到锁 这种活性断言。

要先把文件锁细化练习丢过来我帮你看，还是直接接着开并发单元？
