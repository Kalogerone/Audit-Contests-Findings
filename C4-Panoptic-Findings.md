# [M-1] - `CREATE2` address collision against a Panoptic Pool will allow complete draining of the pool

(NOTE: This report is very highly inspired from [this past valid report.](https://github.com/sherlock-audit/2023-12-arcadia-judging/issues/59) Necessary changes have been made to suit the Panoptic Protocol.)

## Vulnerability Detailed

The attack consists of two parts: Finding a collision, and actually draining the lending pool. We describe both here:

#### PoC: Finding a collision

Note that in `PanopticFactory::deployNewPool`, `CREATE2` salt is user-supplied which is then passed to `Clones::cloneDeterministic`:

```javascript
    function deployNewPool(address token0, address token1, uint24 fee, bytes32 salt)
        external
        returns (PanopticPool newPoolContract)
    {
        // sort the tokens, if necessary:
        (token0, token1) = token0 < token1 ? (token0, token1) : (token1, token0);

        .
        .
        .

        // This creates a new Panoptic Pool (proxy to the PanopticPool implementation)
        // Users can specify a salt, the aim is to incentivize the mining of addresses with leading zeros
@>      newPoolContract = PanopticPool(POOL_REFERENCE.cloneDeterministic(salt));

        .
        .
        .
    }
```

```javascript
    function cloneDeterministic(address implementation, bytes32 salt) internal returns (address instance) {
        /// @solidity memory-safe-assembly
        assembly {
            // Cleans the upper 96 bits of the `implementation` word, then packs the first 3 bytes
            // of the `implementation` address with the bytecode before the address.
            mstore(0x00, or(shr(0xe8, shl(0x60, implementation)), 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000))
            // Packs the remaining 17 bytes of `implementation` with the bytecode after the address.
            mstore(0x20, or(shl(0x78, implementation), 0x5af43d82803e903d91602b57fd5bf3))
@>          instance := create2(0, 0x09, 0x37, salt)
        }
        require(instance != address(0), "ERC1167: create2 failed");
    }
```

The address collision an attacker will need to find are:

- One undeployed Panoptic Pool address (1).
- Arbitrary attacker-controlled wallet contract (2).

Both sets of addresses can be brute-force searched because:

- As shown above, `salt` is a user-supplied parameter. By brute-forcing many `salt` values, we have obtained many different (undeployed) wallet accounts for (1). The user can know the address of the Panoptic Pool before deploying it, since as shown in the above code snippet, the result is deterministic.
- (2) can be searched the same way. The contract just has to be deployed using `CREATE2`, and the `salt` is in the attacker's control by definition.

An attacker can find any single address collision between (1) and (2) with high probability of success using the following meet-in-the-middle technique, a classic brute-force-based attack in cryptography:

- Brute-force a sufficient number of values of salt ($2^{80}$), pre-compute the resulting account addresses, and efficiently store them e.g. in a Bloom filter data structure.
- Brute-force contract pre-computation to find a collision with any address within the stored set in step 1.

The feasibility, as well as detailed technique and hardware requirements of finding a collision, are sufficiently described in multiple references:

- [1: A past issue on Sherlock describing this attack.](https://github.com/sherlock-audit/2023-07-kyber-swap-judging/issues/90)
- [2: EIP-3607, which rationale is this exact attack. The EIP is in final state.](https://eips.ethereum.org/EIPS/eip-3607)
- [3: A blog post discussing the cost (money and time) of this exact attack.](https://mystenlabs.com/blog/ambush-attacks-on-160bit-objectids-addresses)

The [hashrate of the BTC network](https://www.blockchain.com/explorer/charts/hash-rate) has reached $6.5 x 10^{20}$ hashes per second as of time of writing, taking only just 31 minutes to achieve $2^{80}$ hashes. A fraction of this computing power will still easily find a collision in a reasonably short timeline.

#### PoC: Draining the pool

Even given EIP-3607 which disables an EOA if a contract is already deployed on top, we show that it's still possible to drain the Panoptic Pool entirely given a contract collision.

Assuming the attacker has already found an address collision against an undeployed Panoptic Pool, let's say `0xCOLLIDED`. The steps for complete draining of the Panoptic Pool are as follow:

First tx:

- Deploy the attack contract onto address `0xCOLLIDED`.
- Set infinite allowance for {`0xCOLLIDED` ---> attacker wallet} for any token they want.
- Destroy the contract using `selfdestruct`.

Post Dencun hardfork, [`selfdestruct` is still possible if the contract was created in the same transaction.](https://eips.ethereum.org/EIPS/eip-6780) The only catch is that all 3 of these steps must be done in one tx.

The attacker now has complete control of any funds sent to `0xCOLLIDED`.

Second tx:

- Deploy the Panoptic Pool to `0xCOLLIDED`.
- Wait until the Panoptic Pool will hold as many tokens as you want and drain it.

The attacker has stolen all funds from the Panoptic Pool.

## Impact

Address collision can cause all tokens of a Panoptic Pool to be drain.

## Proof of Concept

While we cannot provide an actual hash collision due to infrastructural constraints, we are able to provide a coded PoC to prove the following two properties of the EVM that would enable this attack:

- A contract can be deployed on top of an address that already had a contract before.
- By deploying a contract and self-destruct in the same tx, we are able to set allowance for an address that has no bytecode.

Here is the PoC, as well as detailed steps to recreate it:

- Paste the following file onto Remix (or a developing environment of choice): [POC](https://gist.github.com/midori-fuse/087aa3248da114a0712757348fcce814)
- Deploy the contract `Test`.
- Run the function `Test.test()` with a salt of your choice, and record the returned address. The result will be:
    - `Test.getAllowance()` for that address will return exactly APPROVE_AMOUNT.
    - `Test.getCodeSize()` for that address will return exactly zero.
    - This proves the second property.
- Using the same salt in step 3, run Test.test() again. The tx will go through, and the result will be:
    - `Test.test()` returns the same address as with the first run.
    - `Test.getAllowance()` for that address will return twice of APPROVE_AMOUNT.
    - `Test.getCodeSize()` for that address will still return zero.
    - This proves the first property.

The provided PoC has been tested on Remix IDE, on the Remix VM - Mainnet fork environment, as well as testing locally on the Holesky testnet fork, which as of time of writing, has been upgraded with the Dencun hardfork.

## Tools Used

Manual Review, Remix IDE

## Recommended Mitigation Steps

- Don't allow the user to control the `salt` used.
- Consider also adding and encoding `block.timestamp` and `block.number` combined with the user's `salt`. Then the attacker, after they successfully found a hash collision, already has to execute the attack at a fixed block and probably conspire with the sequencer to ensure that also the time is fixed.