# The Concept

Inbetween my normal audits, when there is a lot of free time, I thought to take a quick look at other ongoing audits and see if I can find anything on my 1st or 2nd pass of the codebase. I do this as a small challenge to make me pay more attention on my 1st and 2nd passes of the code.

Why?

- Improve my initial look at a codebase.
- Slightly boost my leaderboard rank on different audit platforms.
- Slightly boost my valid findings ratio on different audit platforms.
- Potentially make a small amount of extra money, although most bugs found on the 1st and 2nd passes will be heavily duplicated.
- Get a decent idea of codebases so after I can read the reports and understand them.
- Expose myself to different coding techniques and ideas.

# The results

### Folks Finance - Immunefi - 212,89 USDC

<details>
<summary>[M-01] - Loan creation can be frontrun, preventing the users from creating loans</summary>

## Brief/Intro

A user who tries to create a loan has to choose the `loanId`. Any user can frontrun this transaction with the same `loanId`, making the initial user's transaction to revert because his selected `loanId` is taken.

## Vulnerability Details

Each loan has a unique `bytes32` identifier named `loanId`. During the loan creation, each user is asked to provide the `loanId` that his loan will have.

```javascript
SpokeCommon.sol

    function createLoan(
        Messages.MessageParams memory params,
        bytes32 accountId,
@>      bytes32 loanId,
        uint16 loanTypeId,
        bytes32 loanName
    ) external payable nonReentrant {
        _doOperation(params, Messages.Action.CreateLoan, accountId, abi.encodePacked(loanId, loanTypeId, loanName));
    }
```

```javascript
SpokeToken.sol

    function createLoanAndDeposit(
        Messages.MessageParams memory params,
        bytes32 accountId,
@>      bytes32 loanId,
        uint256 amount,
        uint16 loanTypeId,
        bytes32 loanName
    ) external payable nonReentrant {
        _doOperation(
            params,
            Messages.Action.CreateLoanAndDeposit,
            accountId,
            amount,
            abi.encodePacked(loanId, poolId, amount, loanTypeId, loanName)
        );
    }
```

This arbitrary `loanId` value is sent through a bridge to the `Hub.sol` contract which in turn calls the `createUserLoan` function is `LoanManager.sol`.

```javascript
Hub.sol

    function _receiveMessage(Messages.MessageReceived memory message) internal override {
        Messages.MessagePayload memory payload = Messages.decodeActionPayload(message.payload);
        .
        .
        .
        } else if (payload.action == Messages.Action.CreateLoan) {
            bytes32 loanId = payload.data.toBytes32(index);
            index += 32;
            uint16 loanTypeId = payload.data.toUint16(index);
            index += 2;
            bytes32 loanName = payload.data.toBytes32(index);

@>          loanManager.createUserLoan(loanId, payload.accountId, loanTypeId, loanName);
        } else if (payload.action == Messages.Action.DeleteLoan) {
            bytes32 loanId = payload.data.toBytes32(index);

            loanManager.deleteUserLoan(loanId, payload.accountId);
        } else if (payload.action == Messages.Action.CreateLoanAndDeposit) {
            bytes32 loanId = payload.data.toBytes32(index);
            index += 32;
            uint8 poolId = payload.data.toUint8(index);
            index += 1;
            uint256 amount = payload.data.toUint256(index);
            index += 32;
            uint16 loanTypeId = payload.data.toUint16(index);
            index += 2;
            bytes32 loanName = payload.data.toBytes32(index);

@>          loanManager.createUserLoan(loanId, payload.accountId, loanTypeId, loanName);
            loanManager.deposit(loanId, payload.accountId, poolId, amount);

            // save token received
            receiveToken = ReceiveToken({poolId: poolId, amount: amount});
        } else if (payload.action == Messages.Action.Deposit) {
        .
        .
        .
```

```javascript
LoanManager.sol

    function createUserLoan(
        bytes32 loanId,
        bytes32 accountId,
        uint16 loanTypeId,
        bytes32 loanName
    ) external override onlyRole(HUB_ROLE) nonReentrant {
        // check loan types exists, is not deprecated and no existing user loan for same loan id
        if (!isLoanTypeCreated(loanTypeId)) revert LoanTypeUnknown(loanTypeId);
        if (isLoanTypeDeprecated(loanTypeId)) revert LoanTypeDeprecated(loanTypeId);
@>      if (isUserLoanActive(loanId)) revert UserLoanAlreadyCreated(loanId);

        // create loan
        UserLoan storage userLoan = _userLoans[loanId];
        userLoan.isActive = true;
        userLoan.accountId = accountId;
        userLoan.loanTypeId = loanTypeId;

        emit CreateUserLoan(loanId, accountId, loanTypeId, loanName);
    }
```

At this point, if there is already a loan with the desired `loanId`, the transaction reverts. Upon a valid loan creation, a new `UserLoan` object is created and `UserLoan.isActive` is set to `true`.

```javascript
    function isUserLoanActive(bytes32 loanId) public view returns (bool) {
        return _userLoans[loanId].isActive;
    }
```

An attacker can take advantage of this and frontrun all the loan creation transactions (on the chains with a public mempool, like the `Ethereum mainnet`) and prevent all the users from creating loans.

## Impact Details

This is a griefing attack which prevents all users from creating loans. Every transaction will fail because the attacker can frontrun it with the same `loanId`.

## References

https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/spoke/SpokeCommon.sol#L115

https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/spoke/SpokeToken.sol#L46

https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/Hub.sol#L186-L210

https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/LoanManager.sol#L40

https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/LoanManagerState.sol#L413

## Recommendation

Don't allow for the users to select their desired `loanId`. Use a counter internally and increment it with every loan creation and use it as the `loanId`.


## Proof of Concept

Let's follow this scenario:

1. Bob tries to create a loan with a random `loanId`
2. Alice (the attacker) sees this transaction in the mempool and frontruns bob transaction with the same `loanId`
3. Alice's transaction goes through
4. Bob's transaction gets reverted
5. Repeat

Paste the following test in the `test/hub/LoanManager.test.ts`:

```javascript
  describe("POCs", () => {
    it("Should test loanId frontrun", async () => {
      const { hub, loanManager } = await loadFixture(deployLoanManagerFixture);
      const { loanTypeId } = await loadFixture(addPoolsFixture);

      const loanId = getRandomBytes(BYTES32_LENGTH);
      const accountId1 = getAccountIdBytes("ACCOUNT_ID");
      const accountId2 = getAccountIdBytes("ACCOUNT_ID2");
      const loanName = getRandomBytes(BYTES32_LENGTH);

      // frontrunning transaction
      const createUserLoan2 = await loanManager.connect(hub).createUserLoan(loanId, accountId2, loanTypeId, loanName);

      // initial transaction
      const createUserLoan = loanManager.connect(hub).createUserLoan(loanId, accountId1, loanTypeId, loanName);

      await expect(createUserLoan)
        .to.be.revertedWithCustomError(loanManager, "UserLoanAlreadyCreated")
        .withArgs(loanId);
    });


  });
```

</details>

<details>
<summary>[M-02] - Account creation can be frontrun, making the users unable to create an account</summary>

## Brief/Intro

A user who tries to create an account for the protocol has to choose his `accountId`. Any user can frontrun this transaction with the same `accountId`, making the initial user's transaction to revert because his selected `accountId` is taken.

## Vulnerability Details

Each account has a unique `bytes32` identifier named `accountId`. During the account creation, each user is asked to provide the `accountId` that his account will have.

```javascript
SpokeCommon.sol

    function createAccount(
        Messages.MessageParams memory params,
@>      bytes32 accountId,
        bytes32 refAccountId
    ) external payable nonReentrant {
        _doOperation(params, Messages.Action.CreateAccount, accountId, abi.encodePacked(refAccountId));
    }
```

This arbitrary `accountId` value is sent through a bridge to the `Hub.sol` contract which in turn calls the createAccount function is `AccountManager.sol`.

```javascript
Hub.sol

    function _receiveMessage(Messages.MessageReceived memory message) internal override {
        Messages.MessagePayload memory payload = Messages.decodeActionPayload(message.payload);
        .
        .
        .
        if (payload.action == Messages.Action.CreateAccount) {
            bytes32 refAccountId = payload.data.toBytes32(index);

@>          accountManager.createAccount(payload.accountId, message.sourceChainId, payload.userAddress, refAccountId);
        } else if
        .
        .
        .
    }
```

```javascript
AccountManager.sol

    function createAccount(
        bytes32 accountId,
        uint16 chainId,
        bytes32 addr,
        bytes32 refAccountId
    ) external override onlyRole(HUB_ROLE) {
        // check account is not already created (empty is reserved for admin)
@>      if (isAccountCreated(accountId) || accountId == bytes32(0)) revert AccountAlreadyCreated(accountId);
        .
        .
        .
    }
```

At this point, if there is already an account with the desired `accountId`, the transaction reverts. An attacker can take advantage of this and frontrun all the account creation transactions (on the chains with a public mempool, like the `Ethereum mainnet`) and prevent all the users from creating an account, which is essential for someone to use the protocol.

## Impact Details

This is a griefing attack which prevents any new users from using the protocol, since they can't create an account. Every transaction will fail because the attacker can frontrun it with the same `accountId`.

## References

https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/spoke/SpokeCommon.sol#L27

https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/Hub.sol#L163

https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/AccountManager.sol#L42

## Recommendation

Don't allow for the users to select their desired `accountId`. Use a counter internally and increment it with every account creation and use it as the `accountId`.


## Proof of Concept

Let's follow this scenario:

1. Bob tries to create an account with `accountId = "BOB_ACCOUNT_ID"`
2. Alice (the attacker) sees this transaction in the mempool and frontruns bob transaction with `accountId = "BOB_ACCOUNT_ID"`
3. Alice's transaction goes through
4. Bob's transaction gets reverted
5. Repeat

Add the following test in the `test/AccountManager.test.ts` file under the `describe("Create Account", () => {` tab.

```javascript
  describe("POCs", () => {
    it("Should test loanId frontrun", async () => {
      const { hub, loanManager } = await loadFixture(deployLoanManagerFixture);
      const { loanTypeId } = await loadFixture(addPoolsFixture);

      const loanId = getRandomBytes(BYTES32_LENGTH);
      const accountId1 = getAccountIdBytes("ACCOUNT_ID");
      const accountId2 = getAccountIdBytes("ACCOUNT_ID2");
      const loanName = getRandomBytes(BYTES32_LENGTH);

      // frontrunning transaction
      const createUserLoan2 = await loanManager.connect(hub).createUserLoan(loanId, accountId2, loanTypeId, loanName);

      // initial transaction
      const createUserLoan = loanManager.connect(hub).createUserLoan(loanId, accountId1, loanTypeId, loanName);

      await expect(createUserLoan)
        .to.be.revertedWithCustomError(loanManager, "UserLoanAlreadyCreated")
        .withArgs(loanId);
    });


  });
```

</details>

### Fjord Token Staking - CodeHawks - $0.20

<details>
<summary>[M-01] - Wrong `FjordAuction` owner will result in loss of all the tokens in case of 0 bids</summary>
    
## Summary

The `FjordAuction.sol` contract initializes its `owner` address during construction. This `owner` address is used to send the `totalTokens` back at the end of the auction in the case that there are no bids. However, when creating a new `FjordAuction` through the `FjordAuctionFactory` contract, the `FjordAuction`'s `owner` will be set to be the `FjordAuctionFactory` address. Sending any tokens to that address will lock them forever, since there is no withdrawal mechanism implemented.

## Vulnerability Details

```js
FjordAuction.sol

    constructor(
        address _fjordPoints,
        address _auctionToken,
        uint256 _biddingTime,
        uint256 _totalTokens
    ) {
        if (_fjordPoints == address(0)) {
            revert InvalidFjordPointsAddress();
        }
        if (_auctionToken == address(0)) {
            revert InvalidAuctionTokenAddress();
        }
        fjordPoints = ERC20Burnable(_fjordPoints);
        auctionToken = IERC20(_auctionToken);
@>      owner = msg.sender;
        auctionEndTime = block.timestamp.add(_biddingTime);
        totalTokens = _totalTokens;
    }
```

We see that the `owner` of the `FjordAuction` contract is set to the address that deploys the contract. Creating a new auction through the `FjordAuctionFactory` contract, makes `FjordAuctionFactory`'s address the deployer of the contract and the `owner` of the `FjordAuctions` it deploys.

```js
    function auctionEnd() external {
        if (block.timestamp < auctionEndTime) {
            revert AuctionNotYetEnded();
        }
        if (ended) {
            revert AuctionEndAlreadyCalled();
        }

        ended = true;
        emit AuctionEnded(totalBids, totalTokens);

        if (totalBids == 0) {
@>          auctionToken.transfer(owner, totalTokens);
            return;
        }

        multiplier = totalTokens.mul(PRECISION_18).div(totalBids);

        // Burn the FjordPoints held by the contract
        uint256 pointsToBurn = fjordPoints.balanceOf(address(this));
        fjordPoints.burn(pointsToBurn);
    }
```

If there are 0 `bids` in the auction, when it ends it sends all the tokens to the `owner`. Any tokens sent to the `FjordAuctionFactory` will be lost, because there is no way to withdraw/recover any tokens from this contract.

## Impact

In case there are 0 `bids` in an auction, all the tokens of the auction will be sent to `FjordAuctionFactory`'s address and will be lost forever.

## Proof Of Concept

Create a new file `test/unit/auctionFactory.t.sol` and paste the following code:

```js
// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity =0.8.21;

import "forge-std/Test.sol";
import "src/FjordAuction.sol";
import "src/FjordAuctionFactory.sol";
import { ERC20BurnableMock } from "../mocks/ERC20BurnableMock.sol";
import { SafeMath } from "lib/openzeppelin-contracts/contracts/utils/math/SafeMath.sol";

contract TestAuctionFactory is Test {
    using SafeMath for uint256;

    AuctionFactory public auctionFactory;
    ERC20BurnableMock public fjordPoints;
    ERC20BurnableMock public auctionToken;
    address public owner = address(0x1);
    uint256 public biddingTime = 1 weeks;
    uint256 public totalTokens = 1000 ether;

    function setUp() public {
        fjordPoints = new ERC20BurnableMock("FjordPoints", "fjoPTS");
        auctionToken = new ERC20BurnableMock("AuctionToken", "AUCT");
        vm.startPrank(owner);
        auctionFactory = new AuctionFactory(address(fjordPoints));
        vm.stopPrank();

        deal(address(auctionToken), owner, totalTokens);
    }

    function testAuctionOwner() public {
        bytes32 salt = "1";
        vm.startPrank(owner);
        auctionToken.approve(address(auctionFactory), totalTokens);
        address auction =
            auctionFactory.createAuction(address(auctionToken), biddingTime, totalTokens, salt);
        vm.stopPrank();

        vm.assertEq(FjordAuction(auction).owner(), address(auctionFactory));
        assert(FjordAuction(auction).owner() != owner);
    }
}
```

Also, the `FjordAuctionFactory.sol` file was modified for this test so I could recover the `auctionAddress` after creating the new `FjordAuction` contract:

```diff
FjordAuctionFactory.sol

    function createAuction(
        address auctionToken,
        uint256 biddingTime,
        uint256 totalTokens,
        bytes32 salt
-   ) external onlyOwner {
+   ) external onlyOwner returns (address) {
        address auctionAddress = address(
            new FjordAuction{ salt: salt }(fjordPoints, auctionToken, biddingTime, totalTokens)
        );

        // Transfer the auction tokens from the msg.sender to the new auction contract
        IERC20(auctionToken).transferFrom(msg.sender, auctionAddress, totalTokens);

        emit AuctionCreated(auctionAddress);

+       return auctionAddress;
    }
```

## Tools Used

Manual Review

## Recommendations

During the `FjordAuction` construction, add another address for the correct owner:

```diff
FjordAuction.sol

    constructor(
+       address _owner,
        address _fjordPoints,
        address _auctionToken,
        uint256 _biddingTime,
        uint256 _totalTokens
    ) {
        if (_fjordPoints == address(0)) {
            revert InvalidFjordPointsAddress();
        }
        if (_auctionToken == address(0)) {
            revert InvalidAuctionTokenAddress();
        }
        fjordPoints = ERC20Burnable(_fjordPoints);
        auctionToken = IERC20(_auctionToken);
-       owner = msg.sender;
+       owner = _owner;
        auctionEndTime = block.timestamp.add(_biddingTime);
        totalTokens = _totalTokens;
    }
```

```diff
FjordAuctionFactory.sol

    function createAuction(
        address auctionToken,
        uint256 biddingTime,
        uint256 totalTokens,
        bytes32 salt
    ) external onlyOwner {
        address auctionAddress = address(
-           new FjordAuction{ salt: salt }(fjordPoints, auctionToken, biddingTime, totalTokens)
+           new FjordAuction{ salt: salt }(owner, fjordPoints, auctionToken, biddingTime, totalTokens)
        );

        // Transfer the auction tokens from the msg.sender to the new auction contract
        IERC20(auctionToken).transferFrom(msg.sender, auctionAddress, totalTokens);

        emit AuctionCreated(auctionAddress);
    }
```

</details>

### Zaros Part 1 - CodeHawks - $24.04

<details>
<summary>[M-01] - Insufficient checks to confirm the correct status of the `sequencerUptimeFeed`</summary>

## Summary

The `ChainlinkUtil.sol` contract has `sequencerUptimeFeed` checks in place to assert if the sequencer on `Arbitrum` is running, but these checks are not implemented correctly. Since the protocol implements some checks for the `sequencerUptimeFeed` status, it should implement all of the checks.

## Vulnerability Details

The [chainlink docs](https://docs.chain.link/data-feeds/l2-sequencer-feeds) say that `sequencerUptimeFeed` can return a 0 value for `startedAt` if it is called during an "invalid round".

> * startedAt: This timestamp indicates when the sequencer changed status. This timestamp returns `0` if a round is invalid. When the sequencer comes back up after an outage, wait for the `GRACE_PERIOD_TIME` to pass before accepting answers from the data feed. Subtract `startedAt` from `block.timestamp` and revert the request if the result is less than the `GRACE_PERIOD_TIME`.

Please note that an "invalid round" is described to mean there was a problem updating the sequencer's status, possibly due to network issues or problems with data from oracles, and is shown by a `startedAt` time of 0 and `answer` is 0. Further explanation can be seen as given by an official chainlink engineer as seen here in the chainlink public discord:

[Chainlink Discord Message](https://discord.com/channels/592041321326182401/605768708266131456/1213847312141525002) (must be a member of the Chainlink Discord Channel to view)

Bharath | Chainlink Labs — 03/03/2024 3:55 PM:

> Hello, @EricTee An "invalid round" means there was a problem updating the sequencer's status, possibly due to network issues or problems with data from oracles, and is shown by a `startedAt` time of 0. Normally, when a round starts, `startedAt` is recorded, and the initial status (`answer`) is set to `0`. Later, both the answer and the time it was updated (`updatedAt`) are set at the same time after getting enough data from oracles, making sure that answer only changes from `0` when there's a confirmed update different from the start time. This process helps avoid mistakes in judging if the sequencer is available, which could cause security issues. Making sure `startedAt` isn't `0` is crucial for keeping the system secure and properly informed about the sequencer's status.

Quoting Chainlink's developer final statement:
"Making sure `startedAt` isn't `0` is crucial for keeping the system secure and properly informed about the sequencer's status."

This also makes the implemented check below in the `ChainlinkUtil::getPrice` to be useless if its called in an invalid round:

```js
                uint256 timeSinceUp = block.timestamp - startedAt;
                if (timeSinceUp <= Constants.SEQUENCER_GRACE_PERIOD_TIME) {
                    revert Errors.GracePeriodNotOver();
                }
```

as `startedAt` will be `0`, the arithmetic operation `block.timestamp - startedAt` will result in a value greater than `SEQUENCER_GRACE_PERIOD_TIME` (which is hardcoded to be 3600) i.e block.timestamp = 1719739032, so 1719739032 - 0 = 1719739032 which is bigger than 3600. The code won't revert.

Imagine a case where a round starts, at the beginning `startedAt` is recorded to be 0, and `answer`, the initial status is set to be `0`. Note that docs say that if `answer = 0`, sequencer is up, if equals to `1`, sequencer is down. But in this case here, `answer` and `startedAt` can be `0` initially, till after all data is gotten from oracles and update is confirmed then the values are reset to the correct values that show the correct status of the sequencer.

From these explanations and information, it can be seen that `startedAt` value is a second value that should be used in the check for if a sequencer is down/up or correctly updated. The checks in `ChainlinkUtil::getPrice` will allow for sucessfull calls in an invalid round because reverts dont happen if `answer == 0` and `startedAt == 0` thus defeating the purpose of having a `sequencerFeed` check to assert the status of the `sequencerFeed` on L2 i.e if it is up/down/active or if its status is actually confirmed to be either.

There was also recently a [pull request](https://github.com/smartcontractkit/documentation/pull/1995) to update the [chainlink docs](https://docs.chain.link/data-feeds/l2-sequencer-feeds) sample code with this information, because this check should clearly be displayed there as well.

## Impact

Inadequate checks to confirm the correct status of the `sequencerUptimeFeed` in `ChainlinkUtil::getPrice` contract will cause `getPrice()` to not revert even when the sequencer uptime feed is not updated or is called in an invalid round.

## Tools Used

Manual Review

## Recommendations

```diff
ChainlinkUtil.sol

    function getPrice(
        IAggregatorV3 priceFeed,
        uint32 priceFeedHeartbeatSeconds,
        IAggregatorV3 sequencerUptimeFeed
    )
        internal
        view
        returns (UD60x18 price)
    {
        uint8 priceDecimals = priceFeed.decimals();
        // should revert if priceDecimals > 18
        if (priceDecimals > Constants.SYSTEM_DECIMALS) {
            revert Errors.InvalidOracleReturn();
        }

        if (address(sequencerUptimeFeed) != address(0)) {
            try sequencerUptimeFeed.latestRoundData() returns (
                uint80, int256 answer, uint256 startedAt, uint256, uint80
            ) {
                bool isSequencerUp = answer == 0;
                if (!isSequencerUp) {
                    revert Errors.OracleSequencerUptimeFeedIsDown(address(sequencerUptimeFeed));
                }

+               if (startedAt == 0){
+                   revert();
+               }

                uint256 timeSinceUp = block.timestamp - startedAt;
                if (timeSinceUp <= Constants.SEQUENCER_GRACE_PERIOD_TIME) {
                    revert Errors.GracePeriodNotOver();
                }
            } catch {
                revert Errors.InvalidSequencerUptimeFeedReturn();
            }
        }
        .
        .
        .
```


</details>
