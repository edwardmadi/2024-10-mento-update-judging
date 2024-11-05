# Issue M-1: User to sell the last supply will make the exchange contribution forever stuck 

Source: https://github.com/sherlock-audit/2024-10-mento-update-judging/issues/17 

## Found by 
0x73696d616f
### Summary

[BancorExchangeProvider::_getScaledAmountOut()](https://github.com/sherlock-audit/2024-10-mento-update/blob/main/mento-core/contracts/goodDollar/BancorExchangeProvider.sol#L345) decreases the amount out when the token in is the supply token by the exit contribution, that is, `scaledAmountOut = (scaledAmountOut * (MAX_WEIGHT - exchange.exitContribution)) / MAX_WEIGHT;`.

Whenever the last supply of the token is withdrawn, it will get all the reserve and store it in `scaledAmountOut`, and then apply the exit contribution, leaving these funds forever stuck, as there is 0 supply to redeem it.

### Root Cause

In `BancorExchangeProvider:345`, the exchange contribution is applied regardless of there being supply left to redeem it.

### Internal pre-conditions

1. All supply must be withdrawn from the exchange.

### External pre-conditions

None.

### Attack Path

1. Users call `Broker::swapIn()`, that calls `GoodDollarExchangeProvider::swapIn()`, which is the exchange contract that holds the token and reserve balances, selling supply tokens until the supply becomes 0.

### Impact

The last exit contribution will be forever stuck. This amount is unbounded and may be very significant. 

### PoC

Add the following test to `BancorExchangeProvider.t.sol`:
```solidity
function test_POC_swapIn_whenTokenInIsToken_shouldSwapIn() public {
  BancorExchangeProvider bancorExchangeProvider = initializeBancorExchangeProvider();
  uint256 amountIn =  300_000 * 1e18;

  bytes32 exchangeId = bancorExchangeProvider.createExchange(poolExchange1);

  vm.startPrank(brokerAddress);
  uint256 amountOut = bancorExchangeProvider.swapIn(exchangeId, address(token), address(reserveToken), amountIn);

  (, , uint256 tokenSupplyAfter, uint256 reserveBalanceAfter, , ) = bancorExchangeProvider.exchanges(exchangeId);

  assertEq(amountOut, 59400e18);
  assertEq(reserveBalanceAfter, 600e18);
  assertEq(tokenSupplyAfter, 0);

  vm.expectRevert("ERR_INVALID_SUPPLY");
  bancorExchangeProvider.swapIn(exchangeId, address(token), address(reserveToken), 1e18);
}
```

### Mitigation

The specific mitigation depends on the design.

# Issue M-2: `TradingLimits::update()` incorrectly only rounds up when `deltaFlowUnits` becomes 0, which will silently increase trading limits 

Source: https://github.com/sherlock-audit/2024-10-mento-update-judging/issues/45 

## Found by 
0x73696d616f
### Summary

[TradingLimits::update()](https://github.com/sherlock-audit/2024-10-mento-update/blob/main/mento-core/contracts/libraries/TradingLimits.sol#L124) divides the traded funds by the decimals of the token, `int256 _deltaFlowUnits = _deltaFlow / int256((10 ** uint256(decimals)));`. In a token with 18 decimals, for example, swapping 1.999...e18 tokens will lead to a `_deltaFlowUnits` of just `1`, taking a major error. This can be exploited to swap up to twice the trading limit, if tokens are swapped 2 by 2 and the state is updated only by 1 each time. Overall, even without malicious intent, the limits will always be silently bypassed due to the rounding.

### Root Cause

In `TradingLimits:135`, it only rounds up whenever `deltaFlowUnits` becomes 0, but the error is just as big if it becomes 1 from 2, effectively not providing enough protection.

### Internal pre-conditions

None.

### External pre-conditions

None.

### Attack Path

1. User calls `Broker::swapIn/Out()` with amounts in and out that produce rounding errors (almost always).

### Impact

The trading limits may be severely bypassed with malicious intent (by double the amount) or by a smaller but still significant amount organically.

### PoC

`TradingLimits::update()` only rounds up when `deltaFlowUnits` becomes 0.
```solidity
function update(
  ITradingLimits.State memory self,
  ITradingLimits.Config memory config,
  int256 _deltaFlow,
  uint8 decimals
) internal view returns (ITradingLimits.State memory) {
  int256 _deltaFlowUnits = _deltaFlow / int256((10 ** uint256(decimals)));
  require(_deltaFlowUnits <= MAX_INT48, "dFlow too large");
  
  int48 deltaFlowUnits = int48(_deltaFlowUnits);
  if (deltaFlowUnits == 0) {
    deltaFlowUnits = _deltaFlow > 0 ? int48(1) : int48(-1);
  }
  ...
```

### Mitigation

The correct fix is:
```solidity
int256 _deltaFlowUnits = (_deltaFlow - 1) / int256((10 ** uint256(decimals))) + 1;
```

# Issue M-3: _getReserveRatioScalar() will give a lesser value than expected 

Source: https://github.com/sherlock-audit/2024-10-mento-update-judging/issues/50 

## Found by 
0x73696d616f, 0xc0ffEE, Ollam, Robert, onthehunt, zarkk01
### Summary

[numberOfExpansions = (block.timestamp - config.lastExpansion) / config.expansionFrequency](https://github.com/sherlock-audit/2024-10-mento-update/blob/main/mento-core/contracts/goodDollar/GoodDollarExpansionController.sol#L232)

The calculation divides it by the expansionFrequency, but this will cause significant rounding issues.

If the expansionFrequency is 1 day (as specified in the docs), time may pass without anybody calling the function and the following scenario will be present.

Let's say 30 hours since last expansion and someone decides then to call it, it will be rounded due to the division to be 1 day, producing a smaller value than the hours that've passed.

### Root Cause

The root cause is the potential of **rounding down** `numberOfExpansions`, which will give a significantly smaller value, depending on how big will be remainder of the division. (6 for 30 hours, 3 for 27 hours, etc)

### Internal pre-conditions

`mintUBIFromExpansion()` need to be callable.



### External pre-conditions

_No response_

### Attack Path

1. Alice calls `mintUBIFromExpansion()` to create an expansion
2. 30 hours pass and nobody calls the function, Bob sees that he can call `mintUBIFromExpansion()`
3. Due to the rounding down of the calculation, it will a value equivalent of 24 hours passing.

### Impact

The protocol will expand **slower than intended**, thus less $G will be minted, which will **become significant** overtime.



### PoC

_No response_

### Mitigation

_No response_

# Issue M-4: Calling `GoodDollarExpansionController::mintUBIFromExpansion()` when the `tokenSupply` is low will mint little $G and still increase the reserve ratio 

Source: https://github.com/sherlock-audit/2024-10-mento-update-judging/issues/70 

## Found by 
0x73696d616f
### Summary

[GoodDollarExpansionController::mintUBIFromExpansion()](https://github.com/sherlock-audit/2024-10-mento-update/blob/main/mento-core/contracts/goodDollar/GoodDollarExpansionController.sol#L170) is called every `config.expansionFrequency` seconds to mint $G. The amount to mint is based on the formula:
`amountToMint = (tokenSupply * reserveRatio - tokenSupply * newRatio) / newRatio`, which scales linearly to `totalSupply`.
As there is no constraint in the amount that is minted, it may be called when the `tokenSupply` is really low and yield no results.
For example, a malicious user may take a flashloan, swap a huge amount of token supply out of the system decreasing `tokenSupply`, call `GoodDollarExpansionController::mintUBIFromExpansion()` and then repay the flashloan, minting a much lower amount of $G.
Even without malicious intent, in times of higher volatility the amount minted may become very low.

In short, due to this issue, it is possible to decrease the reserve ratio significant without expanding the $G supply.

### Root Cause

In `GoodDollarExpansionController:175`, no minimium amounts are enforced. Or the function could be made permissioned by keepers.

### Internal pre-conditions

None.

### External pre-conditions

None.

### Attack Path

The most obvious attack path is the one described in the summary.
1. Malicious user takes a flash loan of $G.
2. Then swaps in the `Broker` the $G for reserve, which decreases the token supply as it burns the $G.
3. Calls `GoodDollarExpansionController::mintUBIFromExpansion()`, which mints a very low amount of token supply.
4. Repays the flash loan.
Alternatively, this flow may happen without flashloans or malicious intent in times of higher volatility.

### Impact

The ability to expand $G is compromised and the reserve ratio will keep decreasing.

### PoC

`GoodDollarExpansionController::mintUBIFromExpansion()` enforces no limits or permission on the caller.
```solidity
function mintUBIFromExpansion(bytes32 exchangeId) external returns (uint256 amountMinted) {
  IBancorExchangeProvider.PoolExchange memory exchange = IBancorExchangeProvider(address(goodDollarExchangeProvider))
    .getPoolExchange(exchangeId);
  ExchangeExpansionConfig memory config = getExpansionConfig(exchangeId);

  bool shouldExpand = block.timestamp > config.lastExpansion + config.expansionFrequency;
  if (shouldExpand || config.lastExpansion == 0) {
    uint256 reserveRatioScalar = _getReserveRatioScalar(config);

    exchangeExpansionConfigs[exchangeId].lastExpansion = uint32(block.timestamp);
    amountMinted = goodDollarExchangeProvider.mintFromExpansion(exchangeId, reserveRatioScalar);

    IGoodDollar(exchange.tokenAddress).mint(address(distributionHelper), amountMinted);
    distributionHelper.onDistribution(amountMinted);

    // Ignored, because contracts only interacts with trusted contracts and tokens
    // slither-disable-next-line reentrancy-events
    emit ExpansionUBIMinted(exchangeId, amountMinted);
  }
}
```
`GoodDollarExchangeProvider::mintFromExpansion()` formula scales pro-rata to `tokenSupply`:
```solidity
   * @dev Calculates the amount of G$ tokens that need to be minted as a result of the expansion
   *      while keeping the current price the same.
   *      calculation: amountToMint = (tokenSupply * reserveRatio - tokenSupply * newRatio) / newRatio
   */
  function mintFromExpansion(
  ...
```

`BancorExchangeProvider::executeSwap()` decreases the total supply when `tokenIn != exchange.reserveAsset`. Or, in other words, by transferring in $G it decreases `exchange.tokenSupply`:
```solidity
function executeSwap(bytes32 exchangeId, address tokenIn, uint256 scaledAmountIn, uint256 scaledAmountOut) internal {
  PoolExchange memory exchange = getPoolExchange(exchangeId);
  if (tokenIn == exchange.reserveAsset) {
    exchange.reserveBalance += scaledAmountIn;
    exchange.tokenSupply += scaledAmountOut;
  } else {
    require(exchange.reserveBalance >= scaledAmountOut, "Insufficient reserve balance for swap");
    exchange.reserveBalance -= scaledAmountOut;
    exchange.tokenSupply -= scaledAmountIn;
  }
  exchanges[exchangeId].reserveBalance = exchange.reserveBalance;
  exchanges[exchangeId].tokenSupply = exchange.tokenSupply;
}
```

### Mitigation

Specify minimum amounts of $G to mint for every expansion or make the function permissioned.

