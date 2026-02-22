---
name: aave-risk-assessor
description: This skill should be used when the user asks about "health factor", "liquidation risk", "aave risk", "will I be liquidated", "safe to borrow", "my account health", "collateral risk", "liquidation price", or wants to assess the risk of their AAVE V3 position. Calculates health factor, LTV ratios, liquidation thresholds, and provides risk level assessments for positions on Ethereum and Arbitrum.
---

# AAVE V3 Risk Assessor

Assess risk metrics for AAVE V3 positions including Health Factor, LTV ratios, and liquidation risk.

> **Runtime Compatibility:** This skill uses `AskUserQuestion` for interactive prompts. If `AskUserQuestion` is not available in your runtime, collect the same parameters through natural language conversation instead.

## Overview

Calculate and interpret risk metrics for AAVE V3 positions:

1. **Health Factor (HF)** - Primary liquidation risk indicator
2. **LTV (Loan-to-Value)** - Current and maximum borrowing capacity
3. **Liquidation Threshold** - Point at which liquidation becomes possible
4. **Risk Level Classification** - Safe, Moderate, High, Critical, Liquidation

## Trigger Phrases

This skill should be invoked when users say:

- "health factor"
- "liquidation risk"
- "aave risk"
- "will I be liquidated"
- "safe to borrow"
- "my account health"
- "collateral risk"
- "liquidation price"
- "账户健康度"
- "清算风险"

## Risk Metrics

### Health Factor

The Health Factor is a numeric representation of position safety:

```
HF = (Σ Collateral_i × LiquidationThreshold_i) / TotalDebt
```

| Health Factor | Risk Level | Description | Recommended Action |
|---------------|------------|-------------|-------------------|
| > 2.0 | Safe | Position is well-collateralized | Normal operation |
| 1.5 - 2.0 | Moderate | Position is healthy but monitor | Continue monitoring |
| 1.2 - 1.5 | High | Position is at risk | Consider adding collateral or repaying debt |
| 1.0 - 1.2 | Critical | Position is near liquidation | Urgent: Add collateral or repay immediately |
| < 1.0 | Liquidation | Position can be liquidated | Position is being or will be liquidated |

### Risk Level Implementation

```typescript
function getRiskLevel(healthFactor: number): RiskLevel {
  if (healthFactor > 2.0) return 'safe';
  if (healthFactor >= 1.5) return 'moderate';
  if (healthFactor >= 1.2) return 'high';
  if (healthFactor >= 1.0) return 'critical';
  return 'liquidation';
}

type RiskLevel = 'safe' | 'moderate' | 'high' | 'critical' | 'liquidation';

const riskLevelMessages: Record<RiskLevel, string> = {
  safe: 'Your position is well-collateralized.',
  moderate: 'Your position is healthy, but continue monitoring.',
  high: 'Your position is at risk. Consider adding collateral or repaying debt.',
  critical: 'Warning: Your position is near liquidation! Add collateral or repay immediately.',
  liquidation: 'Your position is eligible for liquidation.'
};
```

## Risk Assessment Interface

```typescript
interface RiskAssessment {
  // Core metrics
  healthFactor: string;           // Current health factor (e.g., "1.85")
  maxLTV: string;                 // Maximum LTV allowed (e.g., "0.80")
  currentLTV: string;             // Current LTV ratio (e.g., "0.45")
  liquidationThreshold: string;   // Liquidation threshold (e.g., "0.825")
  liquidationPenalty: string;     // Liquidation penalty (e.g., "0.05")

  // eMode status
  eModeStatus: boolean;           // Whether eMode is enabled
  eModeCategory?: number;         // eMode category ID if enabled

  // Risk classification
  riskLevel: 'safe' | 'moderate' | 'high' | 'critical' | 'liquidation';

  // Additional metrics
  totalCollateralUSD: string;     // Total collateral value in USD
  totalDebtUSD: string;           // Total debt value in USD
  availableBorrowsUSD: string;    // Available borrowing power in USD
}
```

## Workflow

### Step 1: Get Wallet Address

If not provided in context, ask the user:

```json
{
  "questions": [
    {
      "question": "Please provide your wallet address to check your AAVE position",
      "header": "Wallet Address",
      "options": [],
      "multiSelect": false
    }
  ]
}
```

**Input validation:**
- Address must match `^0x[a-fA-F0-9]{40}$`
- Must be a valid checksummed Ethereum address

### Step 2: Determine Chain

If not specified, ask which chain to check:

```json
{
  "questions": [
    {
      "question": "Which network is your AAVE position on?",
      "header": "Network",
      "options": [
        { "label": "Ethereum", "description": "Check Ethereum Mainnet position" },
        { "label": "Arbitrum", "description": "Check Arbitrum One position" }
      ],
      "multiSelect": false
    }
  ]
}
```

### Step 3: Query On-Chain Data

Use the Pool contract to get user account data:

```typescript
import { createPublicClient, http, parseAbi } from 'viem';
import { mainnet, arbitrum } from 'viem/chains';

const POOL_ADDRESSES = {
  1: '0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2',    // Ethereum
  42161: '0x794a61358D6845594F94dc1DB02A252b5b4814aD'  // Arbitrum
};

const poolAbi = parseAbi([
  'function getUserAccountData(address user) view returns (uint256 totalCollateralBase, uint256 totalDebtBase, uint256 availableBorrowsBase, uint256 currentLiquidationThreshold, uint256 ltv, uint256 healthFactor)'
]);

async function getRiskAssessment(
  userAddress: string,
  chainId: 1 | 42161
): Promise<RiskAssessment> {
  const client = createPublicClient({
    chain: chainId === 1 ? mainnet : arbitrum,
    transport: http()
  });

  const pool = getContract({
    address: POOL_ADDRESSES[chainId],
    abi: poolAbi,
    client
  });

  const data = await pool.read.getUserAccountData([userAddress as `0x${string}`]);

  const healthFactor = Number(data.healthFactor) / 1e18;
  const totalCollateralUSD = Number(data.totalCollateralBase) / 1e8;
  const totalDebtUSD = Number(data.totalDebtBase) / 1e8;
  const availableBorrowsUSD = Number(data.availableBorrowsBase) / 1e8;
  const liquidationThreshold = Number(data.currentLiquidationThreshold) / 10000;
  const ltv = Number(data.ltv) / 10000;

  // Calculate current LTV
  const currentLTV = totalCollateralUSD > 0
    ? totalDebtUSD / totalCollateralUSD
    : 0;

  return {
    healthFactor: healthFactor.toFixed(2),
    maxLTV: ltv.toFixed(2),
    currentLTV: currentLTV.toFixed(2),
    liquidationThreshold: liquidationThreshold.toFixed(3),
    liquidationPenalty: '0.05', // Default 5%, varies by asset
    eModeStatus: false, // Would need additional query
    riskLevel: getRiskLevel(healthFactor),
    totalCollateralUSD: totalCollateralUSD.toFixed(2),
    totalDebtUSD: totalDebtUSD.toFixed(2),
    availableBorrowsUSD: availableBorrowsUSD.toFixed(2)
  };
}
```

### Step 4: Calculate Liquidation Price

For each collateral asset, calculate the price at which liquidation would occur:

```typescript
function calculateLiquidationPrice(
  collateralAmount: number,
  collateralPrice: number,
  liquidationThreshold: number,
  totalDebtUSD: number
): number {
  // HF = 1.0 when: (CollateralValue × LT) / Debt = 1
  // CollateralValue = Debt / LT
  // LiquidationPrice = (Debt / LT) / CollateralAmount

  const liquidationCollateralValue = totalDebtUSD / liquidationThreshold;
  return liquidationCollateralValue / collateralAmount;
}

// Example:
// - Collateral: 10 WETH @ $2000 = $20,000
// - Debt: $10,000 USDC
// - WETH Liquidation Threshold: 82.5%
// - Liquidation Price = $10,000 / 0.825 / 10 = $1,212.12 per WETH
```

### Step 5: Present Risk Assessment

Format the output with clear risk indicators:

```markdown
## AAVE Position Risk Assessment

### Risk Summary

| Metric | Value | Status |
|--------|-------|--------|
| Health Factor | {healthFactor} | {riskEmoji} {riskLevel} |
| Total Collateral | ${totalCollateralUSD} | - |
| Total Debt | ${totalDebtUSD} | - |
| Available to Borrow | ${availableBorrowsUSD} | - |

### Risk Details

| Metric | Value | Description |
|--------|-------|-------------|
| Current LTV | {currentLTV}% | Your debt/collateral ratio |
| Maximum LTV | {maxLTV}% | Maximum allowed borrowing |
| Liquidation Threshold | {liquidationThreshold}% | Point of liquidation eligibility |
| Liquidation Penalty | {liquidationPenalty}% | Bonus paid to liquidators |

### Risk Assessment

{riskLevelMessage}

### Recommendations

{recommendations}
```

**Risk Emojis:**
- Safe: ✅
- Moderate: ℹ️
- High: ⚠️
- Critical: 🚨
- Liquidation: ❌

### Step 6: Provide Recommendations

Based on risk level:

**Safe (HF > 2.0):**
```
✅ Your position is in good health.

You can:
- Continue normal operations
- Consider increasing leverage if desired
- Monitor market conditions regularly
```

**Moderate (1.5 - 2.0):**
```
ℹ️ Your position is healthy, but keep monitoring.

Recommendations:
- Set price alerts for significant market moves
- Consider the impact of market volatility
- Have a plan for adding collateral if needed
```

**High (1.2 - 1.5):**
```
⚠️ Your position is at risk.

Recommended actions:
1. Add more collateral to increase Health Factor
2. Repay a portion of your debt
3. Consider reducing exposure to volatile assets

Target: Increase Health Factor to > 1.5
```

**Critical (1.0 - 1.2):**
```
🚨 WARNING: Your position is near liquidation!

URGENT actions needed:
1. Add collateral IMMEDIATELY
2. Repay debt to increase Health Factor
3. Consider withdrawing from other positions

Target: Increase Health Factor to > 1.2 as soon as possible
```

**Liquidation (HF < 1.0):**
```
❌ Your position is eligible for liquidation.

What this means:
- Anyone can liquidate your position
- Liquidators will repay your debt and seize your collateral
- You will receive any remaining collateral after liquidation

Actions:
- Add collateral immediately if possible
- Partial liquidation may occur to bring HF back to safe levels
```

## Asset-Specific Risk Parameters

### Ethereum Mainnet (Chain ID: 1)

| Asset | LTV | Liquidation Threshold | Liquidation Penalty |
|-------|-----|----------------------|---------------------|
| USDC | 77% | 80% | 5% |
| USDT | 75% | 80% | 5% |
| DAI | 77% | 80% | 5% |
| WETH | 80% | 82.5% | 5% |
| WBTC | 73% | 78% | 7.5% |

### Arbitrum (Chain ID: 42161)

| Asset | LTV | Liquidation Threshold | Liquidation Penalty |
|-------|-----|----------------------|---------------------|
| USDC | 80% | 85% | 5% |
| USDT | 75% | 80% | 5% |
| DAI | 75% | 80% | 5% |
| WETH | 80% | 82.5% | 5% |
| WBTC | 73% | 78% | 7.5% |

## eMode (Efficiency Mode) Considerations

### eMode Categories

| Category ID | Name | Description | Max LTV | Liquidation Threshold |
|-------------|------|-------------|---------|----------------------|
| 0 | None | Default - No eMode | Asset-specific | Asset-specific |
| 1 | Stablecoins | USD-pegged stablecoins | 97% | 97.5% |
| 2 | ETH Correlated | ETH and ETH-pegged assets | 93% | 95% |

### eMode Risk Implications

1. **Higher Leverage**: eMode allows higher LTV (up to 97% for stablecoins), increasing liquidation risk
2. **Correlated Asset Risk**: Assets in the same eMode category are treated as correlated
3. **Category Exit Risk**: Exiting an eMode category may immediately lower your effective LTV

**Note**: eMode management is not supported in the first version. Display current eMode status if detected.

## Liquidation Scenarios

### Scenario 1: Price Drop Triggering Liquidation

```
Initial State:
- Collateral: 10 WETH @ $2000 = $20,000
- Debt: 10,000 USDC
- WETH Liquidation Threshold: 82.5%
- Health Factor: ($20,000 × 0.825) / $10,000 = 1.65

Price Drop Scenario:
- WETH price drops to $1200
- Collateral value: $12,000
- Health Factor: ($12,000 × 0.825) / $10,000 = 0.99

Result: Position is now eligible for liquidation
```

### Scenario 2: Borrowing Up to Limit

```
Initial State:
- Collateral: 10 WETH @ $2000 = $20,000
- WETH LTV: 80%
- Max borrow: $20,000 × 0.80 = $16,000

User borrows $15,000 USDC:
- Current LTV: $15,000 / $20,000 = 75%
- Health Factor: ($20,000 × 0.825) / $15,000 = 1.10 (Critical)

Small price drop (-10%):
- WETH @ $1800, Collateral = $18,000
- Health Factor: ($18,000 × 0.825) / $15,000 = 0.99 (Liquidation)
```

## Advanced: Simulate Position Changes

Allow users to simulate how actions would affect their risk:

```typescript
function simulateBorrow(
  currentAssessment: RiskAssessment,
  borrowAmountUSD: number,
  collateralLTV: number
): RiskAssessment {
  const newDebtUSD = Number(currentAssessment.totalDebtUSD) + borrowAmountUSD;
  const collateralUSD = Number(currentAssessment.totalCollateralUSD);

  const newHealthFactor = collateralUSD > 0
    ? (collateralUSD * Number(currentAssessment.liquidationThreshold)) / newDebtUSD
    : Infinity;

  return {
    ...currentAssessment,
    healthFactor: newHealthFactor.toFixed(2),
    totalDebtUSD: newDebtUSD.toFixed(2),
    currentLTV: (newDebtUSD / collateralUSD).toFixed(2),
    riskLevel: getRiskLevel(newHealthFactor)
  };
}
```

## Error Handling

### Empty Position

If the user has no AAVE position:

```
ℹ️ No AAVE position found

This wallet does not have any active positions on AAVE V3 {Network}.

To get started:
- Supply collateral to enable borrowing
- Visit https://app.aave.com to open a position
```

### Network Errors

```
⚠️ Unable to fetch on-chain data

Possible causes:
- Network connectivity issues
- RPC node temporarily unavailable

Please try again later or check your position directly at https://app.aave.com
```

## Additional Resources

### Reference Files

- **`references/health-factor.md`** - Detailed Health Factor calculation
- **`references/risk-thresholds.md`** - Complete risk parameters by asset
- **`references/token-address-book.md`** - Token addresses and configurations
- **`references/market-config.md`** - Contract addresses

### External Resources

- AAVE V3 Risk Documentation: https://docs.aave.com/risk/
- AAVE Liquidations: https://docs.aave.com/developers/guides/liquidations
- AAVE App: https://app.aave.com
