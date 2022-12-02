obront

medium

# Payer cannot withdraw accidental extra funds sent to the contract without canceling

## Summary

If a different ERC20 is accidentally sent to the contract, the Payer can withdraw it using the `rescueERC20` function. However, if they accidentally send extra of the streaming token's funds to the contract, the only way to withdraw it is to cancel the stream.

## Vulnerability Detail

The Nouns team seems to have made the decision that they should protect against accidental funds being sent into the contract. They implemented the `rescueERC20` function to accomplish this.

However, the `rescueERC20` function only works for non-stream tokens. If they accidentally send too much of the streaming token (which seems like a likely scenario), there is no similar rescue function to retrieve it. 

Instead, their only option is to cancel the stream. In a protocol that's intended to be run via a governance system, canceling the stream could cause problems for the receiver (for example, if they are unable to pass a vote to restart the stream).

## Impact

If too many stream tokens are sent into the contract, the whole stream will need to be canceled to retrieve them.

## Code Snippet

https://github.com/sherlock-audit/2022-11-nounsdao/blob/main/src/Stream.sol#L237-L259

## Tool used

Manual Review

## Recommendation

Adjust the rescueERC20 function to also allow for withdrawing excess stream tokens, as follows:

```diff
function rescueERC20(address tokenAddress, uint256 amount) external onlyPayer {
-    if (tokenAddress == address(token())) revert CannotRescueStreamToken();
+    if (tokenAddress == address(token()) && amount < tokenBalance() - remainingBalance) revert AmountExceedsBalance;

    IERC20(tokenAddress).safeTransfer(msg.sender, amount);
}
```