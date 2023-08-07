# executeOperation() does not check that initiator of flash-loan is the user
## Summary
`MarginTrading` could be fully drained using `executeOperation()` function since there is no check that flash-loan started inside the current contract.

## Vulnerability Detail
`MarginTrading` support flash-loan functionality to interact with Aave `lendingPool`. It could be initiated using `executeFlashLoans()` function:
```solidity
    function executeFlashLoans(
        address[] calldata assets,
        uint256[] calldata amounts,
        uint256[] calldata modes,
        address mainToken,
        bytes calldata params
    ) external onlyFlashLoan {
        address receiverAddress = address(this);

        // the various assets to be flashed

        // the amount to be flashed for each asset

        // 0 = no debt, 1 = stable, 2 = variable

        address onBehalfOf = address(this);
        // bytes memory params = "";
        lendingPool.flashLoan(receiverAddress, assets, amounts, modes, onBehalfOf, params, Types.REFERRAL_CODE);
        emit FlashLoans(assets, amounts, modes, mainToken);
    }
```
Contract logic expects a callback then from `lendingPool` to `executeOperation()` function:
```solidity
    function executeOperation(
        address[] calldata _assets,
        uint256[] calldata _amounts,
        uint256[] calldata _premiums,
        address _initiator,
        bytes calldata _params
    ) external override onlyLendingPool returns (bool) {
        //decode params exe swap and deposit
        {
            (
                uint8 _flag,
                address _swapAddress,
                address _swapApproveTarget,
                address[] memory _swapApproveToken,
                bytes memory _swapParams,
                address[] memory _tradeAssets,
                address[] memory _withdrawAssets,
                uint256[] memory _withdrawAmounts,
                uint256[] memory _rateMode,
                address[] memory _debtTokens
            ) = abi.decode(
                _params,
                (uint8, address, address, address[], bytes, address[], address[], uint256[], uint256[], address[])
            );
            if (_flag == 0 || _flag == 2) {
                //close
                _closetrade(
                    _flag,
                    _swapAddress,
                    _swapApproveTarget,
                    _swapApproveToken,
                    _swapParams,
                    _tradeAssets,
                    _withdrawAssets,
                    _withdrawAmounts,
                    _rateMode,
                    _debtTokens
                );
            }
            if (_flag == 1) {
                //open
                _opentrade(_swapAddress, _swapApproveTarget, _swapApproveToken, _swapParams, _tradeAssets);
            }
        }
        return true;
    }
```
Both functions are gated - `executeFlashLoans()` could be called only by the user directly or using the factory, `executeOperation()` could be called only by `lendingPool`.

However, contract does not check that call to `executeOperation()` was initiated from MarginTrading itself. And due to Aave documentation and contracts logic arbitrary callback receiver could be called using its `flashLoan()` function:

https://github.com/aave/protocol-v2/blob/ice/mainnet-deployment-03-12-2020/contracts/protocol/lendingpool/LendingPool.sol#L508

This allows anybody to initiate flash-loan on Aave and call any `MarginTrading` contract with arbitrary `_params`. For example, this `_params` could be constructed in the way when `_swapAddress` is the token contract address and `_swapParams` is a call to its transfer function, basically allowing an attacker to transfer any assets to its own address. Also, any user position on Aave could be closed and withdrawn with the same approach.

## Impact
Any `MarginTrading` contract could be fully drained by an arbitrary attacker.

## Code Snippet
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L121-L166

## Tool used
Manual Review

## Recommendation
Consider adding a check that the `_initiator` parameter in `executeOperation()` is the appropriate address - user or factory.

## Link to origin issue
https://github.com/sherlock-audit/2023-05-dodo-judging/issues/45