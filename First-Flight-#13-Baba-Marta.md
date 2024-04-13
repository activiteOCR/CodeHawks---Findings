# First Flight #13: Baba marta - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. `MartenitsaToken:updateCountMartenitsaTokensOwner` user can update the count of martenitsaTokens without any restrictions](#H-01)
- ## Medium Risk Findings
  - ### [M-01. `MartenitsaMarketplace:collectReward` in a particular scenario amountRewards can't be correct because `_collectedRewards` mapping isn't reset if users sell at least 3 martenitsa token.](#M-01)
  - ### [M-02. `MartenitsaVoting:voteForMartenitsa` producer can vote for himself during a vote event.](#M-02)
- ## Low Risk Findings

  - ### [L-01. `MartenitsaToken::createMartenitsa` design @param is not properly checked, producer can create a martenitsa token with an empty string as design or with design without any meaning](#L-01)

  - ### [](#L-02)

  - ### [](#L-03)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: CodeHawks First Flight #13

### Dates: Apr 11th, 2024 (14:00) - Apr 18th, 2024 (14:00)

[See more contest details here](https://www.codehawks.com/contests/cluseb1bf0001s4tjl2rzajup)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1
- Medium:
- Low: 1

# High Risk Findings

## <a id='H-01'>`MartenitsaToken:updateCountMartenitsaTokensOwner` user can update the count of martenitsaTokens without any restrictions</a>H-01.

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaToken.sol#L62

## Summary

in `MartenitsaToken:updateCountMartenitsaTokensOwner` users can update the count of martenitsaTokens without any restrictions. Then they will be able to collect as many HealthTokens as they want.

## Vulnerability Details

By updating the count of martenitsaTokens without any restriction users will be able to mint HealthToken indefinitely.

1. Update count of martenitsaTokens with `MartenitsaToken:updateCountMartenitsaTokensOwner`
2. Collect the number of HealthTokens in proportion to the number of (fake) tokens you have (1 HealthToken for 3 fake MartenitsaTokens) by calling `MartenitsaMarketplace:collectReward`.

In this test we will collect 3 HealthToken while we don't have any martinetsaToken.

```cpp
 function testVulnCollectReward() public {
        vm.startPrank(bob);
        martenitsaToken.updateCountMartenitsaTokensOwner(bob, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(bob, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(bob, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(bob, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(bob, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(bob, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(bob, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(bob, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(bob, "add");
        marketplace.collectReward();
        assert(healthToken.balanceOf(bob) == 3 * 10 ** 18);
        vm.stopPrank();
    }
```

## Impact

HealthToken can be drained and user can participate to an event without owning any martenitsaToken

## Tools Used

manuel review

## Recommendations

`MartenitsaToken:updateCountMartenitsaTokensOwner` must be only called by MartenitsaMarketplace contract using internal visibility specifier instead of external. Therefore you will have to refactor correctly MartenitsaMarketplace contract.

# Medium Risk Findings

## <a id='M-01'>`MartenitsaMarketplace:collectReward` in a particular scenario amountRewards can't be correct because `_collectedRewards` mapping isn't reset if users sell at least 3 martenitsa token.</a>M-01.

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaMarketplace.sol#L104

## Summary

`MartenitsaMarketplace:collectReward` in a particular scenario amountRewards can't be correct because `_collectedRewards` mapping isn't reset if users sell at least 3 martenitsa token during an event and rebuy 3 others later, at this time user won't be able to claim his new healthtoken.

## Vulnerability Details

If a user first buys 3 martenitsa tokens, claims his healthtoken, sells his 3 martenitsa tokens during an event and later buys 3 new martenitsa tokens, he will no longer be able to claim a new health token because in `MartenitsaMarketplace:collectReward` line 104 of MartenitsaMarketplace contract the `_collectedRewards` is subtracted from the `amountRewards` and as `mapping(address => uint256) private _collectedRewards;` mapping is not reset to zero after the first 3 tokens have been sold, the `amountRewards` will be equal to zero instead of 1.

## Impact

User won't receive his HealthToken despite having 3 new martenitsa tokens

## Tools Used

Manuel review

## Recommendations

Track the number of sales per user and decrement by 1 `mapping(address => uint256) private \_collectedRewards; every 3 sales of martenitsa token per user.

## <a id='M-02'>`MartenitsaVoting:voteForMartenitsa` producer can vote for himself during a vote event.</a>M-01.

### Relevant GitHub Links

## Summary

`MartenitsaVoting:voteForMartenitsa` producer can vote for himself during a vote event. After listing any producer is able to vote for his martenitsa token.

## Vulnerability Details

`voteForMartenitsa` function don't check if the caller is a producer as demonstrated in the test bellow just after listing his token chasy who is a producer is able to vote for his martenitsa token.

```cpp
function testProducerVoteForMartenitsa() public listMartenitsa {
        vm.prank(chasy);
        voting.voteForMartenitsa(0);
        assert(voting.hasVoted(chasy) == true);
        assert(voting.voteCounts(0) == 1);
    }
```

## Impact

Vote system can be unfair because producer can vote for its creations.

## Tools Used

Manuel review

## Recommendations

Check if caller is a producer if yes revert the transaction.

```cpp
/**
     * @notice Function to vote for martenitsa of the sale list.
     * @param tokenId The tokenId of the martenitsa.
     */
    function voteForMartenitsa(uint256 tokenId) external {
        require(!hasVoted[msg.sender], "You have already voted");
        require(!martenitsaToken.isProducer(msg.sender), "You are producer and not eligible for voting!");
        require(block.timestamp < startVoteTime + duration, "The voting is no longer active");
        list = _martenitsaMarketplace.getListing(tokenId);
        require(list.forSale, "You are unable to vote for this martenitsa");

        hasVoted[msg.sender] = true;
        voteCounts[tokenId] += 1;
        _tokenIds.push(tokenId);
    }
```

# Low Risk Findings

## <a id='L-01'> `MartenitsaToken::createMartenitsa` design @param is not properly checked, producer can create a martenitsa token with an empty string as design or with design without any meaning </a>L-01.

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaToken.sol#L37

## Summary

In `MartenitsaToken::createMartenitsa` design @param is not properly checked, so a producer can create a martenitsa token with a whitespace as design (It holds a specific ASCII value which is 32) or with a design without any meaning.

## Vulnerability Details

The require control structure (L37 of MartenitsaToken.sol) does not correctly control the "Design" input parameter.

```cpp
function testCreateMartenitsaCalledWithDesignEqualZero() public {
        vm.prank(jack);
        vm.expectRevert();
        martenitsaToken.createMartenitsa(" ");
    }
```

```cpp
require(bytes(design).length > 0, "Design cannot be empty");
```

## Impact

Martenitsa token can be created with an empty string as design or with a design without any meaning.

## Tools Used

Manual review

## Recommendations

Create a custom error based on your check DesignToBytes == 0 and DesignToBytes is checks against the hexadecimal values of common whitespace characters:
0x20 - Space
0x5f - Horizontal Tab

So whitespace and horzontal tab won't be accepted as design character but you can add more design rules in the if statement if you decide to authorize only some specific design.

```cpp
    // Custom errors
    error MartenitsaToken__DesignLengthIsEmpty();
    error MartenistsaToken__IsAWhitespace();

    /**
     * @notice Function to create a new martenitsa. Only producers can call the function.
     * @param design The type (bracelet, necklace, Pizho and Penda and other) of martenitsa.
     */
    function createMartenitsa(string memory design) external {
        require(isProducer[msg.sender], "You are not a producer!");
        bytes memory designToBytes = bytes(design);
        if (designToBytes.length == 0) {
            revert MartenitsaToken__DesignLengthIsEmpty(); // Consider an empty string as not only whitespace
        }
        for (uint256 i = 0; i < designToBytes.length; i++) {
            if (designToBytes[i] == 0x20 || designToBytes[i] == 0x5f) {
                revert MartenistsaToken__IsAWhitespace();
            }
        }
        uint256 tokenId = _nextTokenId++;
        tokenDesigns[tokenId] = design;
        countMartenitsaTokensOwner[msg.sender] += 1;

        emit Created(msg.sender, tokenId, design);

        _safeMint(msg.sender, tokenId);
    }
```

## <a id='L-02'> </a>L-02.

### Relevant GitHub Links

## Summary

## Vulnerability Details

## Impact

## Tools Used

## Recommendations

## <a id='L-03'> </a>L-03.

### Relevant GitHub Links

## Summary

## Vulnerability Details

## Impact

## Tools Used

## Recommendations
