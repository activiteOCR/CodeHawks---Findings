# First Flight #12: Kitty connect - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [](#H-01)
- ## Medium Risk Findings
  - ### [M-01. ](#M-01)
- ## Low Risk Findings

  - ### [L-01. `KittyConnect::addShop` Functions that iterate over the array can become vectors for denial-of-service (DoS) attacks if they're not properly protected.](#L-01)

  - ### [L-02. Use onlyOwner for Simplicity and Security instead of custom onlyKittyConnectOwner modifier in `KittyConnect::addShop`.](#L-02)

  - ### [L-03. In `KittyConnect::addShop` duplicates insertion in array `address[] private s_kittyShops;`. ](#L-03)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: CodeHawks First Flight #12

### Dates: Mar 28th, 2024 (13:00) - Apr 4th, 2024 (14:00)

[See more contest details here](https://www.codehawks.com/contests/clu7ddcsa000fcc387vjv6rpt)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High:
- Medium:
- Low:

# High Risk Findings

## <a id='H-01'></a>H-01.

### Relevant GitHub Links

## Summary

## Vulnerability Details

## Impact

## Tools Used

## Recommendations

# Medium Risk Findings

## <a id='M-01'></a>M-01.

### Relevant GitHub Links

## Summary

## Vulnerability Details

## Impact

## Tools Used

## Recommendations

# Low Risk Findings

## <a id='L-01'> `KittyConnect::addShop` Functions that iterate over the array can become vectors for denial-of-service (DoS) attacks if they're not properly protected. </a>L-01.

### Relevant GitHub Links

https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L77

## Summary

To prevent the `s_kittyShops` array from growing indefinitely in KittyConnect smart contract, you can set limits on its size

## Vulnerability Details

Functions that iterate over the array can become vectors for denial-of-service (DoS) attacks if they're not properly protected.

## Impact

An attacker could, in theory, add a large number of addresses to the array, making iterations over it consume excessive amounts of gas, thus preventing regular contract operation or causing functions to fail due to out-of-gas errors

## Tools Used

Manual review

## Recommendations

Define a maximum number of entries allowed in the s_kittyShops array and enforce this limit within the function that adds new shop partners.

```cpp
uint256 private constant MAX_SHOP_PARTNERS = 100; // Example limit

function addShop(address shopAddress) external onlyKittyConnectOwner {
    require(s_kittyShops.length < MAX_SHOP_PARTNERS, "KittyConnect__MaxShopsReached");
    s_isKittyShop[shopAddress] = true;
    s_kittyShops.push(shopAddress);
    emit ShopPartnerAdded(shopAddress);
}
```

## <a id='L-02'> Use onlyOwner for Simplicity and Security instead of custom onlyKittyConnectOwner modifier in `KittyConnect::addShop` </a>L-02.

### Relevant GitHub Links

https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L77

## Summary

The onlyOwner() modifier provided by OpenZeppelin's Ownable contract is a widely recognized and trusted method for restricting access to certain functions within your smart contract to the contract's owner only. It's a part of the OpenZeppelin Contracts library, which is known for its security and adherence to best practices in smart contract development.

## Vulnerability Details

`onlyKittyConnectOwner` modifier does not add any additional logic beyond what Ownable provides, it's generally recommended to use onlyOwner() for its simplicity, security, and the benefits of adhering to a standard.

## Impact

Implementing custom access control logic introduces the potential for security vulnerabilities, especially if the custom implementation is not thoroughly tested and audited.

## Tools Used

Manual review

## Recommendations

**_Upgradability and Maintainability:_** Choosing standard patterns and well-known libraries like OpenZeppelin can make your contract easier to upgrade and maintain. If you use custom modifiers, document their behavior thoroughly to aid future development and auditing efforts.

Import Ownable.sol from openzeppelin library

```cpp
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
}
```

Add inheritance to the contract

```cpp
contract KittyConnect is ERC721, Ownable{...}
}
```

Refactor `KittyConnect::addShop()`
Add inheritance to the contract

```cpp
/**
     * @notice Allows the owner of the protocol to add a new shop partner
     * @param shopAddress The address of new shop partner
     */
    function addShop(address shopAddress) external onlyOwner {
        s_isKittyShop[shopAddress] = true;
        s_kittyShops.push(shopAddress);
        emit ShopPartnerAdded(shopAddress);
    }
}
```

## <a id='L-03'> In `KittyConnect::addShop` duplicates insertion in array `address[] private s_kittyShops;`. </a>L-03.

### Relevant GitHub Links

https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L77

## Summary

## Vulnerability Details

## Impact

## Tools Used

Manual review

## Recommendations

Avoiding duplicates when adding new shop partners, make sure to check the s_isKittyShop mapping before adding a new address to both the mapping and the array. This prevents duplicates from being added in the first place.

```cpp
function addShop(address shopAddress) external onlyKittyConnectOwner {
    require(!s_isKittyShop[shopAddress], "KittyConnect__AlreadyAShopPartner");
    s_isKittyShop[shopAddress] = true;
    s_kittyShops.push(shopAddress);
    emit ShopPartnerAdded(shopAddress);
}

```
