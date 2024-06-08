# First Flight #16: Mafia Takedown - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings

  - ### [H-01.] Flaw in `Laundrette::quiTheGang` function: any member of the gang can kick another member. (Validated but downgraded to medium and selected for report) (#H-01)

  - ### [H-02.] $USDC address deployed in `Deployer.sol` is not the addresse of the native $USDC polygon mainnet.(#H-02)

  - ### [H-03.] Dependency Misconfiguration Vulnerability in configureDependencies Function. (Validated) (#H-03)

  - ### [H-04.] Funds from `moneyShelf` during migration won't be transferred to `moneyVault` (Validate but downgraded to medium) (#H-04)

  - ### [H-05.] Arbitrary `account` in `moneyShelf::usdc.transferFrom` (Validated) (#H-05)

- ## Medium Risk Findings

  - ### [M-01.] Laundrette policy is not authorized to call `Kernel:executeAction` function. (Validated but downgraded to low)(#M-01)

- ## Low Risk Findings

  - ### [L-01.] Panic: arithmetic underflow flaws in `Laundrette::withdrawMoney`.(#L-01)

# <a id='contest-summary'></a>Contest Summary

An undercover AMA agent (anti-mafia agency) discovered a protocol used by the Mafia. In several days, a raid will be conducted by the police and we need as much information as possible about this protocol to prevent any problems. But the AMA doesn’t have any web3 experts on their team. Hawks, they need your help! Find flaws in this protocol and send us your findings.

### Sponsor: CodeHawks First Flight #16

### Dates: May 23rd, 2024 (14:00) - May 30th, 2024 (14:00)

[See more contest details here](https://www.codehawks.com/contests/clwgiehgu00119zwn2xx92ay8)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 3
- Medium: 1
- Low: 1

# <a id='final-results'></a> Final Results

https://www.codehawks.com/report/clwgiehgu00119zwn2xx92ay8

Final rank: #6

# High Risk Findings

## <a id='H-01'> Flaw in `Laundrette::quiTheGang` function: any member of the gang can kick another member.</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/policies/Laundrette.sol#L88

## Summary

Flaw in `Laundrette::quiTheGang` function: any member of the gang can kick another member.

## Vulnerability Details

Unauthorized Role Revocation by Any Gang Member:

Any member with the gangmember role could potentially revoke the role of any other member. This could lead to malicious activities where one member revokes the roles of other members, causing disruptions within the system.

1.  Unauthorized Access:
    If a gang member can revoke the role of any other member, it undermines the principle of least privilege. Members could exercise power beyond their intended scope, leading to potential abuse.

2.  Operational Disruption:
    Revoking roles disrupts the normal operations of the contract. Members who have their roles revoked might be unable to perform essential functions, leading to operational inefficiencies or failure of certain functionalities.

3.  Potential for Internal Conflict:
    If gang members can revoke each other's roles without restrictions, it could lead to internal conflicts and mistrust among members. This could erode the cohesion and cooperation within the gang.

## Impact

Failing to add the require statement to the quitTheGang function can lead to unauthorized role revocations, operational disruptions, and governance challenges.

**Example Scenarios:**

- Scenario 1: Malicious Member:

  - A malicious gang member, Alice, decides to revoke the roles of all other members. Without the require statement, Alice can call quitTheGang for each member, effectively removing them from the gang and gaining control.

- Scenario 2: Compromised Account:

  - Bob, a gang member, has his account compromised. The attacker uses Bob's credentials to revoke the roles of other gang members, causing chaos and potentially taking over the gang’s operations.

<details>
  <summary>PoC</summary>

Add `BaseTest::addToGang` and `LaundretteTest::testKickGangMember`

### Unit test

```cpp
function addToGang(address account) internal {
      vm.prank(godFather);
      laundrette.addToTheGang(account);
  }
```

```cpp
    function testKickGangMember() public {
      joinGang(gangMemberOne);
      assertEq(kernel.hasRole(gangMemberOne, Role.wrap("gangmember")), true);

      addToGang(gangMemberTwo);
      assertEq(kernel.hasRole(gangMemberTwo, Role.wrap("gangmember")), true);

      vm.startPrank(gangMemberTwo);
      laundrette.quitTheGang(gangMemberOne);
      assertEq(kernel.hasRole(gangMemberOne, Role.wrap("gangmember")), false);
      vm.stopPrank();
  }

```

### Traces output

```

vesper3023@LAPTOP-7INTR2BC:~/First_Flight/2024-05-mafia-take-down$ forge test --match-test testKickGangMember
[⠊] Compiling...
[⠃] Compiling 7 files with 0.8.24
[⠊] Solc 0.8.24 finished in 2.78s
Compiler run successful!

Ran 1 test for test/Laundrette.t.sol:LaundretteTest
[PASS] testKickGangMember() (gas: 140904)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.29ms (1.02ms CPU time)

Ran 1 test suite in 13.00ms (5.29ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

```

</details>

Test pass, this means that any member can kick any address.

## Tools Used

Manual review.

Unit test (Foundry).

## Recommendations

By adding the require statement in `Laundrette::quitTheGang` to ensure that only the account itself or the godFather can revoke the role, you significantly reduce these vulnerabilities and improve the overall security and robustness of the contract.

**Mitigation Approach**

- Add `require` function in The `Laundrette::quitTheGang` to validate inputs and conditions before execution.
- `require` function must validate if the caller is the account itself or the GodFather

```diff
function quitTheGang(address account) external onlyRole("gangmember")
{
+     require(account == msg.sender || kernel.executor() == msg.sender, "Caller must be the account itself or the GodFather");
      kernel.revokeRole(Role.wrap("gangmember"), account);
}
```

[Error handling: Assert, Require, Revert and Exceptions](https://docs.soliditylang.org/en/v0.8.26/control-structures.html#index-8)

**Validation**

Methodology, we are using the same unit test showed in impact section.

Return of `forge test --match-test testKickGangMember` unit test

```
vesper3023@LAPTOP-7INTR2BC:~/First_Flight/2024-05-mafia-take-down$ forge test --match-test testKickGangMember
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/Laundrette.t.sol:LaundretteTest
[FAIL. Reason: revert: Caller must be the account itself or the GodFather] testKickGangMember() (gas: 156431)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 5.54ms (1.04ms CPU time)

Ran 1 test suite in 15.29ms (5.54ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/Laundrette.t.sol:LaundretteTest
[FAIL. Reason: revert: Caller must be the account itself or the GodFather] testKickGangMember() (gas: 156431)

Encountered a total of 1 failing tests, 0 tests succeeded
```

`Laundrette::quitTheGang` revert as expected by applying unauthorized access control.

## <a id='H-02'> $USDC address deployed in `Deployer.sol` is not the addresse of the native USDC polygon mainnet. </a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/script/Deployer.s.sol#L30

## Summary

USDC address deployed in `Deployer.sol` is not the addresse of the native USDC polygon mainnet.

## Vulnerability Details

The main vulnerability in your current deployment script stems from the fact that it uses a mock USDC token instead of the native USDC token on the Polygon mainnet.

- Mock Token Usage: The HelperConfig class points to a mock USDC token. While this is useful for testing, it is not suitable for production. The mock token does not hold any real value and cannot interact with other smart contracts or services that expect the real USDC token.

## Impact

The impact of using a mock USDC token instead of the native USDC token on the Polygon mainnet includes:

1. Loss of Funds: If users or other protocols mistakenly interact with your contract under the assumption that it uses real USDC, they could lose funds. For example, users might deposit real USDC expecting it to be used within your protocol, only to find that it is converted to or treated as a valueless mock token.

2. Operational Disruptions: The need to eventually switch from a mock token to the native USDC token will require significant operational changes. This could involve pausing the protocol, migrating user balances, and ensuring all components correctly interact with the real USDC. This process can be complex, error-prone, and disruptive to users.

## Tools Used

Manual review

## Recommendations

Modify the deploy script to ensure that your contracts interact with the native USDC on the Polygon mainnet. Here's a quick breakdown:

1. **Defined posUsdc Address:**

   You must correctly set the native Usdc address to 0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359, which is the address for the native USDC on Polygon mainnet.

2. **Update usdc Initialization:**

   The usdc variable must be initialized with the native USDC address within the deploy function.

3. **Integration with MoneyShelf:**

   The MoneyShelf contract must be initialized with the usdc variable, ensuring it uses the native USDC.

<details>
  <summary>Mitigation</summary>

```diff
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import { Script, console2 } from "lib/forge-std/src/Script.sol";
import { IERC20 } from "lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import { HelperConfig } from "script/HelperConfig.s.sol";

import { Kernel, Actions, Role } from "src/Kernel.sol";
import { CrimeMoney } from "src/CrimeMoney.sol";

import { WeaponShelf } from "src/modules/WeaponShelf.sol";
import { MoneyShelf } from "src/modules/MoneyShelf.sol";

import { Laundrette } from "src/policies/Laundrette.sol";

contract Deployer is Script {
    address public godFather;
+   address public posUsdc = 0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359;
+   IERC20 public usdc;

    function run() public {
        vm.startBroadcast();
        deploy();
        vm.stopBroadcast();
    }

    function deploy() public returns (Kernel, IERC20, CrimeMoney, WeaponShelf, MoneyShelf, Laundrette) {
        godFather = msg.sender;

-        // Deploy USDC mock
-        HelperConfig helperConfig = new HelperConfig();
-        IERC20 usdc = IERC20(helperConfig.getActiveNetworkConfig().usdc);
+       usdc = IERC20(posUsdc);

        Kernel kernel = new Kernel();
        CrimeMoney crimeMoney = new CrimeMoney(kernel);

        WeaponShelf weaponShelf = new WeaponShelf(kernel);
        MoneyShelf moneyShelf = new MoneyShelf(kernel, usdc, crimeMoney);
        Laundrette laundrette = new Laundrette(kernel);

        kernel.grantRole(Role.wrap("moneyshelf"), address(moneyShelf));

        kernel.executeAction(Actions.InstallModule, address(weaponShelf));
        kernel.executeAction(Actions.InstallModule, address(moneyShelf));
        kernel.executeAction(Actions.ActivatePolicy, address(laundrette));

        kernel.executeAction(Actions.ChangeAdmin, address(laundrette));
        kernel.executeAction(Actions.ChangeExecutor, godFather);

        return (kernel, usdc, crimeMoney, weaponShelf, moneyShelf, laundrette);
    }
}

```

</details>

## <a id='H-03'> Dependency Misconfiguration Vulnerability in configureDependencies Function.</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/policies/Laundrette.sol#L26

## Summary

Dependency Misconfiguration Vulnerability in configureDependencies Function.

## Vulnerability Details

1. Incorrect Dependency Recording:

By assigning `dependencies[0]` twice, the function overwrites the first dependency ("MONEY") with the second one ("WEAPN"). Consequently, the dependencies array will contain two references to the "WEAPN" keycode and none to the "MONEY" keycode.

2. Missing Dependency:

The "MONEY" dependency is not recorded, which means any functionality dependent on "MONEY" will not recognize this dependency and may not function correctly.

3. Inconsistent State:

The system expects the dependencies array to correctly represent all dependencies. An incorrect array can lead to an inconsistent state where the modules and policies may not work as intended.

## Impact

1. Functionality Issues:

The moneyShelf module will not be recognized as a dependency. Any policy or module that relies on moneyShelf will fail to function correctly because it won't be able to establish the necessary connections or permissions.

2. Security Risks:

The incorrect dependency registration can lead to security vulnerabilities if critical checks or balances depend on the moneyShelf module. For example, if moneyShelf handles financial transactions or holds funds, its absence from the dependency list might bypass important security checks.

3. Policy Activation Issues:

During the policy activation process, the system records dependencies. If these dependencies are incorrect, the policy may not activate properly, leading to further issues down the line when the policy tries to interact with its expected modules.

<details>
  <summary>Example scenario</summary>

Consider a scenario where the `Laundrette` policy needs to interact with both `moneyShelf` and `weaponShelf`. If `configureDependencies` only records `weaponShelf` as a dependency twice, the following issues can arise:

- Interactions with `moneyShelf` Fail: Any function within `Laundrette` that needs to interact with `moneyShelf` will fail because the dependency is not registered.

- Incorrect Policy Permissions: Permissions required to interact with `moneyShelf` might not be granted, leading to unauthorized access errors or transaction failures.

- Module Dependent Index Inconsistency: The system maintains an index of which policies depend on which modules. Incorrectly recording dependencies can cause this index to be inaccurate, leading to potential issues in module upgrades or policy deactivations.

</details>

## Tools Used

Manual review

## Recommendations

To avoid these issues, ensure that each dependency is correctly assigned to a unique index in the `dependencies` array:

```diff
function configureDependencies() external override onlyKernel returns (Keycode[] memory dependencies) {
    dependencies = new Keycode ;

    dependencies[0] = toKeycode("MONEY");
    moneyShelf = MoneyShelf(getModuleAddress(toKeycode("MONEY")));

-   dependencies[0] = toKeycode("WEAPN");
+   dependencies[1] = toKeycode("WEAPN");
    weaponShelf = WeaponShelf(getModuleAddress(toKeycode("WEAPN")));
}
```

## <a id='H-04'> Funds from `moneyShelf` during migration won't be transferred to `moneyVault` </a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/script/EmergencyMigration.s.sol#L28

## Summary

Funds from `moneyShelf` during migration won't be transferred to `moneyVault`

## Vulnerability Details

**Funds Locked in the Old Contract:**

- If funds are not transferred from the old contract (moneyShelf) to the new MoneyVault, they might remain locked in the old contract. This could render the funds inaccessible or difficult to retrieve.

**Scenario:**

- During an AMA-led dismantling operation, mafia funds can be confiscated in their entirety.

## Impact

**Financial Loss:**

- The most immediate and obvious impact is the potential financial loss if funds remain locked in the old contract or are improperly transferred.

## Tools Used

Manual review

## Recommendations

**Automated Fund Transfer Implementation**

1. Modify the MoneyVault Contract

- Add in MoneyVault a function to receive the funds.

<details>
  <summary>MoneyVault refactorisation</summary>

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { CrimeMoney } from "src/CrimeMoney.sol";
import { Kernel, Keycode } from "src/Kernel.sol";
import { Shelf } from "src/modules/Shelf.sol";

// Emergency money vault to store USDC and CrimeMoney
contract MoneyVault is Shelf {
    IERC20 private usdc;
    CrimeMoney private crimeMoney;

    constructor(Kernel kernel_, IERC20 _usdc, CrimeMoney _crimeMoney) Shelf(kernel_) {
        usdc = _usdc;
        crimeMoney = _crimeMoney;
    }

    function KEYCODE() public pure override returns (Keycode) {
        return Keycode.wrap("MONEY");
    }

    function depositUSDC(address, address, uint256) external pure {
        revert("MoneyVault: depositUSDC is disabled");
    }

    function withdrawUSDC(address account, address to, uint256 amount) external {
        require(to == kernel.executor(), "MoneyVault: only GodFather can receive USDC");
        withdraw(account, amount);
        crimeMoney.burn(account, amount);
        usdc.transfer(to, amount);
    }

+    // New function to receive funds during migration
+    function receiveFunds(uint256 amount) external {
+        usdc.transferFrom(msg.sender, address(this), amount);
+    }
}

```

</details>

2. Modify the migrate Function

- Update the migrate function to automate the transfer of funds from the moneyShelf contract to the new MoneyVault contract.

<details>
  <summary>migrate refactorisation</summary>

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import { Kernel } from "src/Kernel.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { CrimeMoney } from "src/CrimeMoney.sol";
import { MoneyVault } from "path/to/MoneyVault.sol";

contract MafiTakeDown {
    function migrate(Kernel kernel, IERC20 usdc, CrimeMoney crimeMoney, address moneyShelf) public returns (MoneyVault) {
        vm.startBroadcast(kernel.executor());

        // Create new MoneyVault
        MoneyVault moneyVault = new MoneyVault(kernel, usdc, crimeMoney);

        // Execute upgrade to the new MoneyVault
        kernel.executeAction(Actions.UpgradeModule, address(moneyVault));

+        // Transfer funds from moneyShelf to the new MoneyVault
+        uint256 balance = usdc.balanceOf(moneyShelf);
+        approveMigration(address(moneyVault), balance);
+        require(usdc.transferFrom(moneyShelf, address(moneyVault), balance), "Transfer failed");

        vm.stopBroadcast();

        return moneyVault;
    }
}

```

</details>

3. Adding approval in `moneyShelf`

<details>
  <summary>moneyShelf refactorisation</summary>

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { CrimeMoney } from "src/CrimeMoney.sol";

import { Kernel, Keycode } from "src/Kernel.sol";

import { Shelf } from "src/modules/Shelf.sol";

contract MoneyShelf is Shelf {
    IERC20 private usdc;
    CrimeMoney private crimeMoney;

    constructor(Kernel kernel_, IERC20 _usdc, CrimeMoney _crimeMoney) Shelf(kernel_) {
        usdc = _usdc;
        crimeMoney = _crimeMoney;
    }

    function KEYCODE() public pure override returns (Keycode) {
        return Keycode.wrap("MONEY");
    }

    // "permissioned" already used by Shelf functions
    function depositUSDC(address account, address to, uint256 amount) external {
        deposit(to, amount);
        usdc.transferFrom(account, address(this), amount);
        crimeMoney.mint(to, amount);
    }

    function withdrawUSDC(address account, address to, uint256 amount) external {
        withdraw(account, amount);
        crimeMoney.burn(account, amount);
        usdc.transfer(to, amount);
    }

+    function approveMigration(address migrator, uint256 amount) external {
+        usdc.approve(migrator, amount);
+    }
}

```

</details>

## <a id='H-05'> Arbitrary `account` in `moneyShelf::usdc.transferFrom` </a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/603ea5aa1b17c3c399502f5d8ba277396b9c70bd/src/modules/MoneyShelf.sol#L27

## Summary

Arbitrary `account` in `moneyShelf::usdc.transferFrom`w

## Vulnerability Details

if attacked approves this contract to spend his/her USDC tokens. Attacker can call a and specify attacked's address as the from/account parameter in transferFrom, allowing him/her to transfer attacked's tokens to moneyShelf contract.

## Impact

Attacked risks having all its USDC drained.

## Tools Used

Manual review

## Recommendations

Use msg.sender as from/account in transferFrom.

```diff
// "permissioned" already used by Shelf functions
-    function depositUSDC(address account, address to, uint256 amount) external {
+    function depositUSDC(address to, uint256 amount) external {
        deposit(to, amount);
-       usdc.transferFrom(account, address(this), amount);
+       usdc.transferFrom(msg.sender, address(this), amount);
        crimeMoney.mint(to, amount);
    }
```

# Medium Risk Findings

## <a id='M-01'> Laundrette policy is not authorized to call `Kernel::executeAction` function.</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/policies/Laundrette.sol#L92

## Summary

Laundrette policy is not authorized to call `Kernel::executeAction` function.

## Vulnerability Details

The current admin of gang policy is Laundrette. As admin Laundrette can grant or revoke roles. The purpose of `Laundrette::retrieveAdmin` is to change the current admin of gang policy. For any reason the god father must be able to change the current admin!

One concrete example : If he wants to prevent gang members to leave the gang.

The implementation of `Laundrette::retrieveAdmin` doesn't allow the God Father to change the current admin, because only `executor` can call `kernel::executeAction` and the `executor` is `God Father`.

**Laundrette.sol**

```cpp
    function retrieveAdmin() external {
        kernel.executeAction(Actions.ChangeAdmin, kernel.executor());
    }
```

**Kernel.sol**

```cpp
    // ######################## ~ MODIFIERS ~ ########################

    // Role reserved for governor or any executing address
    modifier onlyExecutor() {
        if (msg.sender != executor) revert Kernel_OnlyExecutor(msg.sender);
        _;
    }
```

```cpp
    // ######################## ~ KERNEL INTERFACE ~ ########################

    function executeAction(Actions action_, address target_) external onlyExecutor {
        if (action_ == Actions.InstallModule) {
            ensureContract(target_);
            ensureValidKeycode(Module(target_).KEYCODE());
            _installModule(Module(target_));
        } else if (action_ == Actions.UpgradeModule) {
            ensureContract(target_);
            ensureValidKeycode(Module(target_).KEYCODE());
            _upgradeModule(Module(target_));
        } else if (action_ == Actions.ActivatePolicy) {
            ensureContract(target_);
            _activatePolicy(Policy(target_));
        } else if (action_ == Actions.DeactivatePolicy) {
            ensureContract(target_);
            _deactivatePolicy(Policy(target_));
        } else if (action_ == Actions.MigrateKernel) {
            ensureContract(target_);
            _migrateKernel(Kernel(target_));
        } else if (action_ == Actions.ChangeExecutor) {
            executor = target_;
        } else if (action_ == Actions.ChangeAdmin) {
            admin = target_;
        }

        emit ActionExecuted(action_, target_);
    }
```

## Impact

The god father won't be able to change the current admin directly from the policy

## Tools Used

Manual review

Foundry unit test

## Recommendations

Here’s a revised implementation that should work correctly:

Laundrette.sol: Modify the retrieveAdmin function to check if the caller is the executor and then proceed with the action.

```cpp
// Laundrette.sol
pragma solidity ^0.8.19;

interface IKernel {
    function executeAction(Actions action, address target) external;
    function executor() external view returns (address);
}

enum Actions {
    InstallModule,
    UpgradeModule,
    ActivatePolicy,
    DeactivatePolicy,
    MigrateKernel,
    ChangeExecutor,
    ChangeAdmin
}

contract Laundrette {
    IKernel public kernel;

    constructor(address _kernel) {
        kernel = IKernel(_kernel);
    }

    modifier onlyExecutor() {
        require(msg.sender == kernel.executor(), "Laundrette: Only executor can call this function");
        _;
    }

    function retrieveAdmin() external onlyExecutor {
        kernel.executeAction(Actions.ChangeAdmin, msg.sender);
    }
}
```

# Low Risk Findings

## <a id='L-01'> Panic: arithmetic underflow flaws in `Laundrette::withdrawMoney`. </a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/modules/Shelf.sol#L17

## Summary

Panic: arithmetic underflow flaws in `Laundrette::withdrawMoney`.

## Vulnerability Details

1. Function Call Sequence:

   - The `withdrawMoney` function calls `withdrawUSDC`.
   - `withdrawUSDC` calls `withdraw`, `crimeMoney.burn`, and `usdc.transfer`.
   - The `withdraw` function performs the arithmetic operation `bank[account] -= amount`.

2. Potential Underflow in withdraw Function:

   - The `withdraw` function directly subtracts the `amount` from `bank[account]`.
   - If `amount` is greater than `bank[account]`, an underflow occurs, causing the resulting balance to wrap around to a very large number (`2^256 - (amount - bank[account])`).

## Impact

The impact in your specific case, given the protections provided by Solidity 0.8.0 and later, is minimal. The built-in checks effectively prevent the underflow from causing unauthorized actions or corrupting the contract state. While the concern for underflows is valid in general.

<details>
  <summary>PoC</summary>

### Stateless fuzzing test

```cpp
    function test_WithdrawUSDCWithoutCrimeMoney(uint256 depositAmount, uint256 withdrawAmount) public payable {
        uint256 initialUserBalance = 100e6;
        console.log("depositAmount:", depositAmount);
        console.log("withdrawAmount:", withdrawAmount);

        // Ensure depositAmount starts at 2 and withdrawAmount starts at 1
        vm.assume(depositAmount >= 100);
        vm.assume(withdrawAmount >= 100);

        joinGang(gangMemberOne);

        vm.prank(godFather);
        usdc.transfer(gangMemberOne, 100e6);

        vm.startPrank(gangMemberOne);
        // Only attempt deposit if the user has enough balance
        if (depositAmount <= initialUserBalance) {
            usdc.approve(address(moneyShelf), depositAmount);
            laundrette.depositTheCrimeMoneyInATM(gangMemberOne, gangMemberOne, depositAmount);

            if (withdrawAmount <= depositAmount) {
                laundrette.withdrawMoney(gangMemberOne, gangMemberOne, withdrawAmount);

                assertEq(
                    crimeMoney.balanceOf(gangMemberOne),
                    depositAmount - withdrawAmount,
                    "Balance after withdrawal should match expected value"
                );
            } else {
                // Expect a revert due to insufficient balance
                vm.expectRevert("Insufficient balance");
                laundrette.withdrawMoney(gangMemberOne, gangMemberOne, withdrawAmount);
            }
        }
        vm.stopPrank();
    }
```

### Traces output

```
vesper3023@LAPTOP-7INTR2BC:~/First_Flight/2024-05-mafia-take-down$ forge test --match-test test_WithdrawUSDCWithoutCrimeMoney -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/Laundrette.t.sol:LaundretteTest
[FAIL. Reason: Error != expected error:  != Insufficient balance; counterexample: calldata=0x788f9c95000000000000000000000000000000000000000000000000000000000000080c00000000000000000000000000000000000000000000000000000000000018ab args=[2060, 6315]] test_WithdrawUSDCWithoutCrimeMoney(uint256,uint256) (runs: 4, μ: 152390, ~: 152390)
Logs:
  ⚠️ You have deployed a mock conract!
  Make sure this was intentional
  depositAmount: 2060
  withdrawAmount: 6315

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 10.94ms (6.21ms CPU time)

Ran 1 test suite in 13.58ms (10.94ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/Laundrette.t.sol:LaundretteTest
[FAIL. Reason: Error != expected error:  != Insufficient balance; counterexample: calldata=0x788f9c95000000000000000000000000000000000000000000000000000000000000080c00000000000000000000000000000000000000000000000000000000000018ab args=[2060, 6315]] test_WithdrawUSDCWithoutCrimeMoney(uint256,uint256) (runs: 4, μ: 152390, ~: 152390)

Encountered a total of 1 failing tests, 0 tests succeeded

```

</details>

## Tools Used

Manual review
Stateless fuzz testing

## Recommendations

Even though the immediate risk is mitigated, it’s good practice to include explicit checks in your smart contracts to handle these conditions gracefully and provide clear error messages. This can enhance the robustness and user experience of your smart contracts:

```diff
function withdraw(address account, uint256 amount) public {
+    require(bank[account] >= amount, "Insufficient balance");
    bank[account] -= amount;
}

```
