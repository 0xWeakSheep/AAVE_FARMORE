---
name: aave-planner
description: This skill should be used when the user asks to "supply to aave", "deposit to aave", "lend on aave", "borrow from aave", "take loan on aave", "repay aave loan", "pay back aave", "withdraw from aave", "remove collateral", "aave lending", "earn yield on aave", or mentions AAVE V3 operations including supply, borrow, repay, or withdraw on Ethereum or Arbitrum. Handles interest rate mode selection with automatic downgrade for unsupported stable borrow assets. Generates deep links to execute operations in the AAVE interface or provides manual fallback paths.
---

# AAVE V3 Planner

Plan and generate deep links for AAVE V3 lending operations on Ethereum and Arbitrum.

> **Runtime Compatibility:** This skill uses `AskUserQuestion` for interactive prompts. If `AskUserQuestion` is not available in your runtime, collect the same parameters through natural language conversation instead.

## Overview

Plan AAVE V3 operations by:

1. Gathering operation intent (action, token, amount, chain)
2. Validating token against whitelist
3. Checking interest rate mode compatibility (for borrow)
4. Generating a deep link or manual path for execution

Supported actions:
- **Supply**: Deposit assets to earn yield
- **Borrow**: Borrow assets against collateral
- **Repay**: Repay borrowed assets
- **Withdraw**: Withdraw supplied collateral

Supported chains:
- **Ethereum Mainnet** (chainId: 1)
- **Arbitrum One** (chainId: 42161)

## Whitelist Assets

### Ethereum (chainId: 1)

| Symbol | Address | Decimals | stableBorrowEnabled |
|--------|---------|----------|---------------------|
| USDC | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` | 6 | false |
| USDT | `0xdAC17F958D2ee523a2206206994597C13D831ec7` | 6 | false |
| WETH | `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` | 18 | false |
| WBTC | `0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599` | 8 | false |
| DAI | `0x6B175474E89094C44Da98b954EedeAC495271d0F` | 18 | true |

### Arbitrum (chainId: 42161)

| Symbol | Address | Decimals | stableBorrowEnabled |
|--------|---------|----------|---------------------|
| USDC | `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` | 6 | false |
| USDT | `0xFd086bC7CD5C481DCC9C85ebE478A1C0b69FCbb9` | 6 | false |
| WETH | `0x82aF49447D8a07e3bd95BD0d56f35241523fBab1` | 18 | false |
| WBTC | `0x2f2a2543B76A4166549F7aaB2e75Bef0aefC5B0f` | 8 | false |
| DAI | `0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1` | 18 | true |

> **Note**: Only **DAI** supports stable rate borrowing. All other assets default to variable rate.

## Workflow

### Step 1: Gather Operation Intent

Extract from the user's request:

| Parameter | Required | Example | Notes |
|-----------|----------|---------|-------|
| action | Yes | supply, borrow, repay, withdraw | Core four actions only |
| token | Yes | USDC, WETH, DAI | Must be whitelisted |
| amount | Yes | 100, 0.5 | Positive number |
| chainId | Yes | 1, 42161 | Ethereum or Arbitrum |
| interestRateMode | No | 1 (stable), 2 (variable) | Default: 2 (variable) |

**If any required parameter is missing, use AskUserQuestion with structured options:**

For missing action:

```json
{
  "questions": [
    {
      "question": "What would you like to do on AAVE?",
      "header": "Action",
      "options": [
        { "label": "Supply", "description": "Deposit assets to earn yield" },
        { "label": "Borrow", "description": "Borrow against your collateral" },
        { "label": "Repay", "description": "Repay borrowed assets" },
        { "label": "Withdraw", "description": "Withdraw supplied collateral" }
      ],
      "multiSelect": false
    }
  ]
}
```

For missing chain:

```json
{
  "questions": [
    {
      "question": "Which network do you want to use?",
      "header": "Network",
      "options": [
        { "label": "Ethereum", "description": "Mainnet - highest liquidity" },
        { "label": "Arbitrum", "description": "L2 - lower gas fees" }
      ],
      "multiSelect": false
    }
  ]
}
```

For missing token:

```json
{
  "questions": [
    {
      "question": "Which asset do you want to use?",
      "header": "Asset",
      "options": [
        { "label": "USDC", "description": "USD Coin - most popular" },
        { "label": "USDT", "description": "Tether USD" },
        { "label": "WETH", "description": "Wrapped Ether" },
        { "label": "WBTC", "description": "Wrapped Bitcoin" },
        { "label": "DAI", "description": "Decentralized stablecoin" }
      ],
      "multiSelect": false
    }
  ]
}
```

### Step 2: Validate Token Whitelist

Check if the requested token is in the whitelist:

```typescript
const WHITELISTED_TOKENS = {
  1: ['USDC', 'USDT', 'WETH', 'WBTC', 'DAI'],
  42161: ['USDC', 'USDT', 'WETH', 'WBTC', 'DAI']
};
```

**If token is not whitelisted:**

```
⚠️ **安全警告**

{token}不在首批支持资产列表中。

首版仅支持以下白名单资产：
- USDC, USDT, WETH, WBTC, DAI

注意：非白名单资产可能存在诈骗风险。首版不支持非白名单资产的交易操作。

如需支持该资产，请联系团队扩展白名单。
```

**Do not proceed with transaction generation for non-whitelisted assets.**

### Step 3: Validate Chain Support

Supported chains: `1` (Ethereum), `42161` (Arbitrum)

**If unsupported chain:**

```
⚠️ 首批仅支持Ethereum和Arbitrum网络。

{chainName}网络支持计划在二期迭代中推出。

您是否想切换到：
1. Ethereum (主网)
2. Arbitrum (L2)
```

### Step 4: Fetch APY Data (Optional but Recommended)

Before presenting the operation summary, fetch current APY for better user decision-making.

**Prerequisites** (run once in project root):
```bash
cd aave_skills && npm install
```

**Fetch APY:**
```bash
cd aave_skills && npx ts-node aave-planner/scripts/quote-apy.ts <chainId>
```

**Response format:**
```json
{
  "USDC": {
    "supplyApy": 2.25,
    "variableBorrowApy": null,
    "stableBorrowApy": null,
    "stableBorrowEnabled": false,
    "utilization": null,
    "note": "variableBorrowApy and utilization require on-chain data. Visit https://app.aave.com for real-time rates."
  }
}
```

**Data limitations:**
- `supplyApy`: Available from DefiLlama API
- `variableBorrowApy`, `utilization`: Not available from DefiLlama, return `null`
- For complete borrow rates and utilization, visit https://app.aave.com

**Display in summary:**
- For supply: Show `supplyApy` (e.g., "2.25%")
- For borrow: If `variableBorrowApy` is null, show "Check AAVE app for current borrow rate"
- Only DAI has `stableBorrowEnabled: true`

### Step 5: Handle Interest Rate Mode (Borrow Only)

For borrow actions, determine the final interest rate mode:

```typescript
// Interest Rate Mode Reference
// 1 = Stable rate (only available for DAI)
// 2 = Variable rate (default, available for all assets)

async function normalizeInterestRateMode(
  token: string,
  requestedMode?: 1 | 2
): Promise<{ finalMode: 1 | 2; downgraded: boolean; reason?: string }> {
  // Default to variable (2) if not specified
  if (!requestedMode || requestedMode === 2) {
    return { finalMode: 2, downgraded: false };
  }

  // User requested stable (1), check if supported
  const stableEnabled = token === 'DAI'; // Only DAI supports stable borrow

  if (stableEnabled) {
    return { finalMode: 1, downgraded: false };
  } else {
    // Downgrade to variable with explanation
    return {
      finalMode: 2,
      downgraded: true,
      reason: `该资产(${token})不支持固定利率借款，已自动切换为浮动利率(variable)`
    };
  }
}
```

**If downgrade occurs:**

```
⚠️ 利率模式调整

{token}不支持固定利率，已为您切换为浮动利率(variable)。

当前浮动利率: X.XX%
```

### Step 6: Generate Deep Link or Manual Path

#### Deep Link Format

> **⚠️ 链路验证声明**: AAVE官方未提供稳定的deeplink文档，以下URL模板基于接口逆向分析，不承诺长期有效。

```
https://app.aave.com/?marketName={market}&token={token}&amount={amount}&action={action}
```

**Market mapping:**
- Ethereum: `proto_mainnet_v3`
- Arbitrum: `proto_arbitrum_v3`

**Action mapping:**
- supply → `supply`
- borrow → `borrow`
- repay → `repay`
- withdraw → `withdraw`

#### Deep Link Validation Threshold

Before using deep links as the primary method:

1. Test 5 cases on Ethereum (USDC/WETH/WBTC/DAI/USDT), pre-fill success rate >= 80%
2. Test 5 cases on Arbitrum (same assets), pre-fill success rate >= 80%
3. Record test timestamp and detailed results

**Decision:**
- If both chains >= 80%: Use deep link as primary
- If either chain < 80%: Use manual path as default, deep link as fallback

#### Manual Path (Fallback)

If deep links are not reliable or user prefers manual:

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

**Parameter summary for manual entry:**

```
操作参数:
- 网络: {chainName}
- 资产: {token}
- 动作: {action}
- 金额: {amount}
- 利率模式: {interestRateMode === 1 ? '固定利率' : '浮动利率'}
```

### Step 7: Present Output

Format the response with:

1. **Summary** of the operation parameters
2. **Current APY** (from Step 4)
3. **Deep link** or manual path
4. **Risk considerations** (for borrow)
5. **Next steps**

**Example output format for supply:**

```markdown
## AAVE Operation Summary

| Parameter | Value |
|-----------|-------|
| Action | Supply |
| Asset | USDC |
| Amount | 100 USDC |
| Network | Arbitrum |
| Supply APY | 3.45% |
| Pool Utilization | 78.5% |

### Execution

**[Click here to open AAVE]({deep_link})**

Or copy this URL: `{deep_link}`

### Notes

- Please review all details in the AAVE interface before confirming
- Gas fees will apply for the transaction
- Your supplied assets will start earning interest immediately
- APY is variable and changes with market conditions
```

**Example output format for borrow:**

```markdown
## AAVE Operation Summary

| Parameter | Value |
|-----------|-------|
| Action | Borrow |
| Asset | USDC |
| Amount | 1,000 USDC |
| Network | Ethereum |
| Interest Rate Mode | Variable |
| Variable Borrow APY | 5.67% |
| Pool Utilization | 78.5% |

### Execution

**[Click here to open AAVE]({deep_link})**

### Risk Warning

⚠️ Borrow Risk Warning

- Monitor your Health Factor to avoid liquidation
- Variable rates can increase with market utilization
- Current utilization is 78.5% (High - rates may be volatile)
- Ensure you have sufficient collateral
```

## Input Validation

Before processing any request:

- **Token symbols**: Must be in whitelist (case-insensitive match)
- **Amounts**: Must be positive numbers (match: `^[0-9]+\.?[0-9]*$`)
- **Chain IDs**: Must be 1 or 42161
- **Actions**: Must be one of: supply, borrow, repay, withdraw

## Failure Handling

### Class 1: Missing/Invalid Parameters

| Scenario | Input | Behavior |
|----------|-------|----------|
| Missing all params | "use aave" | Ask for action |
| Invalid amount | "supply -100 USDC" | Prompt: amount must be positive |
| Invalid token | "supply ABC" | Show whitelist, suggest alternatives |
| Invalid chain | "supply on Bitcoin" | Prompt: only Ethereum/Arbitrum supported |

### Class 2: On-Chain State Restrictions

| Scenario | Behavior |
|----------|----------|
| Stable borrow not supported | Downgrade to variable, notify user |
| Non-whitelist asset | Block transaction, show warning |

### Class 3: Runtime/Network Errors

| Scenario | Behavior |
|----------|----------|
| RPC timeout | Prompt: network error, provide manual path |
| AskUserQuestion unavailable | Use natural language conversation |
| Deep link fails | Fall back to manual steps |

## Important Considerations

### Interest Rate Modes

- **Variable Rate**: Rate fluctuates based on market utilization. Default for all assets.
- **Stable Rate**: Rate fixed at borrow time, predictable payments. Only available for DAI.

### Risk Warnings for Borrow

When generating borrow operations:

```
⚠️ **Borrow Risk Warning**

- Monitor your Health Factor to avoid liquidation
- Variable rates can increase with market conditions
- Ensure you have sufficient collateral
- Consider the liquidation threshold for your collateral assets
```

### Gas Estimation

Gas costs vary by chain:
- **Ethereum**: Higher gas, most secure
- **Arbitrum**: ~10x lower gas than Ethereum

### Slippage and Price Impact

For large operations, advise users to:
- Check current market conditions
- Consider splitting into smaller transactions
- Review all details before confirming

## Position Simulation

When users want to preview how an action would affect their position before executing:

**Trigger phrases:**
- "如果我再借 X USDC，Health Factor 会降到多少？"
- "simulate borrow 1000 USDC"
- "what if I supply 2 ETH?"
- "如果我提供 Y ETH 作为抵押，可以借多少？"

**Workflow:**

1. Parse the simulation request (action, token, amount)
2. Get user's wallet address (from context or ask)
3. Run simulation script:

```bash
cd aave_skills && npx ts-node aave-planner/scripts/simulate-position.ts <chainId> <userAddress> <action> <token> <amount>
```

**Example:**
```bash
cd aave_skills && npx ts-node aave-planner/scripts/simulate-position.ts 1 0x1234... borrow USDC 1000
```

**Present results:**

```markdown
## Position Simulation Result

**Action:** Borrow 1,000 USDC on Ethereum

| Metric | Current | After | Change |
|--------|---------|-------|--------|
| Collateral | $10,000 | $10,000 | - |
| Debt | $2,000 | $3,000 | +$1,000 |
| Health Factor | 2.5 | 1.67 | -0.83 ⚠️ |
| Available Borrows | $5,000 | $4,000 | -$1,000 |
| Risk Level | Safe | Moderate | - |

**Recommendation:**
This borrow would reduce your Health Factor to 1.67. While still safe, monitor your position closely.
```

**Risk warnings in simulation:**
- HF < 1.5: "⚠️ Caution: Position would become HIGH RISK"
- HF < 1.2: "🚨 WARNING: Position would be near LIQUIDATION"
- HF < 1.0: "❌ This action would trigger LIQUIDATION"

## Out-of-Scope Actions

The following actions are **not supported** in the first version:

- eMode management
- Isolation mode operations
- Flash loans
- Liquidations
- Governance operations
- Collateral switching (swap collateral without withdrawing)

**If requested:**

```
⚠️ {action}不在首批功能范围

首批仅支持以下操作：
- Supply (存款)
- Borrow (借款)
- Repay (还款)
- Withdraw (取款)
- Simulate (模拟预测)

{action}支持预计在二期迭代中推出。
```

## Additional Resources

### Reference Files

- **`references/token-address-book.md`** - Token addresses and configurations
- **`references/market-config.md`** - AAVE V3 contract addresses
- **`references/test-scenarios.md`** - Complete test scenarios and acceptance criteria

### External Resources

- AAVE V3 Documentation: https://docs.aave.com/
- AAVE App: https://app.aave.com

## Strategy Planning (Added)

For higher-level position planning, use:

```bash
cd aave_skills && npx ts-node aave-planner/scripts/plan-position-strategy.ts \
  <chainId> <userAddress> <goal> <collateralToken> <borrowToken>
```

- `goal`: `conservative` | `balanced` | `aggressive`
- Output includes suggested target range, step-by-step actions, and safety fallbacks.
