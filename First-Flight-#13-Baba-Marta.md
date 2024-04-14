# First Flight #13: Baba marta - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. `MartenitsaToken:updateCountMartenitsaTokensOwner` user can update the count of martenitsaTokens without any restrictions](#H-01)
  - ### [H-02. `MartenitsaVoting` time assignment of duration variable not correct.](#H-02)
  - ### [H-03. `MartenitsaVoting:announceWinner` don't manage if there is a tie at the end of the voting period.](#H-03)
- ## Medium Risk Findings
  - ### [M-01. `MartenitsaMarketplace:collectReward` in a particular scenario amountRewards can't be correct because `_collectedRewards` mapping isn't reset if users sell at least 3 martenitsa token.](#M-01)
  - ### [M-02. `MartenitsaVoting:voteForMartenitsa` producer can vote for himself during a vote event.](#M-02)
- ## Low Risk Findings
  - ### [L-01. `MartenitsaToken::createMartenitsa` design @param is not properly checked, producer can create a martenitsa token with an empty string as design or with design without any meaning](#L-01)
  - ### [L-02. `MartenitsaVoting:voteForMartenitsa` user can vote even startvoting is not started from the genesis block to the 86400 blocks.](#L-02)
- ### [L-03. `MartenitsaEvent:startEvent` `uint256 public eventEndTime` can lead to a unconsiderer revert (through built-in overflow protection) during the execution of startEvent function.](#L-03)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: CodeHawks First Flight #13

### Dates: Apr 11th, 2024 (14:00) - Apr 18th, 2024 (14:00)

[See more contest details here](https://www.codehawks.com/contests/cluseb1bf0001s4tjl2rzajup)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 3
- Medium: 2
- Low: 2

# High Risk Findings

## <a id='H-01'>`MartenitsaToken:updateCountMartenitsaTokensOwner` user can update the count of martenitsaTokens without any restrictions </a>

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

HealthToken can be easily minted and user can participate to an event without owning any martenitsaToken

## Tools Used

manuel review

## Recommendations

`MartenitsaToken:updateCountMartenitsaTokensOwner` must be only called by MartenitsaMarketplace contract using internal visibility specifier instead of external. Therefore you will have to refactor correctly MartenitsaMarketplace contract.

## <a id='H-02'>`MartenitsaVoting` time assignment of duration variable not correct. </a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaVoting.sol#L15

## Summary

`MartenitsaVoting` time assignment of duration variable not correct. in `MartenitsaVoting` developers have assigned the variable duration 1 days as value instead of the equivalent of 1 day in block time

## Vulnerability Details

In `MartenitsaVoting` developers have assigned the variable duration 1 days as value which is equivalent to 86,400 seconds (24 hours x 60 minutes x 60 seconds).

```cpp
 uint256 public duration = 1 days;
```

Duration variable is used in `MartenitsaVoting:voteForMartenitsa` and `MartenitsaVoting:announceWinner` in require control structure of these two functions

```cpp

    /**
     * @notice Function to vote for martenitsa of the sale list.
     * @param tokenId The tokenId of the martenitsa.
     */
    function voteForMartenitsa(uint256 tokenId) external {
        require(!hasVoted[msg.sender], "You have already voted");
        require(!martenitsaToken.isProducer(msg.sender), "You are producer and not eligible for voting!");
        console.log("startVoteTime: %d + duration: %d", startVoteTime, duration);
        require(startVoteTime != 0 && block.timestamp < startVoteTime + duration, "The voting is no longer active");
        list = _martenitsaMarketplace.getListing(tokenId);
        require(list.forSale, "You are unable to vote for this martenitsa");

        hasVoted[msg.sender] = true;
        voteCounts[tokenId] += 1;
        _tokenIds.push(tokenId);
    }

    /**
     * @notice Function to announce the winner of the voting. The winner receive 1 HealthToken.
     */
    function announceWinner() external onlyOwner {
        require(block.timestamp >= startVoteTime + duration, "The voting is active");

        uint256 winnerTokenId;
        uint256 maxVotes = 0;

        for (uint256 i = 0; i < _tokenIds.length; i++) {
            if (voteCounts[_tokenIds[i]] > maxVotes) {
                maxVotes = voteCounts[_tokenIds[i]];
                winnerTokenId = _tokenIds[i];
            }
        }

        list = _martenitsaMarketplace.getListing(winnerTokenId);
        _healthToken.distributeHealthToken(list.seller, 1);

        emit WinnerAnnounced(winnerTokenId, list.seller);
    }
```

The duration time of voting will not be 1 days but 14 days because currently in the Ethereum blockchain, the average time to mine a block is typically around 13-15 seconds.

Based on an average block time of 14 seconds:

- There are 86,400 seconds in a day (24 hours x 60 minutes x 60 seconds).
- Dividing 86,400 seconds by 14 seconds per block gives the approximate number of blocks mined in one day.

So Approximately 6,171 blocks are mined each day on the Ethereum blockchain, based on an average block time of 14 seconds

## Impact

The voting period will last 14 days instead of 1 day which will compromise the announcement of the winner.

## Tools Used

Manuel review

## Recommendations

`MartenitsaVoting` hardcode voting duration to be compliant with blocktimestamp calcul

```cpp
uint256 public duration = 6171;
```

Or use a function to calculate dynamically the blocktime average and refactor

```cpp
    /**
     * @notice Function to start the voting.
     */
    function startVoting() public onlyOwner {
        startBlock = block.number;
        startVoteTime = block.timestamp;
        while (startBlock == block.number) {} // wait for the next block
        duration = calculateAverageBlockTime(block.number);
        startVoteTime += 1;
        emit Voting(startVoteTime, duration);
    }

    /**
     * @notice Function to calculate the average blocktime
     */
    function calculateAverageBlockTime(uint256 endBlock) public view returns (uint256) {
        require(endBlock > startBlock, "End block must be greater than start block");
        uint256 endTime = block.timestamp; // Assume current block is the end block for simplicity
        uint256 elapsedTime = endTime - startVoteTime;
        uint256 blocksCount = endBlock - startBlock;
        return elapsedTime / blocksCount;
    }
```

## <a id='H-03'>`MartenitsaVoting:announceWinner` don't manage if there is a tie at the end of the voting period. </a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaVoting.sol#L57

## Summary

`MartenitsaVoting:announceWinner` don't manage if there is a tie at the end of the voting period, if there is a tie, the winner will be the one with the lowest tokenID

## Vulnerability Details

In `MartenitsaVoting:announceWinner` the for loop iterate on the length of `_tokenIDs` array and never check if MartenitsaTokenId mapping has an equality regarding the number of votes.

```cpp
    uint256 winnerTokenId;
    uint256 maxVotes = 0;

    for (uint256 i = 0; i < _tokenIds.length; i++) {
        if (voteCounts[_tokenIds[i]] > maxVotes) {
            maxVotes = voteCounts[_tokenIds[i]];
            winnerTokenId = _tokenIds[i];
        }
    }
```

## Impact

In the case of a tie between participants, selection of the winner is unfair and can lead to user disinterest in the voting system

## Tools Used

Manuel review

## Recommendations

`MartenitsaVoting:announceWinner` must be refactored and the best way to get a real randomness for the draw consider to use chainlink VFR

```cpp
    import "@chainlink/contracts/src/v0.8/VRFConsumerBase.sol";
```

Add VRFConsumerBase as inheritance

```cpp
    contract VotingContract is VRFConsumerBase, Ownnable {...}
```

Modifiy the constructor and add some news state variables

```cpp
    bytes32 internal keyHash;
    uint256 internal fee;
    uint256[] private potentialWinners;

    // Parameters for the Goerli Testnet
    constructor(address marketplace, address healthToken, address _martenitsaToken)
        Ownable(msg.sender)
        VRFConsumerBase(
            0x8FbB18354d37f7A587C60f1364b8aA2a05f69B90, // VRF Coordinator for Goerli
            0x326C977E6efc84E512bB9C30f76E30c160eD06FB  // LINK Token for Goerli
        )
    {
         _martenitsaMarketplace = MartenitsaMarketplace(marketplace);
        _healthToken = HealthToken(healthToken);
        martenitsaToken = IMartenitsaToken(_martenitsaToken);
        keyHash = 0x0476f9e5d797756e0f243a643e406a30e46e83c480a91cd96a1769f9c22daa60;
        fee = 0.1 * 10 ** 18; // 0.1 LINK
        owner = msg.sender;  // Setting the owner to the deployer
    }
```

Refactor `MartenitsaVoting:announceWinner`

```cpp
    function announceWinner() external onlyOwner {
        require(block.timestamp >= startVoteTime + duration, "The voting is still active");
        require(LINK.balanceOf(address(this)) >= fee, "Not enough LINK - please fill contract with LINK");

        uint256 maxVotes = 0;
        delete potentialWinners; // Clear previous data
        for (uint256 i = 0; i < _tokenIds.length; i++) {
            uint256 currentVotes = voteCounts[_tokenIds[i]];
            if (currentVotes > maxVotes) {
                maxVotes = currentVotes;
                delete potentialWinners;
                potentialWinners.push(_tokenIds[i]);
            } else if (currentVotes == maxVotes) {
                potentialWinners.push(_tokenIds[i]);
            }
        }

        requestRandomness(keyHash, fee);
    }
```

Add chainlink veritable randomness function

```cpp
    function fulfillRandomness(bytes32 requestId, uint256 randomness) internal override {
        uint256 randomResult = randomness % potentialWinners.length;
        uint256 winnerTokenId = potentialWinners[randomResult];

        list = _martenitsaMarketplace.getListing(winnerTokenId);
        _healthToken.distributeHealthToken(list.seller, 1);
        emit WinnerAnnounced(winnerTokenId, list.seller);
    }
```

# Medium Risk Findings

## <a id='M-01'>`MartenitsaMarketplace:collectReward` in a particular scenario amountRewards can't be correct because `_collectedRewards` mapping isn't reset if users sell at least 3 martenitsa token.</a>

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

## <a id='M-02'>`MartenitsaVoting:voteForMartenitsa` producer can vote for himself during a vote event.</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaVoting.sol#L43

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

Check if caller is a producer if yes revert the transaction, add an interface to access to isProducer and refactor the constructor of MartenitsaVoting contract

```cpp
interface IMartenitsaToken {
    function isProducer(address producer) external view returns (bool);
}

constructor(address marketplace, address healthToken, address _martenitsaToken) Ownable(msg.sender) {
        _martenitsaMarketplace = MartenitsaMarketplace(marketplace);
        _healthToken = HealthToken(healthToken);
        martenitsaToken = IMartenitsaToken(_martenitsaToken);
    }

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

## <a id='L-01'> `MartenitsaToken::createMartenitsa` design @param is not properly checked, producer can create a martenitsa token with an empty string as design or with design without any meaning </a>

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

## <a id='L-02'>`MartenitsaVoting:voteForMartenitsa` user can vote even startvoting is not started from the genesis block to the 86400 blocks.</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaVoting.sol#L45

## Summary

`MartenitsaVoting:voteForMartenitsa` user can vote even startvoting is not started from the genesis block to the 86400 blocks, what represent approximatively 14 days.

## Vulnerability Details

In MartenitsaVoting contract even if `startVoteTime == 0 ;` instead of `startVoteTime = block.timestamp;` it's possible for users to pass the require control structure in `MartenitsaVoting:voteForMartenitsa` during the first 86400 blocks (~14days).

```cpp
/**
     * @notice Function to vote for martenitsa of the sale list.
     * @param tokenId The tokenId of the martenitsa.
     */
    function voteForMartenitsa(uint256 tokenId) external {
        require(!hasVoted[msg.sender], "You have already voted");
        require(block.timestamp < startVoteTime + duration, "The voting is no longer active");
        list = _martenitsaMarketplace.getListing(tokenId);
        require(list.forSale, "You are unable to vote for this martenitsa");

        hasVoted[msg.sender] = true;
        voteCounts[tokenId] += 1;
        _tokenIds.push(tokenId);
    }
```

## Impact

Users can vote during the first 14 days even if voting is not started

## Tools Used

manuel review

## Recommendations

Add a second check in require structure control in `MartenitsaVoting:voteForMartenitsa`

```cpp
require(startVoteTime != 0 && block.timestamp < startVoteTime + duration, "The voting is no longer active");
```

## <a id='L-03'>`MartenitsaEvent:startEvent` `uint256 public eventEndTime` can lead to a unconsiderer revert (through built-in overflow protection) during the execution of startEvent function.</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaEvent.sol#L33

## Summary

`MartenitsaEvent:startEvent` `uint256 public eventEndTime` can lead to unconsiderer revert (through built-in overflow protection) during the execution of startEvent function.. `uint256 public eventEndTime` get the result of two uint256 variables.

## Vulnerability Details

lvalue eventStartTime and eventDuration are declared both as `uint256` the result of adding these 2 lvalue can lead to unconsiderer revert (through built-in overflow protection) during the calcul of the eventEndTime lvalue, this is the operation performed on line 33 of the `MartenitsaEvent.sol` contract

Nota: From version 0.8.0 onward, Solidity includes built-in overflow and underflow protection by default. Arithmetic operations automatically revert if they result in an overflow or underflow.

```cpp
    uint256 public eventStartTime;
    uint256 public eventDuration;
    uint256 public eventEndTime;
```

```cpp
    eventEndTime = eventStartTime + duration;
```

## Impact

Unconsiderer revert (through built-in overflow protection) can occur during the calcul of the eventEndTime, owner the owner may not understand that this is the reason for the revert

## Tools Used

Manuel review

## Recommendations

Using Custom Error Handling

```cpp
    error OverflowDetected();
```

```cpp
    function startEvent(uint256 duration) external onlyOwner {
        eventStartTime = block.timestamp;
        eventDuration = duration;
        if (eventStartTime + duration < eventStartTime || eventStartTime + duration < duration) {  // Check for overflow
            revert OverflowDetected();
        }
        eventEndTime = eventStartTime + duration;
        emit EventStarted(eventStartTime, eventEndTime);
    }
```
