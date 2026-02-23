# Test Scenarios & Acceptance Criteria

> **QA Owner Deliverable** for AAVE Skill Team
>
> This document defines the complete test scenarios and acceptance criteria for the AAVE V3 Skill implementation.

---

## 标准触发词测试集（20条，需命中>=18条）

| # | 输入语句 | 期望Skill | 期望参数 |
|---|---------|----------|---------|
| 1 | "supply to aave" | aave-planner | action=supply |
| 2 | "deposit to aave" | aave-planner | action=supply |
| 3 | "lend on aave" | aave-planner | action=supply |
| 4 | "borrow from aave" | aave-planner | action=borrow |
| 5 | "take loan on aave" | aave-planner | action=borrow |
| 6 | "repay aave loan" | aave-planner | action=repay |
| 7 | "pay back aave" | aave-planner | action=repay |
| 8 | "withdraw from aave" | aave-planner | action=withdraw |
| 9 | "remove collateral" | aave-planner | action=withdraw |
| 10 | "aave lending" | aave-planner | (需补全action) |
| 11 | "earn yield on aave" | aave-planner | action=supply |
| 12 | "supply 100 USDC to aave" | aave-planner | action=supply, token=USDC, amount=100 |
| 13 | "borrow ETH from aave" | aave-planner | action=borrow, token=ETH |
| 14 | "repay my aave loan" | aave-planner | action=repay |
| 15 | "withdraw WETH from aave" | aave-planner | action=withdraw, token=WETH |
| 16 | "aave supply" | aave-planner | action=supply |
| 17 | "aave borrow" | aave-planner | action=borrow |
| 18 | "aave repay" | aave-planner | action=repay |
| 19 | "aave withdraw" | aave-planner | action=withdraw |
| 20 | "use aave to earn interest" | aave-planner | action=supply |

**验收标准**: 20条测试中至少命中18条（命中率>=90%）

---

## 对话式验收样例（15条）

### TS01: 简单Supply
**对话**: "我想在AAVE存100 USDC"

**期望**:
- 命中Skill: aave-planner
- 识别参数:
  ```json
  {
    "action": "supply",
    "token": "USDC",
    "amount": "100"
  }
  ```
- 缺失参数: chainId
- 补全行为: 使用AskUserQuestion询问"您想在哪个网络上进行操作？首批支持Ethereum和Arbitrum。"
- 最终确认: 参数齐全后生成deep link或手动路径

---

### TS02: 缺少Amount
**对话**: "我要借ETH"

**期望**:
- 命中Skill: aave-planner
- 识别参数:
  ```json
  {
    "action": "borrow",
    "token": "ETH"
  }
  ```
- 缺失参数: amount, chainId
- 补全行为:
  1. 询问chainId: "您想在哪个网络上借ETH？"
  2. 询问amount: "您想借多少ETH？"
- 校验: amount必须为正数，chainId必须是1或42161

---

### TS03: 无效Token
**对话**: "我想存XYZ"

**期望**:
- 命中Skill: aave-planner
- 识别参数:
  ```json
  {
    "action": "supply",
    "token": "XYZ"
  }
  ```
- 校验失败: XYZ不在白名单资产列表中
- 错误提示:
  ```
  ⚠️ XYZ不在首批支持资产列表中。

  首批支持资产：
  - Ethereum: USDC, USDT, WETH, WBTC, DAI
  - Arbitrum: USDC, USDT, WETH, WBTC, DAI

  您是指WETH还是其他资产？
  ```
- 回退: 提供相似选项供用户选择

---

### TS04: Stable Borrow不支持时降级
**对话**: "我要用固定利率借USDC"

**期望**:
- 命中Skill: aave-planner
- 识别参数:
  ```json
  {
    "action": "borrow",
    "token": "USDC",
    "interestRateMode": 1
  }
  ```
- 链上校验: 查询USDC的stableBorrowRateEnabled
- 校验结果: USDC.stableBorrowRateEnabled = false
- 降级处理:
  ```json
  {
    "finalInterestRateMode": 2,
    "modeDowngraded": true,
    "modeDowngradeReason": "USDC不支持固定利率借款，已自动切换为浮动利率(variable)"
  }
  ```
- 用户提示: "USDC不支持固定利率，已为您切换为浮动利率(variable)。当前浮动利率为X.XX%。"
- 继续流程: 使用variable rate完成后续处理

---

### TS05: 查询Health Factor
**对话**: "我的账户健康度怎么样"

**期望**:
- 命中Skill: aave-risk-assessor
- 触发词匹配: "health factor", "健康度", "账户健康"
- 所需参数: wallet地址
- 补全行为:
  - 若当前会话无wallet: 询问"请提供您的钱包地址"
  - 若已有wallet: 直接使用
- 查询行为:
  1. 调用Pool.getUserAccountData(wallet)
  2. 获取healthFactor、totalCollateralBase、totalDebtBase等
- 输出格式:
  ```
  您的账户健康度: 1.85
  风险等级: Moderate (建议关注)

  详细信息:
  - 总抵押品价值: $10,500
  - 总债务: $5,675
  - 当前LTV: 54%
  - 最大LTV: 75%
  - 清算阈值: 80%
  ```

---

### TS06: 跨链提示
**对话**: "我要在Polygon上borrow"

**期望**:
- 命中Skill: aave-planner
- 识别参数:
  ```json
  {
    "action": "borrow",
    "chainId": 137
  }
  ```
- 校验失败: chainId 137 (Polygon) 不在首批支持列表
- 错误提示:
  ```
  ⚠️ 首批仅支持Ethereum和Arbitrum网络。

  Polygon网络支持计划在二期迭代中推出。

  您是否想切换到：
  1. Ethereum (主网)
  2. Arbitrum (L2)
  ```
- 回退: 提供支持的链选项供用户选择

---

### TS07: 缺少Chain
**对话**: "supply 100 USDC"

**期望**:
- 命中Skill: aave-planner
- 识别参数:
  ```json
  {
    "action": "supply",
    "token": "USDC",
    "amount": "100"
  }
  ```
- 缺失参数: chainId
- 补全行为: 使用AskUserQuestion询问
  ```
  您想在哪个网络上进行supply操作？

  [Ethereum] [Arbitrum]
  ```
- 校验: 用户选择后验证chainId为1或42161

---

### TS08: 非白名单资产警告
**对话**: "我想存RANDOM_TOKEN"

**期望**:
- 命中Skill: aave-planner
- 识别参数:
  ```json
  {
    "action": "supply",
    "token": "RANDOM_TOKEN"
  }
  ```
- 校验: RANDOM_TOKEN不在白名单
- 告警文案:
  ```
  ⚠️ **安全警告**

  RANDOM_TOKEN不在首批支持资产列表中。

  首版仅支持以下白名单资产：
  - USDC, USDT, WETH, WBTC, DAI

  注意：非白名单资产可能存在诈骗风险。首版不支持非白名单资产的交易操作。

  如需支持该资产，请联系团队扩展白名单。
  ```
- 边界处理: 不生成交易参数，流程终止

---

### TS09: 网络错误回退
**对话**: "supply ETH"

**场景**: 链上RPC调用失败

**期望**:
- 命中Skill: aave-planner
- 识别参数:
  ```json
  {
    "action": "supply",
    "token": "ETH"
  }
  ```
- 失败点: 调用getReserveConfigurationData时网络错误
- 错误提示:
  ```
  ⚠️ 无法获取链上数据

  可能原因：
  - 网络连接不稳定
  - RPC节点暂时不可用

  请稍后重试，或手动访问: https://app.aave.com
  ```
- 回退方案（手动路径）:
  ```
  手动操作步骤:
  1. 访问: https://app.aave.com
  2. 选择市场: Aave v3 Ethereum
  3. 选择资产: ETH
  4. 点击: Supply 按钮
  5. 输入金额并确认
  ```

---

### TS10: AskUserQuestion不可用时
**环境**: 运行时无AskUserQuestion工具
**对话**: "我要borrow"

**期望**:
- 命中Skill: aave-planner
- 识别参数:
  ```json
  {
    "action": "borrow"
  }
  ```
- 缺失参数: token, amount, chainId
- 回退行为（自然语言对话）:
  ```
  我来帮您完成borrow操作。请提供以下信息：

  1. 您想借什么资产？(如: USDC, ETH, WBTC)
  2. 您想借多少数量？
  3. 您想在哪个网络上操作？(Ethereum/Arbitrum)
  ```
- 校验保持: 同样的输入校验逻辑（白名单检查、数值校验等）
- 参数收集: 通过多轮对话逐步收集

---

### TS11: Deep link失效回退
**场景**: deeplink预填成功率<80%

**期望**:
- 前置条件: 在Ethereum和Arbitrum各测试5个案例，成功率<80%
- 判定结果: 不使用deep link作为主方案
- 默认行为:
  ```
  请按以下步骤手动操作:

  1. 访问: https://app.aave.com
  2. 选择市场: Aave v3 {Network}
  3. 连接您的钱包
  4. 在资产列表中找到 {Token}
  5. 点击 "{Action}" 按钮
  6. 输入金额: {Amount}
  7. 确认交易
  ```
- 参数对照表:
  ```
  操作参数:
  - 网络: {chainName}
  - 资产: {token}
  - 动作: {action}
  - 金额: {amount}
  - 利率模式: {interestRateMode}
  ```

---

### TS12: interestRateMode默认
**对话**: "borrow 100 USDC"

**期望**:
- 命中Skill: aave-planner
- 识别参数:
  ```json
  {
    "action": "borrow",
    "token": "USDC",
    "amount": "100"
  }
  ```
- 默认值处理:
  ```json
  {
    "interestRateMode": 2,
    "finalInterestRateMode": 2
  }
  ```
- 行为: 用户未指定时，默认使用variable rate (2)
- 无提示: 不显示stable borrow相关提示（用户未询问）
- 输出: 显示当前variable borrow利率

---

### TS13: eMode相关（超范围动作）
**对话**: "我想开启eMode"

**期望**:
- 命中Skill: aave-planner
- 意图识别: 用户想操作eMode
- 范围检查: eMode不在首批功能范围(supply/borrow/repay/withdraw)
- 提示文案:
  ```
  ⚠️ eMode管理不在首批功能范围

  首批仅支持以下操作：
  - Supply (存款)
  - Borrow (借款)
  - Repay (还款)
  - Withdraw (取款)

  eMode支持预计在二期迭代中推出，敬请期待。
  ```
- 参数处理: **不生成** {action: "eMode"} 参数
- 动作定义: 保持与SKILL.md中定义的action枚举一致，避免漂移

---

### TS14: Repay全部
**对话**: "还清我的贷款"

**期望**:
- 命中Skill: aave-planner
- 识别参数:
  ```json
  {
    "action": "repay",
    "amount": "max"
  }
  ```
- 缺失参数: token, chainId
- 补全行为:
  1. 询问chainId
  2. 查询用户当前债务代币列表
  3. 若多种债务: "您想还清哪种资产的贷款？"
  4. 若单一债务: 直接使用该token
- 特殊值: amount="max" 表示还清全部（使用type(uint256).max）
- 确认提示: "确认还清全部 {token} 贷款？当前债务为 {currentDebt}"

---

### TS15: Withdraw超过供给
**对话**: "withdraw 1000 ETH"

**前置条件**: 用户当前仅有10 ETH供给

**期望**:
- 命中Skill: aave-planner
- 识别参数:
  ```json
  {
    "action": "withdraw",
    "token": "ETH",
    "amount": "1000"
  }
  ```
- 链上校验: 查询用户当前ETH供给余额
- 校验结果: currentSupplied = 10 ETH
- 错误提示:
  ```
  ⚠️ 取款金额超过供给余额

  您当前仅有: 10 ETH 可供提取
  请求取款: 1000 ETH

  您是否想：
  1. 提取全部 (10 ETH)
  2. 修改取款金额
  3. 取消操作
  ```
- 回退: 提供选项让用户修改或取消

---

## 失败处理路径验证（至少3类）

### 类别1: 参数缺失/无效
| 场景 | 输入 | 期望行为 |
|-----|------|---------|
| 完全缺参 | "use aave" | 询问具体action |
| 无效amount | "supply -100 USDC" | 提示金额必须为正数 |
| 无效token | "supply ABC" | 提示不在白名单，给出选项 |
| 无效chain | "supply on Bitcoin" | 提示仅支持Ethereum/Arbitrum |

### 类别2: 链上状态限制
| 场景 | 输入 | 期望行为 |
|-----|------|---------|
| 资产冻结 | "supply USDC" (USDC frozen) | 提示资产当前不可supply |
| borrow cap已满 | "borrow WBTC" | 提示已达borrow上限 |
| stable不可借 | "stable borrow USDC" | 降级为variable并提示 |
| HF<1.0 | "borrow more" | 警告已被清算或即将清算 |

### 类别3: 运行时/网络错误
| 场景 | 输入 | 期望行为 |
|-----|------|---------|
| RPC超时 | "supply ETH" | 提示网络错误，给出手动路径 |
| 合约调用失败 | 任何操作 | 提示具体错误，建议重试 |
| AskUserQuestion不可用 | 缺参对话 | 改用自然语言对话补参 |
| Deep link失效 | 生成链接时 | 回退到手动步骤指南 |

---

## 最小验收门槛（Go/No-Go Criteria）

### Go/No-Go Checklist

- [ ] **触发词命中率**: 20条标准触发语句命中>=18条（90%+）
- [ ] **参数补全稳定性**: 缺参时能稳定补全（AskUserQuestion或自然语言），不直接失败
- [ ] **失败路径覆盖**: 至少3类失败路径有明确错误提示和下一步动作
- [ ] **链覆盖**: 首批2条链（Ethereum/Arbitrum）全覆盖
- [ ] **动作覆盖**: 首批4个动作（supply/borrow/repay/withdraw）全覆盖
- [ ] **利率模式降级**: stable borrow不支持时自动降级并提示
- [ ] **Deep link回退**: deep link失效时有回退方案
- [ ] **非白名单告警**: 非白名单资产有明确告警和边界定义
- [ ] **Health Factor查询**: aave-risk-assessor能正确查询和解释健康度
- [ ] **跨链处理**: 不支持的网络有明确提示和切换选项

### 验收通过标准

**Go**: 所有10项检查通过
**Conditional Go**: 8-9项通过，剩余项有明确修复计划
**No-Go**: 少于8项通过

---

## 测试执行记录模板

### 触发词测试记录

| # | 输入语句 | 命中Skill | 命中参数 | 结果 | 备注 |
|---|---------|----------|---------|------|------|
| 1 | "supply to aave" | | | | |
| ... | ... | | | | |

**统计**: __/20 通过 (____%)

### Deep Link验证记录

| 网络 | 资产 | 动作 | 预填成功率 | 结果 |
|-----|------|------|-----------|------|
| Ethereum | USDC | supply | | |
| Ethereum | WETH | borrow | | |
| ... | ... | ... | ... | ... |

**判定**: 主方案 / 回退方案

### 对话式测试记录

| 场景ID | 测试日期 | 测试员 | 结果 | 问题记录 |
|-------|---------|-------|------|---------|
| TS01 | | | | |
| ... | | | | |

---

## 附录: 白名单资产参考

### Ethereum (chainId: 1)
| Symbol | Address | Decimals | stableBorrowEnabled |
|--------|---------|----------|---------------------|
| USDC | 0xA0b86a33E6441E6C7D3D4B4f6c7B8A9c2D3E4F5 | 6 | false |
| USDT | 0xdAC17F958D2ee523a2206206994597C13D831ec7 | 6 | false |
| WETH | 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 | 18 | false |
| WBTC | 0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599 | 8 | false |
| DAI | 0x6B175474E89094C44Da98b954EedeAC495271d0F | 18 | true |

### Arbitrum (chainId: 42161)
| Symbol | Address | Decimals | stableBorrowEnabled |
|--------|---------|----------|---------------------|
| USDC | 0xaf88d065e77c8cC2239327C5EDb3A432268e5831 | 6 | false |
| USDT | 0xFd086bC7CD5C481DCC9C85ebE478A1C0b69FCbb9 | 6 | false |
| WETH | 0x82aF49447D8a07e3bd95BD0d56f35241523fBab1 | 18 | false |
| WBTC | 0x2f2a2543B76A4166549F7aaB2e75Bef0aefC5B0f | 8 | false |
| DAI | 0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1 | 18 | true |

---

*Document Version: 1.0*
*Last Updated: 2026-02-22*
*Owner: QA Owner*
