Koolex

high

# The stream recipient can prevent the payer from cancelling the stream

## Summary
The stream recipient can prevent the payer from cancelling the stream.

## Vulnerability Detail

For stream tokens that implement IERC777 (backward compatible with IERC-20), the recipient can prevent the payer from cancelling the stream. The receipent could be a contract which implements `tokensReceived()` hook which is called when transferring the tokens at the following line in `cancel()` function
```sh
 if (recipientBalance > 0) token_.safeTransfer(recipient_, recipientBalance);
```

The hook `tokensReceived()` can simply revert the transaction if the received amount is less than full fund. This way, the `cancel()` function will always revert till the stream ends.

Another scenario, the receipent can block `cancel()` function always (even after the stream ends) by reverting the transaction if the received amount doesn't equal tokenAmount-1. Then the receipent waits till the stream ends and use `withdraw()` function to withdraw tokenAmount-1. In this scenario, the payer can not withdraw what's left from the contract balance after the stream ends.

Note: **tokenAmount** refers to the amount that was set when the stream was created by the payer.

## Impact

- The recipient guarantees receiving all the fund sent to the stream contract by the payer (of course up to the tokenAmount set when the stream was created).
- The payer can not withdraw what's left from the contract balance after the stream ends


## Code Snippet

`cancel()` function

https://github.com/sherlock-audit/2022-11-nounsdao/blob/main/src/Stream.sol#L237-L259

## Tool used

Manual Review

## Recommendation

Favor pull over push. The `cancel()` function used by two parties. When the function is called, it pulls for one and push for the other. this causes the vulnerability described above.

I suggest: 

1. Let the `cancel()` function transfer the token to the payer only.
An example:
```sh
uint256 recipientBalance = balanceOf(recipient_);
// set remainingBalance to recipientBalance (as it is the maximum at this time)
remainingBalance = recipientBalance;
// payerBalance after deducting recipientBalance
uint256 payerBalance = tokenBalance()-recipientBalance;
if (payerBalance > 0) {
            token_.safeTransfer(payer_, payerBalance);
}
```

2. The recipient can still withdraw their fair share of the funds by calling `withdraw()` and passing their balance.