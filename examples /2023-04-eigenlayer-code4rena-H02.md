# slashQueuedWithdrawal() is unable to skip a malicious strategy, while it should.

## Lines of code
https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L536-L579

## Vulnerability details
The `slashQueuedWithdrawal()` function enables the owner to slash withdrawals that have already been queued by a 'frozen' operator (or a staker delegated to one). This function includes a parameter called 'indicesToSkip', which should contain the indexes of strategies that need to be skipped during slashing. Developers decided it necessary to implement this functionality due to the possibility of encountering malicious strategies in queued withdrawals:
```solidity
/**
     * @notice Slashes an existing queued withdrawal that was created by a 'frozen' operator (or a staker delegated to one)
     * @param recipient The funds in the slashed withdrawal are withdrawn as tokens to this address.
     * @param queuedWithdrawal The previously queued withdrawal to be slashed
     * @param tokens Array in which the i-th entry specifies the `token` input to the 'withdraw' function of the i-th Strategy in the `strategies`
     * array of the `queuedWithdrawal`.
     * @param indicesToSkip Optional input parameter -- indices in the `strategies` array to skip (i.e. not call the 'withdraw' function on). This input exists
     * so that, e.g., if the slashed QueuedWithdrawal contains a malicious strategy in the `strategies` array which always reverts on calls to its 'withdraw' function,
     * then the malicious strategy can be skipped (with the shares in effect "burned"), while the non-malicious strategies are still called as normal.
     */
    function slashQueuedWithdrawal(address recipient, QueuedWithdrawal calldata queuedWithdrawal, IERC20[] calldata tokens, uint256[] calldata indicesToSkip)
```
However, upon examination of the function itself, it becomes apparent that the loop counter is not properly handled in situations where the loop encounters an index that should be skipped:
```solidity
File: StrategyManager.sol
560:         for (uint256 i = 0; i < strategiesLength;) {
561:             // check if the index i matches one of the indices specified in the `indicesToSkip` array
562:             if (indicesToSkipIndex < indicesToSkip.length && indicesToSkip[indicesToSkipIndex] == i) { 
563:                 unchecked {
564:                     ++indicesToSkipIndex;
565:                 }
566:             } else {
567:                 if (queuedWithdrawal.strategies[i] == beaconChainETHStrategy){
568:                      //withdraw the beaconChainETH to the recipient
569:                     _withdrawBeaconChainETH(queuedWithdrawal.depositor, recipient, queuedWithdrawal.shares[i]);
570:                 } else {
571:                     // tell the strategy to send the appropriate amount of funds to the recipient
572:                     queuedWithdrawal.strategies[i].withdraw(recipient, tokens[i], queuedWithdrawal.shares[i]);
573:                 }
574:                 unchecked {
575:                     ++i;
576:                 }
577:             }
578:         }
```
Each time the loop counter 'i' reaches the same value as the index that needs to be skipped, the index counter value increases while the loop counter value remains the same. This means that the same index will be used again in the next iteration, potentially allowing a malicious strategy to block slashing.

## Impact
The functionality designed to enable protocol owners to skip malicious strategies during slashing is not functioning properly.

## Proof of Concept
The addition of the following test to `src/test/unit/StrategyManagerUnit.t.sol` could demonstrate a scenario where the owner attempts to skip one of the strategies in a queued withdrawal, but the skipping functionality fails to work:
```solidity
    function testIndicesToSkipNotWorking() external {
        address recipient = address(333);
        uint256 withdrawalAmount = 1e18;

        IStrategyManager.QueuedWithdrawal memory queuedWithdrawal =
            QueueWithdrawal_ToSelf_MultipleStrategies(withdrawalAmount);

        uint256 balanceBefore = dummyToken.balanceOf(address(recipient));

        // slash the delegatedOperator
        slasherMock.freezeOperator(queuedWithdrawal.delegatedAddress);

        cheats.startPrank(strategyManager.owner());
        uint256[] memory indicesToSkip = new uint256[](1);
        indicesToSkip[0] = 0;
        IERC20[] memory tokenArray = new IERC20[](2);
        tokenArray[0] = dummyToken;
        tokenArray[1] = dummyToken;
        strategyManager.slashQueuedWithdrawal(recipient, queuedWithdrawal, tokenArray, indicesToSkip);
        cheats.stopPrank();

        uint256 balanceAfter = dummyToken.balanceOf(address(recipient));
        // balance should change only for half of withdrawalAmount, since one of strategies should be skipped
        require(balanceAfter == balanceBefore + withdrawalAmount/2, "balanceAfter != balanceBefore + withdrawalAmount / 2");
    }

    function QueueWithdrawal_ToSelf_MultipleStrategies(uint256 withdrawalAmount) public
        returns (IStrategyManager.QueuedWithdrawal memory /* queuedWithdrawal */)
    {
        StrategyWrapper maliciousStrat = new StrategyWrapper(strategyManager, dummyToken);
        // adding second strategy to the protocol 
        cheats.startPrank(strategyManager.owner());
        IStrategy[] memory _strategy = new IStrategy[](1);
        _strategy[0] = maliciousStrat;
        strategyManager.addStrategiesToDepositWhitelist(_strategy);
        cheats.stopPrank();
        // depositing to both strategies half of withdrawalAmount
        IStrategy[] memory strategies = new IStrategy[](2);
        strategies[0] = dummyStrat;
        strategies[1] = maliciousStrat;

        uint256 shares1 = strategyManager.depositIntoStrategy(strategies[0], dummyToken, withdrawalAmount/2);
        uint256 shares2 = strategyManager.depositIntoStrategy(strategies[1], dummyToken, withdrawalAmount/2);

        uint256[] memory shareAmounts = new uint256[](2);
        shareAmounts[0] = shares1;
        shareAmounts[1] = shares2;
        // creating Queued Withdrawal with both strategies
        (IStrategyManager.QueuedWithdrawal memory queuedWithdrawal) =
            _setUpQueuedWithdrawalStructMultipleStrategies(/*staker*/ address(this), /*withdrawer*/ address(this), strategies, shareAmounts);

        {
            uint256[] memory strategyIndexes = new uint256[](2);
            strategyIndexes[0] = 1;
            strategyIndexes[1] = 0;
            strategyManager.queueWithdrawal(
                strategyIndexes,
                queuedWithdrawal.strategies,
                queuedWithdrawal.shares,
                /*withdrawer*/ address(this),
                false
            );
        }
        return queuedWithdrawal;
    }

    function _setUpQueuedWithdrawalStructMultipleStrategies(address staker, address withdrawer, IStrategy[] memory strategies, uint256[] memory shareAmounts)
        internal view returns (IStrategyManager.QueuedWithdrawal memory queuedWithdrawal)
    {
        IStrategyManager.WithdrawerAndNonce memory withdrawerAndNonce = IStrategyManager.WithdrawerAndNonce({
            withdrawer: withdrawer,
            nonce: uint96(strategyManager.numWithdrawalsQueued(staker))
        });
        queuedWithdrawal = 
            IStrategyManager.QueuedWithdrawal({
                strategies: strategies,
                shares: shareAmounts,
                depositor: staker,
                withdrawerAndNonce: withdrawerAndNonce,
                withdrawalStartBlock: uint32(block.number),
                delegatedAddress: strategyManager.delegation().delegatedTo(staker)
            }
        );
        return (queuedWithdrawal);
    }
```
## Recommended Mitigation Steps
Consider updating the slashQueuedWithdrawal() function to increment the loop counter 'i' with each iteration.

## Link to origin issue
https://github.com/code-423n4/2023-04-eigenlayer-findings/issues/186