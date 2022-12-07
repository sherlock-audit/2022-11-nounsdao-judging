neumo

medium

# Overflow can make ratePerSecond zero and prevent recipient from withdrawing for the entire duration of the stream

## Summary
If the amount passed in when creating a stream is sufficiently large, the calculation of `ratePerSecond` can be very small, even 0, making withdrawals impossible for the recipient the whole duration of the stream.

## Vulnerability Detail
The following lines inside function `_recipientBalance()` calculate the balance of the recipient without taking into account the withdrawals made:
https://github.com/sherlock-audit/2022-11-nounsdao/blob/main/src/Stream.sol#L333-L345
We can see that before or at `startTime`, balance is zero, and after or at `stopTime` balance is  `tokenAmount`. But between the two timestamps, the balance is calculated using `ratePerSecond()`.
https://github.com/sherlock-audit/2022-11-nounsdao/blob/main/src/Stream.sol#L125-L138
If we create the stream with a sufficiently large value of `tokenAmount`, we can make the multiplication in line 136 overflow, obtaining a very small value of `ratePerSecond`. For example, if we get to obtain a `ratePerSecond` of zero, the withdrawable balance of the `recipient` would be 0 while the stream is active. Only after `stopTime` the `recipient` could withdraw funds.

## Impact
Medium, no loss of user funds, but contract would be mainly useless.

## Code Snippet
I created a function inside file [Stream.t.sol](https://github.com/sherlock-audit/2022-11-nounsdao/blob/main/test/Stream.t.sol) with a series of logs illustrating what would be the values of the `tokenAmount`, `ratePerSecond` and the balance of the recipient when `block.timestamp` is in the range `(startTime, stopTime)`.
```solidity
function testIssue_overflowInBalanceCalculation() public {

	unchecked{
		uint duration = 31536000; // 1 year
		uint tokenAmount = (type(uint).max/(1e6))+10;

		// ratePerSecond = RATE_DECIMALS_MULTIPLIER * tokenAmount() / duration;
		uint rps = 1e6*tokenAmount/duration;
		console2.log(tokenAmount);
		console2.log(rps);

		// balance = (elapsedTime_ * ratePerSecond()) / RATE_DECIMALS_MULTIPLIER;
		uint balance = (duration)*rps/1e6;
		console2.log(balance);
	}
	// logs
	// token amount: 115792089237316195423570985008687907853269984665640564039457584007913139
	// rate per second: 0
	// balance when block.timestamp > startTime and < stopTime: 0

}
```

## Tool used

Manual Review and forge tests

## Recommendation
Don't use an unchecked block for the calculation of `ratePerSecond` or at least check that it is not greater than `type(uint).max/1e6`.