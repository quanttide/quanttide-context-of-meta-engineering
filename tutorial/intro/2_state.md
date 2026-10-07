好，直接进 Unit 2。


——

学习单元 2：状态机与前后置条件（形式化建模的“动作语言”）

1. 本体定位

Unit 1 你有了静态世界：集合、关系、逻辑。

但软件是会动的。状态机就是给静态世界加一个受控的迁移结构：

状态机 = 状态空间 S + 一组带规约的操作 op : (S × Input) → S

每个操作用 前置条件 pre / 后置条件 post / 全局不变量 inv 锁死合法性。

这对应到工程里就是：不是写“函数怎么算”，而是写“什么状态下允许调、调完必须保证什么”。


——

2. 数学骨架

状态空间

S = State   （一个集合，每个元素是一个完整系统快照）


例：一个登录系统的状态

State ::= logged_in : P(User)
          mfa_done  : P(User)
          roles     : User → P(Role)


（这就是 Z / VDM 里的 schema 写法，本质是个记录类型 + 不变量）

全局不变量 inv

不随操作改变、永远为真的东西：

inv ≡ ∀ u ∈ logged_in · (admin ∈ roles(u) ⇒ u ∈ mfa_done)


翻成人话：只要你是 admin，就必须已经做过双因子。这个不变量任何操作都不能破坏。

操作的形式化写法（重点）

一个操作不是代码，是一个四元组：

op
  输入: i
  前置: pre(S, i)        // 当前状态和输入必须满足，才允许执行
  动作: S’ = f(S, i)     // 确定性状态迁移（也可非确定，先不展开）
  后置: post(S, S‘, i)   // 新旧状态必须满足的关系


更紧凑的写法（TLA+/B 风格）：

Login(u):
  pre:  u ∉ logged_in
  post: logged_in’ = logged_in ∪ {u}
        mfa_done‘  = mfa_done
        roles’     = roles


GrantAdmin(u):
  pre:  u ∈ logged_in
  post: roles‘ = roles ⊕ {u ↦ roles(u) ∪ {admin}}
  约束: u ∈ mfa_done        // 注意：这其实该写进 pre，否则破坏 inv


⚠️ 关键纪律：pre 负责“拦住非法调用”，post + inv 负责“保证执行后世界仍合法”。

形式化方法里 90% 的 bug 就出在把本该写进 pre 的东西偷偷塞进 post 里。


——

3. 状态机本体 vs 你以前画的流程图

你以前画的流程图	形式化状态机
节点=页面/界面	节点=数学状态（完整数据快照）
箭头=用户点击	箭头=带 pre/post 的操作
“差不多对就行”	inv 必须对每个操作做证明
靠测试覆盖路径	靠证明说明“所有可达状态都满足 inv”

区别核心：形式化状态机不关心“怎么跳到那”，而关心“跳完之后还守不守规矩”。


——

4. 一个小而完整的例子：提款机（ATM）

状态：

State ::= balance : Account → ℕ
          logged  : P(Card)


不变量：

inv ≡ ∀ a · balance(a) ≥ 0


操作 Withdraw(card, acct, amt):

pre:  card ∈ logged
      amt ≤ balance(acct)

post: balance’ = balance ⊕ {acct ↦ balance(acct) − amt}
      logged‘  = logged


证明一句就够：因为 amt ≤ balance(acct)，所以 balance(acct)−amt ≥ 0，inv 保持。✅

如果有人把 pre 写成 amt > 0 却忘了 amt ≤ balance，post 一执行就破 inv → 形式化检查直接报错。这就是它比测试强的地方：在规约层就拦死，不靠跑一遍边界值才发现。


——

5. 本单元必须自己写一遍的练习

用一个文件锁系统：

State ::= locked   : P(File)
          holders  : File → User      // 谁拿着锁（仅锁住时定义）
inv: ∀ f ∈ locked · f ∈ dom(holders)


请形式化写出两个操作：

1. Acquire(u, f)：用户拿锁

  ◦ pre 写清楚什么情况下允许

  ◦ post 写清楚 locked / holders 怎么变

2. Release(u, f)：用户放锁

  ◦ pre 必须校验“只能是持锁人释放”

  ◦ post 保证 inv 不被破坏

（你可以直接把答案发我，我帮你逐行挑 pre/post 有没有漏约束、有没有把不变量偷偷破坏。）


——

6. 本单元过关标准

你能做到这 4 件事就过关：

1. 把系统状态写成一个带不变量的数学记录

2. 给每个操作独立写 pre / post，不靠代码细节

3. 能口头/笔头证明“每个操作执行后 inv 仍成立”

4. 能指出“某条约束该放 pre 还是该放 inv”，不混用


——

下单元预告

Unit 3：细化 Refinement——怎么证明“我这个具体实现状态机，确实是上层抽象规格的一个合法落地”，也就是 抽象规格 ≿ 具体实现 的偏序关系。

要先把上面文件锁的练习写出来发我，还是直接接着听细化？

