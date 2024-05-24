# [H-1] - Minting `works` is supposed to refund excess ETH, but it doesn't

## Summary

Minting functions are supposed to send back to the user any amount of ETH remaining after minting a token and paying for the fees, but it doesn't.

## Vulnerability Detail

The `Edition.sol` contract has 4 minting functions in which they all call `Edition::_refundExcess`:

```javascript
    function mint(address to_, uint256 tokenId_, uint256 amount_, address referrer_, bytes calldata data_) external payable override {
        // wake-disable-next-line reentrancy
        FEE_MANAGER.collectMintFee{value: msg.value}(
            this, tokenId_, amount_, msg.sender, referrer_, works[tokenId_].strategy
        );

        _issue(to_, tokenId_, amount_, data_);
@>      _refundExcess();
    }
```
```javascript
    function mintWithComment(address to_, uint256 tokenId_, uint256 amount_, address referrer_, bytes calldata data_, string calldata comment_) external payable {
        Strategy memory strategy = works[tokenId_].strategy;
        // wake-disable-next-line reentrancy
        FEE_MANAGER.collectMintFee{value: msg.value}(
            this, tokenId_, amount_, msg.sender, referrer_, strategy
        );

        _issue(to_, tokenId_, amount_, data_);
@>      _refundExcess();

        emit Comment(address(this), tokenId_, to_, comment_);
    }
```
```javascript
    function mintBatch(address to_, uint256[] calldata tokenIds_, uint256[] calldata amounts_, bytes calldata data_) external payable {
        for (uint256 i = 0; i < tokenIds_.length; i++) {
            Work storage work = works[tokenIds_[i]];

            // wake-disable-next-line reentrancy
            FEE_MANAGER.collectMintFee{value: msg.value}(
                this, tokenIds_[i], amounts_[i], msg.sender, address(0), work.strategy
            );

            _checkTime(work.opensAt, work.closesAt);
            _updateSupply(work, amounts_[i]);
        }

        _batchMint(to_, tokenIds_, amounts_, data_);
@>      _refundExcess();
    }
```
```javascript
    function mintBatch(address[] calldata receivers_, uint256 tokenId_, uint256 amount_, bytes calldata data_) external payable {
        // wake-disable-next-line reentrancy
        FEE_MANAGER.collectMintFee{value: msg.value}(
            this, tokenId_, amount_, msg.sender, address(0), works[tokenId_].strategy
        );

        for (uint256 i = 0; i < receivers_.length; i++) {
            _issue(receivers_[i], tokenId_, amount_, data_);
        }

@>        _refundExcess();
    }
```

As the documentaion of `_refundExcess` describes, this function is supposed to refund to the `user` any excess ETH he sent after all the fees have been collected. Please note that there isn't a `receive` or `fallback` function to the contract, the only way to get ETH into the contract is through the minting functions (plus the `grantRoles` and `revokeRoles` functions that are payable, but must be payable to correctly override Solady's similar payable functions, they are also only callable only by the `EDITION_MANAGER_ROLE`).

```javascript
    /// @notice Refund any excess ETH sent to the contract.
    /// @dev This function is called after minting tokens to refund any ETH left in the contract after all fees have been collected.
    function _refundExcess() internal {
        if (msg.value > 0 && address(this).balance > 0) {
            msg.sender.safeTransferETH(address(this).balance);
        }
    }
```

Now that we have established the intended functionality of the contract let's see what actually happens. All the minting functions above transfer the whole of `msg.value` to the `FeeManager.sol` contract.

```javascript
        FEE_MANAGER.collectMintFee{value: msg.value}(
            this, tokenId_, amount_, msg.sender, referrer_, works[tokenId_].strategy
        );
```

The ETH remaining is never sent back to this contract from `FeeManager.sol` in any way. As a result, `refundExcess` never has any ETH left in the contract to send to the user. A user can send big amounts of ETH when minting, expecting to get back what remains and in the end he loses all his ETH. In addition, any excess ETH remains in the `FeeManager.sol` contract and is available to get refunded only by `ADMIN_ROLE`

```javascript
    /// @notice An escape hatch to transfer any trapped assets from the contract to the given address.
    /// @param asset_ The address of the asset to withdraw.
    /// @param amount_ The amount of the asset to withdraw.
    /// @param to_ The address to send the asset to.
    /// @dev This is meant to be used in cases where the contract is holding assets that it should not be. This function can only be called by an admin.
    function withdraw(address asset_, uint256 amount_, address to_) external onlyRolesOrOwner(ADMIN_ROLE)
    {
        _transfer(asset_, amount_, address(this), to_);
    }
```

## Impact

A user sending excess amounts of ETH is expecting the protocol to only use the ETH that is needed and refund him the rest. This doesn't happen, resulting in the user losing all his ETH.

## Proof of Code

Paste the following code into `Edition.t.sol` test file among the rest of the tests

```javascript
    function test_mint_doesnt_refund() public {

        // a mint in this case costs 0.0106 ETH
        uint256 balanceBefore = address(this).balance;
        edition.mint{value: 0.0106 ether}(address(1), 1, 1, address(0), new bytes(0));
        uint256 balanceAfter = address(this).balance;

        assertEq(address(1).balance, 0.01 ether);
        assertEq(address(0xc0ffee).balance, 0.0006 ether);

        // user is correctly charged and all the fees are distributed correctly
        assert(balanceBefore - 0.0106 ether == balanceAfter);

        // now user sends 10 eth expecting a refund
        balanceBefore = address(this).balance;
        edition.mint{value: 10 ether}(address(1), 1, 1, address(0), new bytes(0));
        balanceAfter = address(this).balance;

        assertEq(address(1).balance, 0.02 ether);
        assertEq(address(0xc0ffee).balance, 0.0012 ether);

        // all the fees are distributed correctly but user has lost 10 eth
        assert(balanceBefore - 10 ether == balanceAfter);
    }
```

## Code Snippet
[https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L228-L320](https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L228-L320)
[https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L512](https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L512
)
[https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L352](https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L352)

## Tool used

Manual Review

## Recommendation

My recommendation is to calculate the fee first and only transfer the necessary ETH to `FeeManager.sol`

```diff
    function mint(address to_, uint256 tokenId_, uint256 amount_, address referrer_, bytes calldata data_) external payable override {
+       uint256 fee = mintFee(tokenId_, amount_);
+       FEE_MANAGER.collectMintFee{value: fee}(
-       FEE_MANAGER.collectMintFee{value: msg.value}(
            this, tokenId_, amount_, msg.sender, referrer_, works[tokenId_].strategy
        );

        _issue(to_, tokenId_, amount_, data_);
        _refundExcess();
    }


    function mintWithComment(address to_, uint256 tokenId_, uint256 amount_, address referrer_, bytes calldata data_, string calldata comment_) external payable {
        Strategy memory strategy = works[tokenId_].strategy;
+       uint256 fee = mintFee(tokenId_, amount_);
+       FEE_MANAGER.collectMintFee{value: fee}(
-       FEE_MANAGER.collectMintFee{value: msg.value}(
            this, tokenId_, amount_, msg.sender, referrer_, strategy
        );

        _issue(to_, tokenId_, amount_, data_);
        _refundExcess();

        emit Comment(address(this), tokenId_, to_, comment_);
    }


    function mintBatch(address to_, uint256[] calldata tokenIds_, uint256[] calldata amounts_, bytes calldata data_) external payable {
+     uint256 fee;
        for (uint256 i = 0; i < tokenIds_.length; i++) {
            Work storage work = works[tokenIds_[i]];

+          fee = mintFee(tokenIds_[i], amounts_[i]);
+          FEE_MANAGER.collectMintFee{value: fee}(
-          FEE_MANAGER.collectMintFee{value: msg.value}(
                this, tokenIds_[i], amounts_[i], msg.sender, address(0), work.strategy
            );

            _checkTime(work.opensAt, work.closesAt);
            _updateSupply(work, amounts_[i]);
        }

        _batchMint(to_, tokenIds_, amounts_, data_);
        _refundExcess();
    }


    function mintBatch(address[] calldata receivers_, uint256 tokenId_, uint256 amount_, bytes calldata data_) external payable {
+       // Here I'm not sure but I'm guessing the user is paying for all the addresses/receivers so:

+       uint256 fee = mintFee(tokenId_, amount_) * receivers_.length;
+       FEE_MANAGER.collectMintFee{value: fee}(
-       FEE_MANAGER.collectMintFee{value: msg.value}(
            this, tokenId_, amount_, msg.sender, address(0), works[tokenId_].strategy
        );

        for (uint256 i = 0; i < receivers_.length; i++) {
            _issue(receivers_[i], tokenId_, amount_, data_);
        }

        _refundExcess();
    }
```

After the above implementations the tests are running correctly and the account is refunded as it should.

# [M-1] - `CREATE` opcode works differently in the zkSync chain

## Summary

`zkSync Era` chain has differences in the usage of the `create` opcode compared to the EVM.

## Vulnerability Detail

According to the contest README, the protocol can be deployed in zkSync Era ([README](https://github.com/sherlock-audit/2024-04-titles/blob/d7f60952df22da00b772db5d3a8272a988546089/README.md?plain=1#L11))

The zkSync Era docs explain how it differs from Ethereum.

The description of CREATE and CREATE2 ([zkSynce Era Docs](https://docs.zksync.io/build/developer-reference/differences-with-ethereum.html#create-create2)) states that Create cannot be used for arbitrary code unknown to the compiler.

`TitlesCore::createEdition` function uses Solady's `LibClone::clone` function:

```javascript
    function createEdition(bytes calldata payload_, address referrer_) external payable returns (Edition edition) {

        EditionPayload memory payload = abi.decode(payload_.cdDecompress(), (EditionPayload));

@>      edition = Edition(editionImplementation.clone()); 

        // wake-disable-next-line reentrancy
        edition.initialize(
            feeManager, graph, payload.work.creator.target, address(this), payload.metadata
        );

        // wake-disable-next-line unchecked-return-value
        _publish(edition, payload.work, referrer_);

        emit EditionCreated(
            address(edition),
            payload.work.creator.target,
            payload.work.maxSupply,
            payload.work.strategy,
            abi.encode(payload.metadata)
        );
    }
```
```javascript
    function clone(uint256 value, address implementation) internal returns (address instance) {
        /// @solidity memory-safe-assembly
        assembly {
            mstore(0x21, 0x5af43d3d93803e602a57fd5bf3)
            mstore(0x14, implementation)
            mstore(0x00, 0x602c3d8160093d39f33d3d3d3d363d3d37363d73)
@>          instance := create(value, 0x0c, 0x35)
            if iszero(instance) {
                mstore(0x00, 0x30116425) // `DeploymentFailed()`.
                revert(0x1c, 0x04)
            }
            mstore(0x21, 0) // Restore the overwritten part of the free memory pointer.
        }
    }
```

As mentioned by the zkSync docs: "The code will not function correctly because the compiler is not aware of the bytecode beforehand".

This will result in loss of funds, since there is a fee to create a new edition, hence the `TitlesCore::createEdition` function is payable.

## Impact

No editions can be created in the zkSync Era chain.

## Code Snippet

[https://github.com/sherlock-audit/2024-04-titles/blob/d7f60952df22da00b772db5d3a8272a988546089/wallflower-contract-v2/src/TitlesCore.sol#L79](https://github.com/sherlock-audit/2024-04-titles/blob/d7f60952df22da00b772db5d3a8272a988546089/wallflower-contract-v2/src/TitlesCore.sol#L79)
[https://github.com/Vectorized/solady/blob/91d5f64b39a4d20a3ce1b5e985103b8ea4dc1cfc/src/utils/LibClone.sol#L137](https://github.com/Vectorized/solady/blob/91d5f64b39a4d20a3ce1b5e985103b8ea4dc1cfc/src/utils/LibClone.sol#L137)

## Tool used

Manual Review

## Recommendation

This can be solved by implementing CREATE2 directly and using `type(Edition).creationCode` as mentioned in the zkSync Era docs.

# [M-2] - `Editions::mintBatch` function will always revert

## Summary

`Editions::mintBatch` function used to mint more than 1 tokens to a single receiver will always revert.

## Vulnerability Detail

This is the `Editions::mintBatch` function:

```javascript
    function mintBatch(
        address to_,
        uint256[] calldata tokenIds_,
        uint256[] calldata amounts_,
        bytes calldata data_
    ) external payable {
        for (uint256 i = 0; i < tokenIds_.length; i++) {
            Work storage work = works[tokenIds_[i]];

            // wake-disable-next-line reentrancy
@>          FEE_MANAGER.collectMintFee{value: msg.value}(
                this, tokenIds_[i], amounts_[i], msg.sender, address(0), work.strategy
            );

            _checkTime(work.opensAt, work.closesAt);
            _updateSupply(work, amounts_[i]);
        }

        _batchMint(to_, tokenIds_, amounts_, data_);
        _refundExcess();
    }
```

As we can see, the function is sending the whole `msg.value` to the `FeeManager.sol` contract inside a `for loop`. This means, if the `tokenIds_` array has a `length > 1`, then the function will revert because it has already sent the whole `msg.value` and at the 2nd loop there are no more funds to send.

## Impact

The `mintBatch` function is unusable as it will always revert if a user wants to mint more than 1 tokens, which is the whole purpose of this function. Otherwise, the user would just use the `mint` function.

## Proof of Code

Paste the following code in `Edition.t.sol` file:

```javascript
    uint256[] tokenIds = [1, 1, 1];
    uint256[] amounts = [1, 1, 1];

    function test_mint_batch_doesnt_work() public {
        vm.expectRevert();
        edition.mintBatch{value: 10 ether}(address(1), tokenIds, amounts, new bytes(0));
    }
```

Running the test with `forge test --mt test_mint_batch_doesnt_work -vvvvv` we can see that it passes (so it reverts as expected) and it also reverts with `OutOfFunds` error.

## Code Snippet

[https://github.com/sherlock-audit/2024-04-titles/blob/d7f60952df22da00b772db5d3a8272a988546089/wallflower-contract-v2/src/editions/Edition.sol#L277](https://github.com/sherlock-audit/2024-04-titles/blob/d7f60952df22da00b772db5d3a8272a988546089/wallflower-contract-v2/src/editions/Edition.sol#L277)

## Tool used

Manual Review

## Recommendation

My recommendation is to calculate the necessary fee in each loop and only transfer the fee for each mint:

```diff
    function mintBatch(address to_, uint256[] calldata tokenIds_, uint256[] calldata amounts_, bytes calldata data_) external payable {
+       uint256 fee;
        for (uint256 i = 0; i < tokenIds_.length; i++) {
            Work storage work = works[tokenIds_[i]];

+          fee = mintFee(tokenIds_[i], amounts_[i]);
+          FEE_MANAGER.collectMintFee{value: fee}(
-          FEE_MANAGER.collectMintFee{value: msg.value}(
                this, tokenIds_[i], amounts_[i], msg.sender, address(0), work.strategy
            );

            _checkTime(work.opensAt, work.closesAt);
            _updateSupply(work, amounts_[i]);
        }

        _batchMint(to_, tokenIds_, amounts_, data_);
        _refundExcess();
    }
```
