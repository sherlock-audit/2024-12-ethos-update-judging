# Issue M-1: Incorrect rounding in the `ReputationMarket._calcCost()` function. 

Source: https://github.com/sherlock-audit/2024-12-ethos-update-judging/issues/66 

## Found by 
0xaxaxa, Al-Qa-qa, X12, bughuntoor, moray5554, tnquanghuy0512, whitehair0330
### Summary

When buying and selling, the cost is calculated using rounding based on `isPositive`. However, the rounding should be based on `isBuy`, not `isPositive`.

### Root Cause

The [_calcCost()](https://github.com/sherlock-audit/2024-12-ethos-update/blob/main/ethos/packages/contracts/contracts/ReputationMarket.sol#L1057) function calculates `cost` by rounding based on `isPositive`.

If `isPositive` is `false` (indicating that `DISTRUST` votes are being traded), the calculation is rounded up.

Consider the following scenario:

1. A user buys 2 `DISTRUST` votes.
2. The user sells a `DISTRUST` vote.
3. The user sells another `DISTRUST` vote.

During the buying process, rounding up occurs once, but when selling, rounding up occurs twice—at steps 2 and 3. As a result, `marketFunds` will be decremented by a dust amount.

If `marketFunds` was originally 0 (with the market created by the admin at a 0 creation cost), then step 3 becomes impossible.

In fact, `isPositive` is never related to the rounding direction.

```solidity
      function _calcCost(
        Market memory market,
        bool isPositive,
        bool isBuy,
        uint256 amount
      ) private pure returns (uint256 cost) {
        ...

        int256 costRatio = LMSR.getCost(
          market.votes[TRUST],
          market.votes[DISTRUST],
          voteDelta[0],
          voteDelta[1],
          market.liquidityParameter
        );

        uint256 positiveCostRatio = costRatio > 0 ? uint256(costRatio) : uint256(costRatio * -1);
        // multiply cost ratio by base price to get cost; divide by 1e18 to apply ratio
        cost = positiveCostRatio.mulDiv(
          market.basePrice,
          1e18,
1057      isPositive ? Math.Rounding.Floor : Math.Rounding.Ceil
        );
      }
```

### Internal pre-conditions

### External pre-conditions

### Attack Path

### Impact

The last vote might not be sold.

### PoC

### Mitigation

Use `!isBuy` instead of `isPositive`.

```diff
        cost = positiveCostRatio.mulDiv(
          market.basePrice,
          1e18,
-         isPositive ? Math.Rounding.Floor : Math.Rounding.Ceil
+         !isBuy ? Math.Rounding.Floor : Math.Rounding.Ceil
        );
```

