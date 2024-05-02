# First Flight #14: Air Dropper - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings

  - ### [H-01.] `s_zkSyncUSDC` public address variable used in the `run()` function of `Deploy.s.sol` to deploy is not the address of zksync USDC (#H-01)
  - ### [H-02.] In `MerkleAirdrop.sol:claim` a Winner can drain the total amount of the airdrop.(#H-02)
  - ### [H-03.] In `Deploy.s.sol` the `s_merkleRoot` variable don't reflect the amount that must be airdropped ($100).(#H-03)
  - ### [H-04.] The owner's address of `MerkleAirdrop.sol` contract is `Deploy.s.sol`, so any Ether sent to it would cause the transfer to revert without the implementation of receive function.(#H-04)
  - ### [H-05.] One of the addresses in the Merkle Tree is a contract specifically a `GnosisSafeProxy` contract.(#H-05)

- ## Medium Risk Findings

  - ### [M-01.](#M-01)

- ## Low Risk Findings
  - ### [L-01.] `Deploy.s.sol` is not approve as spender by the deployer, therefore no USDC won't be transferred to the `MerkleAirdrop.sol`.(#L-01)
  - ### [L-02.] The visibility of the `Deploy:deployMerkleDropper` function is public leading to multiple instances of the MerkleAirdrop contract.(#L-02)

# <a id='contest-summary'></a>Contest Summary

AirDropper is a gas optimized protocol built to assist with token distribution on the zkSync Era chain.

### Sponsor: CodeHawks First Flight #14

### Dates: Apr 25th, 2024 (14:00) - May 02nd, 2024 (14:00)

[See more contest details here](https://www.codehawks.com/contests/clvb821kr0001jzdbi6ggixb0)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High:
- Medium:
- Low:

# High Risk Findings

## <a id='H-01'> `s_zkSyncUSDC` public address variable used in the `run()` function of `Deploy.s.sol` to deploy is not the address of zksync USDC</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/script/Deploy.s.sol#L8C35-L8C77

## Summary

`s_zkSyncUSDC` public address variable used in the `run()` function of `Deploy.s.sol` to deploy is not the address of zksync USDC

## Vulnerability Details

in `MerkleAirDrop:claim` function if the account who claims his reward in spite of valid proof it won't receive any USDC.

```cpp
function claim(address account, uint256 amount, bytes32[] calldata merkleProof) external payable {
        if (msg.value != FEE) {
            revert MerkleAirdrop__InvalidFeeAmount();
        }
        bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(account, amount))));
        if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
            revert MerkleAirdrop__InvalidProof();
        }
        emit Claimed(account, amount);
        i_airdropToken.safeTransfer(account, amount);
    }
```

## Impact

In this configuration No USDC will be airdropped.

## Tools Used

Manual review

## Recommendations

Hardcode correctly the `s_zkSyncUSDC` variable, you can also add more test to be sure to address the correct contract.
Read the name and symbol of the contract to valid it.

## <a id='H-02'> In `MerkleAirdrop.sol:claim` a Winner can drain the total amount of the airdrop </a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/src/MerkleAirdrop.sol#L30

## Summary

A winner can drain the total amount of the airdrop by calling `MerkleAirdrop.sol:claim` as many times as necessary

## Vulnerability Details

No check is carried out to see whether the winner (account address selectionned) has already withdrawn his winnings

```cpp
    function testUsersCanDrain() public {
        uint256 startingBalance = token.balanceOf(collectorOne);
        vm.deal(collectorOne, 4e9);

        vm.startPrank(collectorOne);
        airdrop.claim{ value: airdrop.getFee() }(collectorOne, amountToCollect, proof);
        airdrop.claim{ value: airdrop.getFee() }(collectorOne, amountToCollect, proof);
        airdrop.claim{ value: airdrop.getFee() }(collectorOne, amountToCollect, proof);
        airdrop.claim{ value: airdrop.getFee() }(collectorOne, amountToCollect, proof);
        vm.stopPrank();

        uint256 endingBalance = token.balanceOf(collectorOne);
        assertEq(endingBalance - startingBalance, amountToSend);
    }
```

## Impact

Winners can drain the MerkleAirdrop contract

## Tools Used

Manual review

## Recommendations

Use a mapping to check if a winner (account address selectionned) has already withdrawn his winnings

```cpp
  mapping(address => bool) public hasClaimed;
```

In `MerkleAirdrop:claim` add these part

```cpp
  // Check if the reward has already been claimed
  if (hasClaimed[account]) {
    revert("MerkleAirdrop__RewardAlreadyClaimed");
  }

  // Set the reward as claimed
   hasClaimed[account] = true;
```

## <a id='H-03'> In `Deploy.s.sol` the `s_merkleRoot` variable don't reflect the amount that must be airdropped ($100).</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/script/Deploy.s.sol#L9
https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/src/MerkleAirdrop.sol#L39
https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/script/Deploy.s.sol#L11

## Summary

In `Deploy.s.sol` the `s_merkleRoot` variable don't reflect the amount that must be airdropped ($100).

## Vulnerability Details

The amount values used to create the leafs of the merkle tree are not aligned with the token distribution annouced for this airdropper. The hash value of `s_merkleRoot` in `Deploy.s.sol` corresponds to an amount equivalent to 25e18 USDC.

In `MerkleAirdropTest.t.sol` we can demonstrate this vulnerability by different approach, by inserting wrong merkle root with wrong proof and with wrong amount to collect.

```cpp
  bytes32 public GoodMerkleRoot = 0x3b2e22da63ae414086bec9c9da6b685f790c6fab200c7918f2879f08793d77bd;
  bytes32 public WrongMerkleRoot = 0xf69aaa25bd4dd10deb2ccd8235266f7cc815f6e9d539e9f4d47cae16e0c36a05;
  uint256 goodAmountToCollect = (25 * 1e6); // 25.000000
  uint256 wrongAmountToCollect = (25 * 1e18); // 25.000000000000000000
  uint256 goodAmountToSend = goodAmountToCollect * 4;
  uint256 wrongAmountToSend = wrongAmountToCollect * 4;
  address collectorOne = 0x20F41376c713072937eb02Be70ee1eD0D639966C;

  bytes32 goodProofOne = 0x32cee63464b09930b5c3f59f955c86694a4c640a03aa57e6f743d8a3ca5c8838;
  bytes32 goodProofTwo = 0x8ff683185668cbe035a18fccec4080d7a0331bb1bbc532324f40501de5e8ea5c;
  bytes32 wrongProofOne = 0x4fd31fee0e75780cd67704fbc43caee70fddcaa43631e2e1bc9fb233fada2394;
  bytes32 wrongProofTwo = 0xc88d18957ad6849229355580c1bde5de3ae3b78024db2e6c2a9ad674f7b59f84;
  bytes32[] goodProof = [goodProofOne, goodProofTwo];
  bytes32[] wrongProof = [wrongProofOne, wrongProofTwo];

  function testUsersCanClaim() public {
    uint256 startingBalance = token.balanceOf(collectorOne);
    vm.deal(collectorOne, 1e9);

    vm.startPrank(collectorOne);
    airdrop.claim{ value: 1e9 }(collectorOne, wrongAmountToCollect, wrongProof);
    vm.stopPrank();

    uint256 endingBalance = token.balanceOf(collectorOne);
    assertEq(endingBalance - startingBalance, wrongAmountToCollect);
  }

  function testRevertifwrongAmountToCollect() public {
    vm.deal(collectorOne, 1e9);

    vm.startPrank(collectorOne);
    vm.expectRevert(MerkleAirdrop.MerkleAirdrop__InvalidProof.selector);
    // Expect the specific revert reason
    airdrop.claim{ value: 1e9 }(collectorOne, goodAmountToCollect, wrongProof);
    vm.stopPrank();
  }
```

```
  Ran 1 test for test/MerkleAirdropTest.t.sol:MerkleAirdropTest
  [PASS] testUsersCanClaim() (gas: 33930)
  Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.44s (1.78s CPU time)

  Ran 1 test suite in 6.44s (6.44s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

```
  Ran 1 test for test/MerkleAirdropTest.t.sol:MerkleAirdropTest
  [PASS] testRevertifwrongAmountToCollect() (gas: 27206)
  Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.77s (728.31ms CPU time)

  Ran 1 test suite in 4.77s (4.77s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The first test pass because we are collecting the wrong amount (25e18) with the wrong merkleroot, the second test pass because we are trying to collect the good amount but this amount doesn't reflect the hash of the merkle root.

## Impact

Winners won't be airdropped because the `s_amountToAirdrop` which is equal to `4 * (25 * 1e6)` is not aligned with amount used to create the leafs of the merkle tree wich is `25e18` so the function `i_airdropToken.safeTransfer(account, amount);` in `MerkleAirdrop.sol` will revert because the contract doesn't have enough funds.

## Tools Used

Manual review

## Recommendations

Use the correct merkle root hash to be aligned with the token distibution announced.

```cpp
 bytes32 public s_merkleRoot = 0x3b2e22da63ae414086bec9c9da6b685f790c6fab200c7918f2879f08793d77bd;
```

## <a id='H-04'> The owner's address of `MerkleAirdrop.sol` contract is `Deploy.s.sol`, so any Ether sent to it would cause the transfer to revert without the implementation of receive function.</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/script/Deploy.s.sol#L14
https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/src/MerkleAirdrop.sol#L43

## Summary

The owner's address of `MerkleAirdrop.sol` contract is `Deploy.s.sol` any Ether sent to it would cause the transfer to revert without the implementation of receive function.

## Vulnerability Details

The low-lvl call in `MerkleAirdrop:claimFees` will revert because the owner account isn't capable of receiving Ether. The owner's address of `MerkleAirdrop.sol` is a contract `Deploy.s.sol` and this contract doesn't implement a function for handling plain Ether transfers.

## Impact

The owner won't be able to claim the FEES generated during the airdrop session.

## Tools Used

Manual review

## Recommendations

Add a receive function in `Deploy.s.sol` or transfer the ownership of `MerkleAirdrop.sol` to an EOA.

```cpp
  event Received(address caller, uint256 amount);
  // This function is triggered when the contract receives Ether without data
    receive() external payable {
        emit Received(msg.sender, msg.value);
    }
```

```cpp
  event OwnershipTransferred(address indexed oldOwner, address indexed newOwner);
  // Function to transfer ownership to a new address
  function transferOwnership(address newOwner) public onlyOwner {
      require(newOwner != address(0), "New owner cannot be the zero address");
      emit OwnershipTransferred(owner, newOwner);
      owner = newOwner;
  }
```

## <a id='H-05'> One of the addresses in the Merkle Tree is a contract specifically a `GnosisSafeProxy` contract</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/script/Deploy.s.sol#L9
https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/src/MerkleAirdrop.sol#L39

## Summary

One of the addresses in the Merkle Tree is a contract specifically a `GnosisSafeProxy` contract which uses a delegatecall to forward calls to a singleton (master copy).

## Vulnerability Details

The GnosisSafeProxy uses a delegatecall to forward calls to a singleton (master copy):

- State Manipulation: The behavior of the proxy will depend entirely on the state and logic of the master copy it points to. If the master copy is updated or points to a malicious contract, the behavior of the proxy in your airdrop process could lead to security vulnerabilities.
- Transaction Calls: The proxy can be used to interact with your contract in a high-level manner, possibly invoking other contracts or performing actions beyond simple token transfers. This includes invoking contracts that change state, manipulate token balances, or interact with other contracts in a harmful way.

Delegate Call Risks: The use of delegatecall within the proxy can lead to serious vulnerabilities, especially if the contract to which it delegates is not secure. This could potentially lead to the execution of unintended logic that can manipulate the proxy's state.

## Impact

This specific case highlights the fact that a malicious contract, if selected, can interact with and introduce a potential risk of draining or taking control of the contract.

## Tools Used

Manual review

## Recommendations

In the process of winners selection be sure to select only EOAs, in case or contract addresses are mandatory use at least reentrancy guard in `MerkleAirdrop:claim` and ensuring that they are properly configured and not susceptible to attacks that could impact the broader system. Ask for review before to select it.

```cpp
  import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
```

```cpp
  contract MerkleAirdrop is Ownable, ReentrancyGuard {
    // contract code
  }
```

```cpp
  function claim(address account, uint256 amount, bytes32[] calldata merkleProof) external payable nonReentrant {
    if (msg.value != FEE) {
        revert MerkleAirdrop__InvalidFeeAmount();
    }
    bytes32 leaf = keccak256(abi.encode(account, amount));
    if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
        revert MerkleAirdrop__InvalidProof();
    }
    emit Claimed(account, amount);
    i_airdropToken.safeTransfer(account, amount);
  }
```

# Medium Risk Findings

## <a id='M-01'> </a>

### Relevant GitHub Links

## Summary

## Vulnerability Details

## Impact

## Tools Used

## Recommendations

# Low Risk Findings

## <a id='L-01'> `Deploy.s.sol` is not approve as spender by the deployer, therefore no USDC won't be transferred to the `MerkleAirdrop.sol`. </a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/script/Deploy.s.sol#L18

## Summary

`Deploy.s.sol` is not approve as spender by the deployer, therefore no USDC won't be transferred to the `MerkleAirdrop.sol`.

## Vulnerability Details

In `Deploy.s.sol:run` after deploying the `MerkleAirdrop.sol` contract the owner want to transfer $100 USDC in order to aidrop the 4 winners, however at this step `Deploy.s.sol` contract can't transfer anything without the approval of deployer. Operation will fail and return `false`, contract doesn't have enough funds.

```cpp
function run() public {
        vm.startBroadcast();
        MerkleAirdrop airdrop = deployMerkleDropper(s_merkleRoot, IERC20(s_zkSyncUSDC));
        // Send USDC -> Merkle Air Dropper
        IERC20(0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4).transfer(address(airdrop), s_amountToAirdrop);
        vm.stopBroadcast();
    }
```

## Impact

Contract not deployed (with gas cost associated)

## Tools Used

Manual review

## Recommendations

Approve the amount equivalent to `s_amountToAirdrop` on `Deploy.s.sol`, check the balance of `Deploy.s.sol` before to deploy `MerkleAirdrop.sol` contract.

```cpp
    function approveUSDCSpending(address spender, uint256 amount) public {
        deployerKey = vm.envUint("PRIVATE_KEY");
        vm.startBroadcast(deployerKey);

        // USDC Contract Interface
        IERC20 usdc = IERC20(s_zkSyncUSDC);

        // Approving the spender to use the specified amount of USDC
        require(usdc.approve(spender, amount), "Approval failed");

        vm.stopBroadcast();
    }
```

## <a id='L-02'> The visibility of the `Deploy:deployMerkleDropper` function is public leading to multiple instances of the MerkleAirdrop contract.</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/script/Deploy.s.sol#L22

## Summary

The visibility of the `Deploy:deployMerkleDropper` function is public leading to multiple instances of the MerkleAirdrop contract.

## Vulnerability Details

As `Deploy:deployMerkleDropper` function has a public visibiliy, mutliple instance of `MerkleAirdrop` can be deployed

## Impact

Multiple instances of the `MerkleAirdrop` contract can result in confusion or errors, such as the wrong contract being used by users to claim airdrops or multiple contracts holding funds when only one was intended to be funded and used.

## Tools Used

Manual review

## Recommendations

Restricting Access: Modify the visibility of the deployMerkleDropper function to internal or private to be used within the Deploy contract itself or consider implementing access control mechanisms, such as requiring that the caller be the owner or have specific permissions (using modifiers such as onlyOwner if the contract inherits from Ownable).
