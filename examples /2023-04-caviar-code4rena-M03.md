# Wrong flashFee calculation

## Lines of code
https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L750

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L731-L738

## Vulnerability details
`PrivatePool.sol` functionality includes a few types of fees. One of them is `changeFee`, which is charged to users in two cases:

During exchange users' NFTs to NFTs deposited in the pool using `change()` function:
```solidity
File: PrivatePool.sol
385:     function change(
386:         uint256[] memory inputTokenIds,
387:         uint256[] memory inputTokenWeights,
388:         MerkleMultiProof memory inputProof,
389:         IStolenNftOracle.Message[] memory stolenNftProofs,
390:         uint256[] memory outputTokenIds,
391:         uint256[] memory outputTokenWeights,
392:         MerkleMultiProof memory outputProof
393:     ) public payable returns (uint256 feeAmount, uint256 protocolFeeAmount) {
...
415:             // calculate the fee amount
416:             (feeAmount, protocolFeeAmount) = changeFeeQuote(inputWeightSum);
...
421:         if (baseToken != address(0)) {
422:             // transfer the fee amount of base tokens from the caller
423:             ERC20(baseToken).safeTransferFrom(msg.sender, address(this), feeAmount);
...
427:         } else {
428:             // check that the caller sent enough ETH to cover the fee amount and protocol fee
429:             if (msg.value < feeAmount + protocolFeeAmount) revert InvalidEthAmount();
...
434:             // refund any excess ETH to the caller
435:             if (msg.value > feeAmount + protocolFeeAmount) {
436:                 msg.sender.safeTransferETH(msg.value - feeAmount - protocolFeeAmount);
437:             }
438:         }
```
During the execution of flashLoan() calls:
```solidity
File: PrivatePool.sol
623:     function flashLoan(IERC3156FlashBorrower receiver, address token, uint256 tokenId, bytes calldata data) 
624:         external
625:         payable
626:         returns (bool)
627:     {
...
631:         // calculate the fee
632:         uint256 fee = flashFee(token, tokenId);
633: 
634:         // if base token is ETH then check that caller sent enough for the fee
635:         if (baseToken == address(0) && msg.value < fee) revert InvalidEthAmount();
...
650:         // transfer the fee from the borrower
651:         if (baseToken != address(0)) ERC20(baseToken).transferFrom(msg.sender, address(this), fee);
652: 
653:         return success;
654:     }
```
Calculation of those fees depends on the same variable `changeFee` (which can't be changed after pool deployment):
```solidity
File: PrivatePool.sol
731:     function changeFeeQuote(uint256 inputAmount) public view returns (uint256 feeAmount, uint256 protocolFeeAmount) { 
732:         // multiply the changeFee to get the fee per NFT (4 decimals of accuracy)
733:         uint256 exponent = baseToken == address(0) ? 18 - 4 : ERC20(baseToken).decimals() - 4;
734:         uint256 feePerNft = changeFee * 10 ** exponent;
735: 
736:         feeAmount = inputAmount * feePerNft / 1e18;
737:         protocolFeeAmount = feeAmount * Factory(factory).protocolFeeRate() / 10_000;
738:     }
...
750:     function flashFee(address, uint256) public view returns (uint256) {
751:         return changeFee;
752:     }
```
However, due to the logic of `flashFee()` and `changeFeeQuote()` functions size of those fees would be different dramatically for any of the popular ERC20 token and native eth pools, since `changeFeeQuote()` calculates fee in basis points depending on `baseToken` decimals, while `flashFee()` function return only value of `changeFee` variable.
changeFee should be in basis points (due to the protocol docs) and has `uint56` type with a max value of ~72e15, so it's likely that for most of the popular ERC20 tokens `flashFee()` would be dust value in any case.
It's also opening the attack vector described in the previous Code4rena Caviar report M-04.

## Impact
FlashLoan fee sizes would be unreasonably low for any popular ERC20 tokens and native ETH. Even if the pool owner chooses a `baseToken` with a low decimal number (e.g. USDC) and sets a reasonable value for the `changeFee` (in terms of flash loan fee calculation), this would result in the fee for the `change()` function being too high. As a result, either the `change()`` function would be too expensive for trades to use or the `flashLoan()` calls would not generate any real profit for the pool owner.

## Proof of Concept
A short test added to `test/PrivatePool/Flashloan.t.sol` could demonstrate the difference between fees:
```solidity
    function test_ChangeFee() public {
        // Getting amount of flash fee for 1 NFT
        uint256 flashFee = privatePool.flashFee(address(milady), 1);
        // Getting amount of change fee for 1 NFT 
        (uint256 changeFee, ) = privatePool.changeFeeQuote(1e18);
        // Failed log shows how huge is difference between these two fees
        assertEq(flashFee, changeFee, "Flash loan fee and change fee are different");
    }
```
## Recommended Mitigation Steps
Consider updating `flashFee()` function to next:
```solidity
    function flashFee(address, uint256) public view returns (uint256) {
+       uint256 exponent = baseToken == address(0) ? 18 - 4 : ERC20(baseToken).decimals() - 4;
+       uint256 flashFee = changeFee * 10 ** exponent;
        return flashFee; 
    }
```

## Link to origin issue
https://github.com/code-423n4/2023-04-caviar-findings/issues/243