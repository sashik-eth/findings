# Incorrect royalty calculation
## Lines of code
https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L244 

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L274

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L338

## Vulnerability details
The buy() and sell() functions calculate the average sale price of NFTs bought or sold during each trade to provide the royaltyInfo() call and send appropriate amounts of royalties to its receivers:
```solidity
File: PrivatePool.sol
328:         uint256 royaltyFeeAmount = 0;
329:         for (uint256 i = 0; i < tokenIds.length; i++) {
330:             // transfer each nft from the caller
331:             ERC721(nft).safeTransferFrom(msg.sender, address(this), tokenIds[i]);
332: 
333:             if (payRoyalties) {
334:                 // calculate the sale price (assume it's the same for each NFT even if weights differ)
335:                 uint256 salePrice = (netOutputAmount + feeAmount + protocolFeeAmount) / tokenIds.length;
336: 
337:                 // get the royalty fee for the NFT
338:                 (uint256 royaltyFee, address recipient) = _getRoyalty(tokenIds[i], salePrice);
339: 
340:                 // tally the royalty fee amount
341:                 royaltyFeeAmount += royaltyFee;
342: 
343:                 // transfer the royalty fee to the recipient if it's greater than 0
344:                 if (royaltyFee > 0 && recipient != address(0)) {
345:                     if (baseToken != address(0)) {
346:                         ERC20(baseToken).safeTransfer(recipient, royaltyFee);
347:                     } else {
348:                         recipient.safeTransferETH(royaltyFee);
349:                     }
350:                 }
351:             }
352:         }
```
However, each NFT's price should be calculated based on its own weight, as different amounts may be received as royalties for NFTs traded in different buy() or sell() calls due to the individual royalty calculation logic. Also, the royalty recipient addresses may not necessarily be the same for all tokens in the collection. Therefore, different royalties receivers receive the same amount of royalties in case "their" tokens would be traded in the same call, but with different weights(=prices).

## Impact
Royalty receivers could receive the wrong amount of royalties.

## Proof of Concept
Royalty calculation due to [EIP-2981](https://eips.ethereum.org/EIPS/eip-2981) could have custom logic in terms of how and who would receive a royalty on which id from the collection. Consider two scenarios where royalty receivers get the wrong amounts due to describing the issue:

1.1 Alice creates an NFT collection with custom royaltyInfo logic that specifies a 10% royalty fee when the sale price is greater than 1e18 and 0% otherwise.
1.2 Alice creates a Caviar private pool where half of the collection has a weight of 1e18 and the other half has a weight of 2e18.
1.3 Bob purchases two NFTs (one with a weight of 1e18 and the other with a weight of 2e18) in a single buy() call for a total price of 2.5 ETH, paying 0.25 ETH as a royalty fee (0.125 ETH per NFT). In this case, the amount of royalties paid is incorrect, as Bob should only pay a royalty fee for the NFT with a weight of 2e18 as he would if he bought the same NFTs in different calls.

2.1 Alice creates an NFT collection with custom royaltyInfo logic that specifies that the royalty receiver for half of the collection ids is herself, while the other half - is Charlie.
2.2 Alice creates a Caviar private pool where half of the collection has a weight of 1e18 (those where she is a royalty receiver) and the other half has a weight of 2e18 (those where Charlie is a royalty receiver).
2.3 Bob purchases two NFTs (one with a weight of 1e18 and the other with a weight of 2e18) in a single buy() call for a total price of 2.5 ETH, paying 0.25 ETH as a royalty fee (0.125 ETH per NFT). In this case, both royalty receivers got the same amount of funds, while it would (and should) be different if Bob would buy the same NFTs in separate transactions.

## Recommended Mitigation Steps
Consider the calculation of the royalty amount based on the token's individual weights:
```solidity
+   uint256 tokenPrice = (netOutputAmount + feeAmount + protocolFeeAmount) * tokenWeights[i] / weightSum;
+   (uint256 royaltyFee,) = _getRoyalty(tokenIds[i], tokenPrice); 
-   uint256 salePrice = (netOutputAmount + feeAmount + protocolFeeAmount) / tokenIds.length;
-   (uint256 royaltyFee,) = _getRoyalty(tokenIds[i], salePrice); 
```
## Link to origin issue
https://github.com/code-423n4/2023-04-caviar-findings/issues/175