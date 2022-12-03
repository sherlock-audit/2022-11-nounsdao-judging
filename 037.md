cccz

medium

# If the recipient is added to the USDC blacklist, then cancel() does not work

## Summary
cancel() will send the vested USDC to the recipient, if the recipient is added to the USDC blacklist, then cancel() will not work

## Vulnerability Detail
When cancel() is called, it sends the vested USDC to the recipient and cancels future payments.
Consider a scenario where if the payer intends to call cancel() to cancel the payment stream, a malicious recipient can block the address from receiving USDC by adding it to the USDC blacklist (e.g. by doing something malicious with that address, etc.), which prevents the payer from canceling the payment stream and withdrawing future payments 
```solidity
    function cancel() external onlyPayerOrRecipient {
        address payer_ = payer();
        address recipient_ = recipient();
        IERC20 token_ = token();

        uint256 recipientBalance = balanceOf(recipient_);

        // This zeroing is important because without it, it's possible for recipient to obtain additional funds
        // from this contract if anyone (e.g. payer) sends it tokens after cancellation.
        // Thanks to this state update, `balanceOf(recipient_)` will only return zero in future calls.
        remainingBalance = 0;

        if (recipientBalance > 0) token_.safeTransfer(recipient_, recipientBalance);
```
## Impact
A malicious recipient may prevent the payer from canceling the payment stream and withdrawing future payments 
## Code Snippet
https://github.com/sherlock-audit/2022-11-nounsdao/blob/main/src/Stream.sol#L237-L249
## Tool used

Manual Review

## Recommendation
Instead of sending tokens directly to the payer or recipient in cancel(), consider storing the number of tokens in variables and having the payer or recipient claim it later