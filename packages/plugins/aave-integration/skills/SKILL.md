---
name: aave-integration
description: This skill should be used when the user needs to interact with AAVE V3 protocol contracts directly, read on-chain data, get reserve configurations, fetch current APY rates, simulate position changes, or execute protocol operations programmatically. Provides low-level access to AAVE Pool contracts, UI Pool Data Provider, and quote generation for supply, borrow, repay, and withdraw operations on Ethereum and Arbitrum.
---

# AAVE V3 Integration

Low-level integration with AAVE V3 protocol contracts for reading on-chain data and generating operation quotes.

## Overview

This skill provides:

1. **Contract Interface Definitions** - ABIs for AAVE V3 contracts
2. **Quote Generation** - Scripts to get supply, borrow, repay, withdraw quotes
3. **APY Data** - Scripts to fetch current supply/borrow APY for all assets
4. **Position Simulation** - Scripts to preview how actions affect Health Factor and risk
5. **Reserve Configuration** - Reading asset parameters from chain
6. **User Account Data** - Reading user positions and health metrics

## Contract Addresses

### Ethereum Mainnet (chainId: 1)

| Contract | Address |
|----------|---------|
| PoolAddressesProvider | `0x2f39d218133AFaB8F2B819B1066c7E434Ad94E9e` |
| Pool (Proxy) | `0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2` |
| UiPoolDataProvider | `0x91c0eA31b49B69Ea18607702c5d9aC360bf3dE7d` |
| PoolDataProvider | `0x7B4EB56E7CD4b454BA8ff71E4518426369a138a3` |

### Arbitrum One (chainId: 42161)

| Contract | Address |
|----------|---------|
| PoolAddressesProvider | `0xa97684ead0e402dC232d5A977953DF7ECBaB3CDb` |
| Pool (Proxy) | `0x794a61358D6845594F94dc1DB02A252b5b4814aD` |
| UiPoolDataProvider | `0x5c5228aC8BC1528482514aF3e27D692c20E5c41F` |
| PoolDataProvider | `0x69FA688f1Dc47d4B5d8029D5a35FB7a548310654` |

## ABI Definitions

### Pool ABI (Core Operations)

```typescript
const poolAbi = parseAbi([
  // Supply
  'function supply(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external',
  // Withdraw
  'function withdraw(address asset, uint256 amount, address to) external returns (uint256)',
  // Borrow
  'function borrow(address asset, uint256 amount, uint256 interestRateMode, uint16 referralCode, address onBehalfOf) external',
  // Repay
  'function repay(address asset, uint256 amount, uint256 interestRateMode, address onBehalfOf) external returns (uint256)',
  // User data
  'function getUserAccountData(address user) view returns (uint256 totalCollateralBase, uint256 totalDebtBase, uint256 availableBorrowsBase, uint256 currentLiquidationThreshold, uint256 ltv, uint256 healthFactor)',
  // Reserve data
  'function getReserveData(address asset) view returns (tuple(uint256 configuration, uint128 liquidityIndex, uint128 currentLiquidityRate, uint128 variableBorrowIndex, uint128 currentVariableBorrowRate, uint128 currentStableBorrowRate, uint40 lastUpdateTimestamp, uint16 id, address aTokenAddress, address stableDebtTokenAddress, address variableDebtTokenAddress, address interestRateStrategyAddress, uint128 accruedToTreasury, uint128 unbacked, uint128 isolationModeTotalDebt))'
]);
```

### UiPoolDataProvider ABI

```typescript
const uiPoolDataProviderAbi = parseAbi([
  {
    "inputs": [
      { "internalType": "contract IPoolAddressesProvider", "name": "provider", "type": "address" }
    ],
    "name": "getReservesData",
    "outputs": [
      {
        "components": [
          { "internalType": "address", "name": "underlyingAsset", "type": "address" },
          { "internalType": "string", "name": "name", "type": "string" },
          { "internalType": "string", "name": "symbol", "type": "string" },
          { "internalType": "uint256", "name": "liquidityRate", "type": "uint256" },
          { "internalType": "uint256", "name": "variableBorrowRate", "type": "uint256" },
          { "internalType": "uint256", "name": "stableBorrowRate", "type": "uint256" },
          { "internalType": "uint256", "name": "averageStableRate", "type": "uint256" },
          { "internalType": "uint256", "name": "baseLTVasCollateral", "type": "uint256" },
          { "internalType": "uint256", "name": "reserveLiquidationThreshold", "type": "uint256" },
          { "internalType": "uint256", "name": "reserveLiquidationBonus", "type": "uint256" },
          { "internalType": "bool", "name": "usageAsCollateralEnabled", "type": "bool" },
          { "internalType": "bool", "name": "borrowingEnabled", "type": "bool" },
          { "internalType": "bool", "name": "stableBorrowRateEnabled", "type": "bool" },
          { "internalType": "bool", "name": "isActive", "type": "bool" },
          { "internalType": "bool", "name": "isFrozen", "type": "bool" }
        ],
        "internalType": "struct IUiPoolDataProviderV3.AggregatedReserveData[]",
        "name": "",
        "type": "tuple[]"
      }
    ],
    "stateMutability": "view",
    "type": "function"
  }
]);
```

## Quote Interfaces

### Supply Quote

```typescript
interface SupplyQuote {
  token: string;
  tokenAddress: string;
  amount: string;
  amountWei: string;
  apy: string;              // Current supply APY
  aTokenAddress: string;    // Corresponding aToken
  usageRatio: string;       // Pool utilization ratio
  totalLiquidity: string;   // Total liquidity in pool
}
```

### Borrow Quote

```typescript
interface BorrowQuote {
  token: string;
  tokenAddress: string;
  amount: string;
  amountWei: string;
  apyVariable: string;      // Variable borrow rate
  apyStable?: string;       // Fixed rate (only if supported)
  stableBorrowEnabled: boolean;
  availableLiquidity: string;
  ltv: number;              // Loan-to-Value ratio
  liquidationThreshold: number;
  liquidationPenalty: number;
}
```

### Repay Quote

```typescript
interface RepayQuote {
  token: string;
  tokenAddress: string;
  amount: string;
  amountWei: string;
  currentDebt: string;      // Current total debt for this asset
  interestRateMode: 1 | 2;  // 1=stable, 2=variable
  isMaxRepay: boolean;      // true if repaying full debt
}
```

### Withdraw Quote

```typescript
interface WithdrawQuote {
  token: string;
  tokenAddress: string;
  amount: string;
  amountWei: string;
  maxWithdrawable: string;  // Maximum withdrawable amount
  currentSupplied: string;  // Current aToken balance
  remainingCollateral: string;
}
```

## Reading Reserve Configuration

```typescript
// IMPORTANT: Must explicitly pass chainId, ensure it matches the calling context
async function getReserveConfigurationData(
  tokenAddress: string,
  chainId: 1 | 42161
): Promise<{
  ltv: number;                      // Basis points (e.g., 8000 = 80%)
  liquidationThreshold: number;     // Basis points
  liquidationBonus: number;         // Basis points
  stableBorrowRateEnabled: boolean;
  borrowingEnabled: boolean;
  usageAsCollateralEnabled: boolean;
  isActive: boolean;
  isFrozen: boolean;
}> {
  const uiPoolDataProvider = getContract({
    address: UI_POOL_DATA_PROVIDER[chainId],
    abi: uiPoolDataProviderAbi,
    client: publicClient
  });

  const [reservesData] = await uiPoolDataProvider.read.getReservesData([
    POOL_ADDRESSES_PROVIDER[chainId]
  ]);

  const reserve = reservesData.find(r =>
    r.underlyingAsset.toLowerCase() === tokenAddress.toLowerCase()
  );

  if (!reserve) throw new Error(`Reserve not found for ${tokenAddress}`);

  return {
    ltv: Number(reserve.baseLTVasCollateral),
    liquidationThreshold: Number(reserve.reserveLiquidationThreshold),
    liquidationBonus: Number(reserve.reserveLiquidationBonus),
    stableBorrowRateEnabled: reserve.stableBorrowRateEnabled,
    borrowingEnabled: reserve.borrowingEnabled,
    usageAsCollateralEnabled: reserve.usageAsCollateralEnabled,
    isActive: reserve.isActive,
    isFrozen: reserve.isFrozen
  };
}
```

## Reading User Account Data

```typescript
async function getUserAccountData(
  userAddress: string,
  chainId: 1 | 42161
): Promise<{
  totalCollateralUSD: number;
  totalDebtUSD: number;
  availableBorrowsUSD: number;
  currentLiquidationThreshold: number;
  ltv: number;
  healthFactor: number;
}> {
  const pool = getContract({
    address: POOL_ADDRESSES[chainId],
    abi: poolAbi,
    client: publicClient
  });

  const data = await pool.read.getUserAccountData([userAddress as `0x${string}`]);

  return {
    totalCollateralUSD: Number(data.totalCollateralBase) / 1e8,
    totalDebtUSD: Number(data.totalDebtBase) / 1e8,
    availableBorrowsUSD: Number(data.availableBorrowsBase) / 1e8,
    currentLiquidationThreshold: Number(data.currentLiquidationThreshold) / 10000,
    ltv: Number(data.ltv) / 10000,
    healthFactor: Number(data.healthFactor) / 1e18
  };
}
```

## Data Source Priority

When fetching AAVE data:

1. **Primary**: On-chain reads (viem/cast direct contract calls)
2. **Secondary**: AAVE official Subgraph
3. **Tertiary**: Third-party aggregators (DeFiLlama, etc.)
4. **Conflict Resolution**: On-chain data is authoritative

Always disclose data source in outputs when using non-primary sources.

## Input Validation

Before any contract interaction:

- **Token addresses**: Must match `^0x[a-fA-F0-9]{40}$`
- **Chain IDs**: Must be 1 or 42161
- **Amounts**: Must be valid decimal numbers
- **User addresses**: Must be valid Ethereum addresses

**Reject any input containing shell metacharacters** (`;`, `|`, `$`, `` ` ``, `&`, `(`, `)`, `>`, `<`, `\`)

## Error Handling

### Contract Call Failures

| Error | Cause | Response |
|-------|-------|----------|
| `Reserve not found` | Token not listed on AAVE | Check token address, verify on app.aave.com |
| `RPC timeout` | Network connectivity | Retry with exponential backoff |
| `Invalid call data` | ABI mismatch | Verify ABI version matches contract |

### Rate Limiting

For RPC calls:
- Implement request batching where possible
- Use exponential backoff for retries
- Consider using dedicated RPC endpoints for high-volume operations

## Token Address Reference

### Ethereum (chainId: 1)

| Symbol | Address | Decimals |
|--------|---------|----------|
| USDC | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` | 6 |
| USDT | `0xdAC17F958D2ee523a2206206994597C13D831ec7` | 6 |
| WETH | `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` | 18 |
| WBTC | `0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599` | 8 |
| DAI | `0x6B175474E89094C44Da98b954EedeAC495271d0F` | 18 |

### Arbitrum (chainId: 42161)

| Symbol | Address | Decimals |
|--------|---------|----------|
| USDC | `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` | 6 |
| USDT | `0xFd086bC7CD5C481DCC9C85ebE478A1C0b69FCbb9` | 6 |
| WETH | `0x82aF49447D8a07e3bd95BD0d56f35241523fBab1` | 18 |
| WBTC | `0x2f2a2543B76A4166549F7aaB2e75Bef0aefC5B0f` | 8 |
| DAI | `0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1` | 18 |

## Scripts

### quote-apy.ts

Fetches current APY data for all whitelisted assets.

**Prerequisites:** `cd aave_skills && npm install`

```bash
cd aave_skills && npx ts-node aave-integration/scripts/quote-apy.ts <chainId>
# Example: cd aave_skills && npx ts-node aave-integration/scripts/quote-apy.ts 1
```

**Output:** JSON with supplyApy, variableBorrowApy (null - see note), stableBorrowApy (null), utilization (null), and note per asset.

**Data source limitations:**
- Uses DefiLlama API for supply APY
- Borrow APY and utilization not available from this API (return null)
- For complete data, visit https://app.aave.com

### simulate-position.ts

Simulates how an action would affect a user's position.

**Prerequisites:** `cd aave_skills && npm install`

```bash
cd aave_skills && npx ts-node aave-integration/scripts/simulate-position.ts <chainId> <userAddress> <action> <token> <amount>
# Example: cd aave_skills && npx ts-node aave-integration/scripts/simulate-position.ts 1 0x... borrow USDC 1000
```

**Output:** Current position, simulated position, changes, and risk level assessment.

## Additional Resources

### Reference Files

- **`references/token-address-book.md`** - Complete token addresses and aToken mappings
- **`references/market-config.md`** - Contract addresses and market configuration
- **`references/health-factor.md`** - Health factor calculation details
- **`references/risk-thresholds.md`** - Risk parameters and liquidation thresholds

### External Documentation

- AAVE V3 Developer Docs: https://docs.aave.com/developers
- AAVE V3 Core Contracts: https://github.com/aave/aave-v3-core
- AAVE Address Book: https://github.com/aave/aave-address-book

## Execution Scripts (Added)

Execution-capable scripts are available under `aave-integration/scripts` for end-to-end flow:

- `execute-approve.ts`
- `execute-supply.ts`
- `execute-borrow.ts`
- `execute-repay.ts`
- `execute-withdraw.ts`

All execution scripts support a normalized output shape:

```json
{
  "ok": true,
  "action": "supply",
  "chainId": 1,
  "token": "0x...",
  "amount": "100",
  "txHash": "0x...",
  "receipt": {
    "status": "success",
    "gasUsed": "...",
    "blockNumber": "...",
    "transactionHash": "0x..."
  },
  "warnings": []
}
```

Use `--dryRun` to estimate without broadcasting:

```bash
cd aave_skills && npx ts-node aave-integration/scripts/execute-borrow.ts \
  --chainId 1 --token USDC --amount 100 --account 0x... --dryRun
```
