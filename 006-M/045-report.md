zimu

medium

# It doesn't handle fee-on-transfer/deflationary tokens

## Summary
Although the protocol currently doesn't have fee-on-transfer or rabasing tokens in on-chain context, since `StreamFactory.createStream(...)` is proposed by users, it is not guaranteed fee-to-transfer/deflationary tokens won't be used to transfer in stream from payers to recipients in the future.

## Vulnerability Detail
1. The streaming solution consists of Streamer and Token Buyer. After `StreamFactory.createStream(...)` creates a proposal with Streamer, the user as the Payer can submit a call of `Payer.sendOrRegisterDebt(...)` and register debt. https://github.com/nounsDAO/token-buyer/blob/main/src/Payer.sol#L100-L118
```solidity
    function sendOrRegisterDebt(address account, uint256 amount) {
        uint256 availableBalance = paymentToken.balanceOf(address(this));
        if (amount <= availableBalance) {
            paymentToken.safeTransfer(account, amount);
        } else if (availableBalance > 0) {
            paymentToken.safeTransfer(account, availableBalance);
            registerDebt(account, amount - availableBalance);
        } else {
            registerDebt(account, amount);
        }
    }
```

2. Then the Payer can call `Payer.payBackDebt(...)` to fund the stream asynchronously. As can be seen in the following, `Payer.payBackDebt(...)` doesn't check the balance after the execution of `paymentToken.safeTransfer(_debtAccount, amount)` or `paymentToken.safeTransfer(_debtAccount, _debtAmount)` and the debt amount is calculated by dequeuing from the registered debt queue before the transfer. When in fee-on-transfer/deflationary tokens case, the Stream contract will receive the amount of tokens less than the amount declared in the proposal of `StreamFactory.createStream(...)`.  https://github.com/nounsDAO/token-buyer/blob/main/src/Payer.sol#L139-L191
```solidity
    function payBackDebt(uint256 amount) {
        uint256 debtPaidBack = 0;
        while (amount > 0 && !queue.empty()) {
            DebtQueue.DebtEntry storage debt = queue.front();
            uint96 _debtAmount = debt.amount;
            address _debtAccount = debt.account;
            if (amount < _debtAmount) {
                uint96 remainingDebt = _debtAmount - uint96(amount);
                debt.amount = remainingDebt;
                debtPaidBack += amount;
                paymentToken.safeTransfer(_debtAccount, amount);
                emit PaidBackDebt(_debtAccount, amount, remainingDebt);
                break;
            } else {
                amount -= _debtAmount;
                debtPaidBack += _debtAmount;
                queue.popFront();
                paymentToken.safeTransfer(_debtAccount, _debtAmount);
                emit PaidBackDebt(_debtAccount, _debtAmount, 0);
            }
        }
        if (debtPaidBack > 0) {
            totalDebt -= debtPaidBack;
        }
    }
```

3. After the stop time, the recipient would finally withdraw and receive less amount of tokens than the amount declared in the proposal. https://github.com/sherlock-audit/2022-11-nounsdao/blob/main/src/Stream.sol#L213-L228
```solidity
    function withdraw(uint256 amount) {
        if (amount == 0) revert CantWithdrawZero();
        address recipient_ = recipient();
        uint256 balance = balanceOf(recipient_);
        if (balance < amount) revert AmountExceedsBalance();
        unchecked {
            remainingBalance = remainingBalance - amount;
        }
        token().safeTransfer(recipient_, amount);
        emit TokensWithdrawn(msg.sender, recipient_, amount);
    }
```

## Impact
The recipient will receive less amount of tokens than the amount declared in the proposal.

## Code Snippet
https://github.com/nounsDAO/token-buyer/blob/main/src/Payer.sol#L100-L118
https://github.com/nounsDAO/token-buyer/blob/main/src/Payer.sol#L139-L191
https://github.com/sherlock-audit/2022-11-nounsdao/blob/main/src/Stream.sol#L213-L228

## Tool used
Manual Review

## Recommendation
1. In Token Buyer, check the balance after the execution of `paymentToken.safeTransfer(_debtAccount, amount)` or `paymentToken.safeTransfer(_debtAccount, _debtAmount)` in `Payer.payBackDebt(...)`;
2. In Streamer, change from
```solidity
    function withdraw(uint256 amount) external onlyPayerOrRecipient {
        if (amount == 0) revert CantWithdrawZero();
        address recipient_ = recipient();
        uint256 balance = balanceOf(recipient_);
        if (balance < amount) revert AmountExceedsBalance();
        unchecked {
            remainingBalance = remainingBalance - amount;
        }
        token().safeTransfer(recipient_, amount);
        emit TokensWithdrawn(msg.sender, recipient_, amount);
    }
```
to 
```solidity
    function withdraw(uint256 amount) external onlyPayerOrRecipient {
        if (amount == 0) revert CantWithdrawZero();
        address recipient_ = recipient();
        uint256 balance = balanceOf(recipient_);
        if (balance < amount) revert AmountExceedsBalance();
        balanceBefore_ = token().balanceOf(address(recipient_));
        token().safeTransfer(recipient_, amount);
        balanceAfter_ = token().balanceOf(address(recipient_));
        amountSub_ = balanceAfter_ - balanceBefore_;
        unchecked {
            remainingBalance = remainingBalance - amountSub_;
        }
        emit TokensWithdrawn(msg.sender, recipient_, amountSub_);
    }
```