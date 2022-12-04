DecorativePineapple

medium

# Two address tokens can be withdrawn by the payer even when the stream has began

## Summary
It has been identified that if a stream has begun with a two address token the payer can withdraw the full amount via the [`rescueERC20`](https://github.com/sherlock-audit/2022-11-nounsdao/blob/main/src/Stream.sol#L268-L272) function.

## Vulnerability Detail
Two address tokens exists in the blockchain. For example, Synthetix's ProxyERC20 contract is such a token which exists in many forms (sUSD, sBTC...). A stream can be created with such tokens, but the payer can withdraw the full amount via the [`rescueERC20`](https://github.com/sherlock-audit/2022-11-nounsdao/blob/main/src/Stream.sol#L268-L272) function. The only check in the [`rescueERC20`](https://github.com/sherlock-audit/2022-11-nounsdao/blob/main/src/Stream.sol#L268-L272) function is that `tokenAddress == address(token()`, which is irrelevant in the case of two address tokens.

## Impact
This can make the payer be able to withdraw the funds that are deposited in the stream and break the system, because the balance of the stream contract is zero and the `recipient()` cannot withdraw the fair amount based on the rate and the time elapsed.

## Code Snippet
The payer can withdraw the full amount of token via the [`rescueERC20`](https://github.com/sherlock-audit/2022-11-nounsdao/blob/main/src/Stream.sol#L268-L272) function:
```solidity
    function rescueERC20(address tokenAddress, uint256 amount) external onlyPayer {
        if (tokenAddress == address(token())) revert CannotRescueStreamToken();

        IERC20(tokenAddress).safeTransfer(msg.sender, amount);
    }
```

## Tool used
Manual Review

## Recommendation
Replace the address check with a balance check - record the balance of the token that's deposited in the stream before and after the transfer and assert that they are equal.