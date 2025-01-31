# Issue M-4: Precision loss in boostPrice in V3AMO
Source: https://github.com/sherlock-audit/2024-10-axion-judging/issues/191

Found by
0x37, Bigsam, durov, s1ce, yuza101

## Summary
There is precision loss in a specific case in SolidlyV3AMO, which leads to incorrect value of boostPrice being reported. This leads to issues with upstream functions that use it.

## Root Cause
https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/SolidlyV3AMO.sol#L343

Consider the following code:
```
function boostPrice() public view override returns (uint256 price) {  
        (uint160 _sqrtPriceX96, , , ) = ISolidlyV3Pool(pool).slot0();  
        uint256 sqrtPriceX96 = uint256(_sqrtPriceX96);  
        if (boost < usd) {  
            price = (10 ** (boostDecimals - usdDecimals + PRICE_DECIMALS) * sqrtPriceX96 ** 2) / Q96 ** 2;  
        } else {  
            if (sqrtPriceX96 >= Q96) {  
                // @audit: massive precision loss here  
                price = 10 ** (boostDecimals - usdDecimals + PRICE_DECIMALS) / (sqrtPriceX96 ** 2 / Q96 ** 2);  
            } else {  
                price = (10 ** (boostDecimals - usdDecimals + PRICE_DECIMALS) * Q96 ** 2) / sqrtPriceX96 ** 2;  
            }  
        }  
    }
```
Notice that in this specific clause:
```
if (sqrtPriceX96 >= Q96) {  
                // @audit: massive precision loss here  
                price = 10 ** (boostDecimals - usdDecimals + PRICE_DECIMALS) / (sqrtPriceX96 ** 2 / Q96 ** 2);  
            }
```
We divide (sqrtPriceX96 ** 2 / Q96 ** 2). However, consider a case where the value of the stablecoin is 20% higher than the value of boost; in this case a price of around 0.8 should be reported by the function but because (sqrtPriceX96 ** 2 / Q96 ** 2) will round down to 1, the function will end up reporting a price of 1.

## Internal pre-conditions
*No response*

## External pre-conditions
We need the price of the stablecoin to be at least a bit higher than the price of BOOST for this to be relevant

## Attack Path
Described above, precision loss leads to boostPrice calculation reported by SolidlyV3AMO being incorrect.

## Impact
Impact is that, because the price is reported incorrect, public calls to unfarmBuyBurn() will fail because the following check will fail:
```
if (newBoostPrice > boostUpperPriceBuy) revert PriceNotInRange(newBoostPrice);
```
newBoostPrice will be reported as 1 even though it should be much lower.

## PoC
First, set sqrtPriceX96 to 87150978765690771352898345369 (10% above 2^96)
```
pool = await ethers.getContractAt("ISolidlyV3Pool", poolAddress);  
await pool.initialize(sqrtPriceX96);  
Then:

if("Showcases the incorrect price that is returned", async function() {  
        console.log(await solidlyV3AMO.boostPrice()) // 1000000  
})
```

## Mitigation
No response

Discussion
sherlock-admin2

The protocol team fixed this issue in the following PRs/commits:
https://github.com/AXION-MONEY/liquidity-amo/pull/8
