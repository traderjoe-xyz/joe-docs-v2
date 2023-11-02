---
sidebar_position: 0
sidebar_label: Swap Tokens
---

# Swap Tokens

## Introduction

Token swaps can be executed through the `LBRrouter` contract. This contract will abstract some of the complexity of the swap, perform safety checks and will revert if certain conditions were to not be met. The `LBRouter` in V2.1 is able to perform swaps through `LBPair` V2.1, V2.0 and pair contracts from V1.

The rest of the document describes:

- Which functions should be called to swap tokens
- How to use them in code examples

## Swap Functions

Swap functions can be broadly divided into two categories:

When the input amount is specified:

- `swapExactTokensForTokens`
- `swapExactTokensForNATIVE`
- `swapExactNATIVEForTokens`

When the desired output amount is specified:

- `swapTokensForExactTokens`
- `swapTokensForExactNATIVE`
- `swapNATIVEForExactTokens`

### When Input Amount Is Specified

- `swapExactTokensForTokens` - When you specify an exact amount of ERC-20 to swap for another ERC-20. E.g. USDC/USDT and you input USDC; router will then fetch how much USDT to expect as output and perform swap.
- `swapExactTokensForNATIVE` - When you specify an exact amount of ERC-20 to swap for NATIVE tokens. E.g AVAX/USDC and you input USDC; router will then fetch how much AVAX to expect as output and perform swap.
- `swapExactNATIVEForTokens` - When you specify an exact amount of NATIVE tokens to swap for an ERC-20. E.g AVAX/USDC and you input AVAX; router will then fetch how much USDC to expect as output and perform swap.

### When Output Amount Is Specified

- `swapTokensForExactTokens` - When you specify exact amount of ERC-20 that you want to receive. E.g. USDC/USDT and you input USDC; router will then fetch how much USDT to transfer from your wallet and perform swap.
- `swapTokensForExactNATIVE` - When you specify exact amount of NATIVE tokens that you want to receive. E.g. AVAX/USDT and you input AVAX; router will then fetch how much USDT to transfer from your wallet and perform swap.
- `swapNATIVEForExactTokens` - When you specify exact amount of ERC-20 that you want to receive. E.g. AVAX/USDT and you input USDT; router will then fetch how much AVAX to transfer from your wallet and perform swap.

## Code Examples

#### 1. Swap 10 USDC for USDT using `swapExactTokensForTokens` with no intermediate swap paths:

```js
uint128 amountIn = 10e6;

USDC.approve(address(router), amountIn);

IERC20[] memory tokenPath = new IERC20[](2);
tokenPath[0] = USDC;
tokenPath[1] = USDT;

uint256[] memory pairBinSteps = new uint256[](1); // pairBinSteps[i] refers to the bin step for the market (x, y) where tokenPath[i] = x and tokenPath[i+1] = y
pairBinSteps[0] = 1;

ILBRouter.Version[] memory versions = new ILBRouter.Version[](1);
versions[0] = ILBRouter.Version.V2_1; // add the version of the Dex to perform the swap on

ILBRouter.Path memory path; // instanciate and populate the path to perform the swap.
path.pairBinSteps = pairBinSteps;
path.versions = versions;
path.tokenPath = tokenPath;

(, uint128 amountOut, ) = router.getSwapOut(pair, amountIn, true);
uint256 amountOutWithSlippage = amountOut * 99 / 100; // We allow for 1% slippage
uint256 amountOutReal = router.swapExactTokensForTokens(amountIn, amountOutWithSlippage, path, to, block.timestamp + 1);
```

#### 2. Swap 1 AVAX for USDT using `swapExactNATIVEForTokens` with no intermediate swap paths:

```js
uint256 amountIn = 1e18;

IERC20[] memory tokenPath = new IERC20[](2);
tokenPath[0] = WAVAX;
tokenPath[1] = USDT;

uint256[] memory pairBinSteps = new uint256[](1);
pairBinSteps[0] = 15;

ILBRouter.Version[] memory versions = new ILBRouter.Version[](1);
versions[0] = ILBRouter.Version.V2_1; // add the version of the Dex to perform the swap on

ILBRouter.Path memory path; // instanciate and populate the path to perform the swap.
path.pairBinSteps = pairBinSteps;
path.versions = versions;
path.tokenPath = tokenPath;

(, uint256 amountOut, ) = router.getSwapOut(pairWavax, amountIn, false);
uint256 amountOutWithSlippage = amountOut * 99 / 100; // We allow for 1% slippage
uint256 amountOutReal = router.swapExactNATIVEForTokens{value: amountIn}(amountOutWithSlippage, path, to, block.timestamp + 1);
```

#### 3. Swap USDT to get 10 USDC output using `swapTokensForExactTokens` that routes through WAVAX. In this example, the first swap occurs through USDT/WAVAX V2.1 pool and the second swap occurs through WAVAX/USDC V1 pool:

```js
uint256 amountOut = 10e6;

USDT.approve(address(router), 11e6);

IERC20[] memory tokenPath = new IERC20[](3);
tokenPath[0] = USDT;
tokenPath[1] = WAVAX;
tokenPath[2] = USDC;

pairBinSteps = new uint256[](2);
pairBinSteps[0] = 15;
pairBinSteps[1] = 0; // Bin step of 0 points to the Joe V1 pair

ILBRouter.Version[] memory versions = new ILBRouter.Version[](2);
versions[0] = ILBRouter.Version.V2_1;
versions[1] = ILBRouter.Version.V1;

ILBRouter.Path memory path;
path.pairBinSteps = pairBinSteps;
path.versions = versions;
path.tokenPath = tokenPath;

// We define amountInMax as an arbitrary amount of 11e6 here
uint256[] memory amountsIn = router.swapTokensForExactTokens(amountOut, 11e6, path, to, block.timestamp + 1);
```
