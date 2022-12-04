WATCHPUG

medium

# Lack of sanity check for `stoptime`

## Summary

We believe when the stoptime is before the current time, then it might be wrong.

## Vulnerability Detail

Creating a stream with a `stopTime` earlier than the current time is the equivalent of making a simple ERC20 transfer, except that a stream will be much more expensive in terms of gas cost.

Therefore, we believe there is no such use case and the system should include a sanity check to ensure `stoptime > block.timestamp`.

## Impact

If the user calls `createAndFundStream()` with a wrong/unintended `stoptime` that is earlier than the current time, the stream will not be able to be canceled.

## Code Snippet

https://github.com/sherlock-audit/2022-11-nounsdao/blob/main/src/StreamFactory.sol#L184-L213

## Tool used

Manual Review

## Recommendation

```diff
function createStream(
        address payer,
        address recipient,
        uint256 tokenAmount,
        address tokenAddress,
        uint256 startTime,
        uint256 stopTime,
        uint8 nonce
    ) public returns (address stream) {
        // These input checks are here rather than in Stream because these parameters are written
        // using clone-with-immutable-args, meaning they are already set when Stream is created and can't be
        // verified there. The main benefit of this approach is significant gas savings.
        if (payer == address(0)) revert PayerIsAddressZero();
        if (recipient == address(0)) revert RecipientIsAddressZero();
        if (tokenAmount == 0) revert TokenAmountIsZero();
        if (stopTime <= startTime) revert DurationMustBePositive();
        if (tokenAmount < stopTime - startTime) revert TokenAmountLessThanDuration();
+       if (stopTime <= block.timestamp) revert StopTimeLessThanCurrentTime();

        stream = streamImplementation.cloneDeterministic(
            encodeData(payer, recipient, tokenAmount, tokenAddress, startTime, stopTime),
            salt(
                msg.sender, payer, recipient, tokenAmount, tokenAddress, startTime, stopTime, nonce
            )
        );
        IStream(stream).initialize();

        emit StreamCreated(
            msg.sender, payer, recipient, tokenAmount, tokenAddress, startTime, stopTime, stream
            );
    }
```