---
sidebar_position: 1
sidebar_label: Making a Trade 
---

# Making a Trade 

This guide demonstrates how to execute a swap. We will be swapping 20 USDC for AVAX.

### 1. Required imports for this guide
```js
import { PairV2, RouteV2, TradeV2, LB_ROUTER_ADDRESS, LBRouterABI } from '@traderjoe-xyz/sdk-v2'
import { Token, ChainId, WAVAX: _WAVAX, TokenAmount, JSBI, Percent} from '@traderjoe-xyz/sdk'
import { Wallet } from 'ethers'
import { Contract } from '@ethersproject/contracts'
import { parseUnits } from '@ethersproject/units'
import { JsonRpcProvider } from '@ethersproject/providers'
```

### 2. Declare required constants
```js
// declare chainId, provider, and signer 
const FUJI_URL = 'https://api.avax-test.network/ext/bc/C/rpc'
const CHAIN_ID = ChainId.FUJI
const PROVIDER = new JsonRpcProvider(FUJI_URL)
const WALLET_PK = "{WALLET_PRIVATE_KEY}"
const SIGNER = new Wallet(WALLET_PK, PROVIDER)
const ACCOUNT = await SIGNER.getAddress()

// initialize tokens
const WAVAX = _WAVAX[CHAIN_ID] // Token instance of WAVAX
const USDC = new Token(
    CHAIN_ID,
    '0xB6076C93701D6a07266c31066B298AeC6dd65c2d',
    6,
    'USDC',
    'USD Coin'
)
const USDT = new Token(
    CHAIN_ID,
    '0xAb231A5744C8E6c45481754928cCfFFFD4aa0732',
    6,
    'USDT.e',
    'Tether USD'
)

// declare bases used to generate trade routes
const BASES = [WAVAX, USDC, USDT] 
```

### 3. Declare user inputs and initialize TokenAmount
```js
// the input token in the trade
const inputToken = USDC

// the expected output token in the trade
const outputToken = WAVAX

// specify whether user gives an exact inputToken or outputToken value for the trade
const isExactIn = true

// user string input; in this case representing 20 USDC
const typedValueIn = '20' 

// parse user input into inputToken's decimal precision, which is 6 for USDC
const typedValueInParsed = parseUnits( 
  typedValueIn, 
  inputToken.decimals
).toString() // returns 20000000

// wrap into TokenAmount
const amountIn = new TokenAmount(
  inputToken, 
  JSBI.BigInt(typedValueInParsed)
) 
```

### 4. Use PairV2 and RouteV2 functions to generate all possible routes
```js
// get all [Token, Token] combinations 
const allTokenPairs = PairV2.createAllTokenPairs(
  inputToken,
  outputToken,
  BASES
)

// init PairV2 instances for the [Token, Token] pairs
const allPairs = PairV2.initPairs(allTokenPairs) 

// generates all possible routes to consider
const allRoutes = RouteV2.createAllRoutes(
  allPairs,
  inputToken,
  outputToken,
  2 // maxHops 
) 
```

### 5. Generate TradeV2 instances and get the best trade
```js
const isAvaxIn = false
const isAvaxOut = true // set to 'true' if swapping for AVAX; otherwise, 'false'

// generates all possible TradeV2 instances
const trades = await TradeV2.getTradesExactIn(
  allRoutes,
  amountIn,
  outputToken,
  isAvaxIn,
  isAvaxOut, 
  provider,
  chainId
) 

// chooses the best trade 
const bestTrade = TradeV2.chooseBestTrade(trades, isExactIn)
```

### 6. Check the trade information
```js
// print useful information about the trade, such as the quote, executionPrice, fees, etc
console.log(bestTrade.toLog())

// get trade fee information
const { totalFeePct, feeAmountIn } = await bestTrade.getTradeFee(provider)
console.log('Total fees percentage', totalFeePct.toSignificant(6), '%')
console.log(`Fee: ${feeAmountIn.toSignificant(6)} ${feeAmountIn.token.symbol}`)
```

### 7. Declare slippage tolerance and swap method/parameters
```js
// set slippage tolerance
const userSlippageTolerance = new Percent(JSBI.BigInt(50), JSBI.BigInt(10000)) // 0.5%

// set swap options
const swapOptions = {
  recipient: ACCOUNT, 
  allowedSlippage: userSlippageTolerance, 
  deadline: 1000, // in seconds
  feeOnTransfer: false // or true
}

// generate swap method and parameters for contract call
const {
  methodName, // swapExactTokensForAVAX,
  args, // [amountIn, amountOut, binSteps, path, to, deadline]
  value // 0x0
} = bestTrade.swapCallParameters(swapOptions)

```
### 8. Execute trade using Ethers.js
```js
// init router contract
const router = new Contract(
  LB_ROUTER_ADDRESS[CHAIN_ID],
  LBRouterABI,
  SIGNER
)

// estimate gas
const gasOptions = value && !isZero(value) ? { value } : {} 
const gasEstimate = await router.estimateGas[methodName](...args, options)

// execute swap
const options = value && !isZero(value) 
  ? { value, from: ACCOUNT }
  : { from: ACCOUNT }
await router[methodName](...args, {
  gasLimit: calculateGasMargin(gasEstimate),
  ...options
})
```