# First Flight #15: Mondrian Wallet - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings

  - ### [H-01.] When a user creates a new Mondrian wallet smart account, no Mondrian art painting NFT is minted. (Validated but downgraded to Medium) (#H-01)

  - ### [H-02.] The URIs point only on the images instead of a json which is not compliant with ERC-721 Metadata Standards. (Validated but downgraded to low and selected for report) (#H-02)

  - ### [H-03.] `_validateSignature` function in `MondrianWallet.sol` does not handle the validationData return value appropriately. (Valideted) (#H-03)

  - ### [H-04.] The current implementation of `MondrianWallet.sol`isn't a valid for zkSync chain. (Validated and selected for report) (#H-04)

- ## Medium Risk Findings

  - ### [M-01.] The function `MondrianWallet:tokenURI` does not provide an equal distribution among the four possible URIs. One URI is favored significantly over the others. (Validated) (#M-01)

  - ### [M-02.] The function `MondrianWallet:tokenURI` does not introduce any randomness itself; it is deterministic based on the value of tokenId. (Validated) (#M-02)

- ## Low Risk Findings

  - ### [L-01.] (#L-01)

# <a id='contest-summary'></a>Contest Summary

Our team loves account abstraction, and abstract art, so we decided to combine them! Users who create an account abstraction wallet with MondrianWallet will get a cool account abstraction wallet, with a random Mondrian art painting!

### Sponsor: CodeHawks First Flight #15

### Dates: May 9th, 2024 (14:00) - May 16th, 2024 (14:00)

[See more contest details here](https://www.codehawks.com/contests/clvxt8idd00014zcc81dv6rde)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 4
- Medium: 2
- Low:

# <a id='final-results'></a> Final Results

https://www.codehawks.com/report/clvxt8idd00014zcc81dv6rde

Final rank: #3

# High Risk Findings

## <a id='H-01'> When a user creates a new Mondrian wallet smart account, no Mondrian art painting NFT is minted. </a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/10886f0f5b34f29e2e950ee3c8147a4793c03257/contracts/MondrianWallet.sol#L59

## Summary

When a user creates a new Mondrian wallet smart account, no Mondrian art painting NFT is minted.

## Vulnerability Details

`MondrianWallet.sol` doesn't call a function `_safeMint` for minting an NFT. The fact that MondrianWallet.sol inherits functions from an ERC721 is not enough to be able to mint an nft every time user create a cool account abstraction wallet.

## Impact

No Mondrian art painting NFT will be minted and distributed to the account abstraction wallet creator.

## Tools Used

Manual review

## Recommendations

To be compliant with your protocol announcement you must refactor your smartcontract to include a way to mint a Mondrian Art Painting NFT during the account abstraction wallet creation, first add an AccountFactory contract to create and deploy your AA wallet and then modify the constructor to allows your smartWallet to mint a Mondrian art painting NFT.

**AccountFactory.sol**

```cpp
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@chainlink/contracts/src/v0.8/VRFConsumerBase.sol";
import "./MondrianWallet.sol";

contract AccountFactory is VRFConsumerBase {
    bytes32 internal keyHash;
    uint256 internal fee;
    address public entryPoint;

    // VRF-related variables
    mapping(bytes32 => address) public requestIdToSender;

    constructor(
        address _vrfCoordinator,
        address _linkToken,
        bytes32 _keyHash,
        uint256 _fee,
        address _entryPoint
    ) VRFConsumerBase(_vrfCoordinator, _linkToken) {
        keyHash = _keyHash;
        fee = _fee;
        entryPoint = _entryPoint;
    }

    function createAccount(address owner) external returns (bytes32 requestId) {
        require(LINK.balanceOf(address(this)) >= fee, "Not enough LINK - fill contract with faucet");
        requestId = requestRandomness(keyHash, fee);
        requestIdToSender[requestId] = owner;
    }

    function fulfillRandomness(bytes32 requestId, uint256 randomness) internal override {
        address owner = requestIdToSender[requestId];
        uint256 tokenId = randomness % 10; // Reduce the randomness range for token ID
        deploy(owner, tokenId);
    }

    function deploy(address owner, uint256 tokenId) internal {
        bytes32 salt = keccak256(abi.encodePacked(owner, tokenId));
        bytes memory creationCode = type(MondrianWallet).creationCode;
        bytes memory bytecode = abi.encodePacked(creationCode, abi.encode(entryPoint, tokenId));

        address walletAddress;
        assembly {
            walletAddress := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
        }
        require(walletAddress != address(0), "Create2: Failed on deploy");
        // Optionally: emit an event here with walletAddress and owner
    }
}
```

**MondrianWallet.sol**

```cpp
constructor(address entryPoint, uint256 _tokenId)
        Ownable(msg.sender)
        ERC721("MondrianWallet", "MW")
    {
        i_entryPoint = IEntryPoint(entryPoint);
        tokenId = _tokenId;
        _safeMint(owner(), tokenId);
        _setTokenURI(tokenId, tokenURI(tokenId));
    }

    // Existing functions...
```

## <a id='H-02'> The URIs point only on the images instead of a JSON which is not compliant with ERC-721 Metadata Standards.</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/10886f0f5b34f29e2e950ee3c8147a4793c03257/contracts/MondrianWallet.sol#L29

## Summary

The URIs point only on the images instead of a json which is not compliant with ERC-721 Metadata Standards.

## Vulnerability Details

All the URIs provided in `MondrianWallet.sol` point to an image instead of a JSON file.

Each can be checked with a browser access as follows:

https://arweave.net/jMRC4pksxwYIgi6vIBsMKXh3Sq0dfFFghSEqrchd_nQ

https://arweave.net/8NI8_fZSi2JyiqSTkIBDVWRGmHCwqHT0qn4QwF9hnPU

https://arweave.net/AVwp_mWsxZO7yZ6Sf3nrsoJhVnJppN02-cbXbFpdOME

https://arweave.net/n17SzjtRkcbHWzcPnm0UU6w1Af5N1p0LAcRUMNP-LiM

## Impact

Non-Compliance with ERC-721 Metadata Standards:

- The ERC-721 standard includes an optional metadata extension (ERC-721 Metadata JSON Schema) which is widely adopted and expected in most use cases. This extension specifies that the URI should return a JSON object. Your contract only links directly to images, it is not utilizing this metadata extension fully.

- Marketplaces, wallets, and other NFT platforms usually rely on this JSON metadata to display information about the NFT to users. Without the JSON, the functionality and integration capabilities of your NFTs can be limited.

## Tools Used

Manual review

## Recommendations

To create a JSON file for an NFT using ArDrive (part of the Arweave ecosystem), you'll need to go through several steps to ensure the JSON meets the ERC-721 metadata standards and is hosted permanently on the Arweave network. Here’s a step-by-step guide:

1. Prepare the JSON Content
   Before uploading anything to ArDrive, you need to prepare the JSON content that will represent your NFT's metadata. Here is a basic structure of what this JSON might include:

```json
{
  "name": "Mondrian ART Painting",
  "description": "A detailed description of what the NFT represents.",
  "image": "https://arweave.net/ExampleImageHash",
  "attributes": [
    {
      "trait_type": "Background",
      "value": "Blue"
    },
    {
      "trait_type": "Eyes",
      "value": "Green"
    }
    // Add more attributes as needed
  ]
}
```

Make sure to replace "https://arweave.net/ExampleImageHash" with the actual URL of the image for your NFT that is already uploaded to Arweave.

2. Upload the JSON File to Arweave using ArDrive

3. Use the JSON File URL in Your Smart Contract

## <a id='H-03'> `_validateSignature` function in `MondrianWallet.sol` does not handle the validationData return value appropriately.</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/78743748751efc269d4cfb6de087f771d950d7c8/contracts/MondrianWallet.sol#L120

## Summary

`_validateSignature` function in `MondrianWallet.sol` does not handle the validationData return value appropriately.

## Vulnerability Details

In `MondrianWallet.sol`, the `_validateSignature` function doesn't recover the address and doesn't check if the recovered address (which should be recovered) matches any authorized signer (such as the owner).

## Impact

This makes the signature validation incomplete and always returns SIG_VALIDATION_SUCCESS.

## Tools Used

Manual review

## Recommendations

Refactor `_validateSignature` to recover the address of signer and to check if considered an authorized user, finaly change function from pure to view to allows the function read state variable.

```cpp
function _validateSignature(PackedUserOperation calldata userOp, bytes32 userOpHash)
        internal
        view
        returns (uint256 validationData)
    {
        bytes32 hash = MessageHashUtils.toEthSignedMessageHash(userOpHash);
        address recoveredAddress = ECDSA.recover(hash, userOp.signature);
        if (recoveredAddress != owner()) {
            return SIG_VALIDATION_FAILED;
        }
        return SIG_VALIDATION_SUCCESS;
    }
```

## <a id='H-04'> The current implementation of `MondrianWallet.sol`isn't a valid for zkSync chain.</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/78743748751efc269d4cfb6de087f771d950d7c8/contracts/MondrianWallet.sol#L19

## Summary

The current implementation of `MondrianWallet.sol`isn't a valid for zkSync chain.

## Vulnerability Details

The structure of MondrianWallet wallet is not compliant with zkSync chain, zkSync use native AA and differ in some points compared to EIP4337.

Key difference:

- Implementation Level
- Account Types
- Transaction Processing
- Paymasters support

Let's focus on transaction processing, EIP 4337 introduces a separate transaction flow for smart contract accounts, which relies on a separate mempool for user operations, and Bundlers - nodes that bundle user operations and sends them to be processed by the EntryPoint contract, resulting in two separate transaction flows. In contrast, on zkSync Era there is a unified mempool for transactions from both Externally Owned Accounts (EOAs) and smart contract accounts. On zkSync Era, the Operator takes on the role of bundling transactions, irrespective of the account type, and sends them to the Bootloader (similar to the EntryPoint contract), which results in a single mempool and transaction flow.

Full description ==> https://docs.zksync.io/build/developer-reference/differences-with-ethereum.html#native-aa-vs-eip-4337

## Impact

You can't use this code base to implement MondrianWallet on zkSync

## Tools Used

Manual review

## Recommendations

Refactor completely your code base and project to be compliant with zkSync chain, you can check DefaultAccount.sol. Keep in mind that you must provide to different code base to address both ERC4337 (Ethereum) and zkSync native AA.

```cpp
// SPDX-License-Identifier: MIT

pragma solidity 0.8.20;

import "./interfaces/IAccount.sol";
import "./libraries/TransactionHelper.sol";
import "./libraries/SystemContractHelper.sol";
import "./libraries/EfficientCall.sol";
import {BOOTLOADER_FORMAL_ADDRESS, NONCE_HOLDER_SYSTEM_CONTRACT, DEPLOYER_SYSTEM_CONTRACT, INonceHolder} from "./Constants.sol";

/**
 * @author Matter Labs
 * @custom:security-contact security@matterlabs.dev
 * @notice The default implementation of account.
 * @dev The bytecode of the contract is set by default for all addresses for which no other bytecodes are deployed.
 * @notice If the caller is not a bootloader always returns empty data on call, just like EOA does.
 * @notice If it is delegate called always returns empty data, just like EOA does.
 */
contract DefaultAccount is IAccount {
    using TransactionHelper for *;

    /**
     * @dev Simulate the behavior of the EOA if the caller is not the bootloader.
     * Essentially, for all non-bootloader callers halt the execution with empty return data.
     * If all functions will use this modifier AND the contract will implement an empty payable fallback()
     * then the contract will be indistinguishable from the EOA when called.
     */
    modifier ignoreNonBootloader() {
        if (msg.sender != BOOTLOADER_FORMAL_ADDRESS) {
            // If function was called outside of the bootloader, behave like an EOA.
            assembly {
                return(0, 0)
            }
        }
        // Continue execution if called from the bootloader.
        _;
    }

    /**
     * @dev Simulate the behavior of the EOA if it is called via `delegatecall`.
     * Thus, the default account on a delegate call behaves the same as EOA on Ethereum.
     * If all functions will use this modifier AND the contract will implement an empty payable fallback()
     * then the contract will be indistinguishable from the EOA when called.
     */
    modifier ignoreInDelegateCall() {
        address codeAddress = SystemContractHelper.getCodeAddress();
        if (codeAddress != address(this)) {
            // If the function was delegate called, behave like an EOA.
            assembly {
                return(0, 0)
            }
        }

        // Continue execution if not delegate called.
        _;
    }

    /// @notice Validates the transaction & increments nonce.
    /// @dev The transaction is considered accepted by the account if
    /// the call to this function by the bootloader does not revert
    /// and the nonce has been set as used.
    /// @param _suggestedSignedHash The suggested hash of the transaction to be signed by the user.
    /// This is the hash that is signed by the EOA by default.
    /// @param _transaction The transaction structure itself.
    /// @dev Besides the params above, it also accepts unused first paramter "_txHash", which
    /// is the unique (canonical) hash of the transaction.
    function validateTransaction(
        bytes32, // _txHash
        bytes32 _suggestedSignedHash,
        Transaction calldata _transaction
    ) external payable override ignoreNonBootloader ignoreInDelegateCall returns (bytes4 magic) {
        magic = _validateTransaction(_suggestedSignedHash, _transaction);
    }

    /// @notice Inner method for validating transaction and increasing the nonce
    /// @param _suggestedSignedHash The hash of the transaction signed by the EOA
    /// @param _transaction The transaction.
    function _validateTransaction(
        bytes32 _suggestedSignedHash,
        Transaction calldata _transaction
    ) internal returns (bytes4 magic) {
        // Note, that nonce holder can only be called with "isSystem" flag.
        SystemContractsCaller.systemCallWithPropagatedRevert(
            uint32(gasleft()),
            address(NONCE_HOLDER_SYSTEM_CONTRACT),
            0,
            abi.encodeCall(INonceHolder.incrementMinNonceIfEquals, (_transaction.nonce))
        );

        // Even though for the transaction types present in the system right now,
        // we always provide the suggested signed hash, this should not be
        // always expected. In case the bootloader has no clue what the default hash
        // is, the bytes32(0) will be supplied.
        bytes32 txHash = _suggestedSignedHash != bytes32(0) ? _suggestedSignedHash : _transaction.encodeHash();

        // The fact there is are enough balance for the account
        // should be checked explicitly to prevent user paying for fee for a
        // transaction that wouldn't be included on Ethereum.
        uint256 totalRequiredBalance = _transaction.totalRequiredBalance();
        require(totalRequiredBalance <= address(this).balance, "Not enough balance for fee + value");

        if (_isValidSignature(txHash, _transaction.signature)) {
            magic = ACCOUNT_VALIDATION_SUCCESS_MAGIC;
        }
    }

    /// @notice Method called by the bootloader to execute the transaction.
    /// @param _transaction The transaction to execute.
    /// @dev It also accepts unused _txHash and _suggestedSignedHash parameters:
    /// the unique (canonical) hash of the transaction and the suggested signed
    /// hash of the transaction.
    function executeTransaction(
        bytes32, // _txHash
        bytes32, // _suggestedSignedHash
        Transaction calldata _transaction
    ) external payable override ignoreNonBootloader ignoreInDelegateCall {
        _execute(_transaction);
    }

    /// @notice Method that should be used to initiate a transaction from this account by an external call.
    /// @dev The custom account is supposed to implement this method to initiate a transaction on behalf
    /// of the account via L1 -> L2 communication. However, the default account can initiate a transaction
    /// from L1, so we formally implement the interface method, but it doesn't execute any logic.
    /// @param _transaction The transaction to execute.
    function executeTransactionFromOutside(Transaction calldata _transaction) external payable override {
        // Behave the same as for fallback/receive, just execute nothing, returns nothing
    }

    /// @notice Inner method for executing a transaction.
    /// @param _transaction The transaction to execute.
    function _execute(Transaction calldata _transaction) internal {
        address to = address(uint160(_transaction.to));
        uint128 value = Utils.safeCastToU128(_transaction.value);
        bytes calldata data = _transaction.data;
        uint32 gas = Utils.safeCastToU32(gasleft());

        // Note, that the deployment method from the deployer contract can only be called with a "systemCall" flag.
        bool isSystemCall;
        if (to == address(DEPLOYER_SYSTEM_CONTRACT) && data.length >= 4) {
            bytes4 selector = bytes4(data[:4]);
            // Check that called function is the deployment method,
            // the others deployer method is not supposed to be called from the default account.
            isSystemCall =
                selector == DEPLOYER_SYSTEM_CONTRACT.create.selector ||
                selector == DEPLOYER_SYSTEM_CONTRACT.create2.selector ||
                selector == DEPLOYER_SYSTEM_CONTRACT.createAccount.selector ||
                selector == DEPLOYER_SYSTEM_CONTRACT.create2Account.selector;
        }
        bool success = EfficientCall.rawCall(gas, to, value, data, isSystemCall);
        if (!success) {
            EfficientCall.propagateRevert();
        }
    }

    /// @notice Validation that the ECDSA signature of the transaction is correct.
    /// @param _hash The hash of the transaction to be signed.
    /// @param _signature The signature of the transaction.
    /// @return EIP1271_SUCCESS_RETURN_VALUE if the signaure is correct. It reverts otherwise.
    function _isValidSignature(bytes32 _hash, bytes memory _signature) internal view returns (bool) {
        require(_signature.length == 65, "Signature length is incorrect");
        uint8 v;
        bytes32 r;
        bytes32 s;
        // Signature loading code
        // we jump 32 (0x20) as the first slot of bytes contains the length
        // we jump 65 (0x41) per signature
        // for v we load 32 bytes ending with v (the first 31 come from s) then apply a mask
        assembly {
            r := mload(add(_signature, 0x20))
            s := mload(add(_signature, 0x40))
            v := and(mload(add(_signature, 0x41)), 0xff)
        }
        require(v == 27 || v == 28, "v is neither 27 nor 28");

        // EIP-2 still allows signature malleability for ecrecover(). Remove this possibility and make the signature
        // unique. Appendix F in the Ethereum Yellow paper (https://ethereum.github.io/yellowpaper/paper.pdf), defines
        // the valid range for s in (301): 0 < s < secp256k1n ÷ 2 + 1, and for v in (302): v ∈ {27, 28}. Most
        // signatures from current libraries generate a unique signature with an s-value in the lower half order.
        //
        // If your library generates malleable signatures, such as s-values in the upper range, calculate a new s-value
        // with 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141 - s1 and flip v from 27 to 28 or
        // vice versa. If your library also generates signatures with 0/1 for v instead 27/28, add 27 to v to accept
        // these malleable signatures as well.
        require(uint256(s) <= 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0, "Invalid s");

        address recoveredAddress = ecrecover(_hash, v, r, s);

        return recoveredAddress == address(this) && recoveredAddress != address(0);
    }

    /// @notice Method for paying the bootloader for the transaction.
    /// @param _transaction The transaction for which the fee is paid.
    /// @dev It also accepts unused _txHash and _suggestedSignedHash parameters:
    /// the unique (canonical) hash of the transaction and the suggested signed
    /// hash of the transaction.
    function payForTransaction(
        bytes32, // _txHash
        bytes32, // _suggestedSignedHash
        Transaction calldata _transaction
    ) external payable ignoreNonBootloader ignoreInDelegateCall {
        bool success = _transaction.payToTheBootloader();
        require(success, "Failed to pay the fee to the operator");
    }

    /// @notice Method, where the user should prepare for the transaction to be
    /// paid for by a paymaster.
    /// @dev Here, the account should set the allowance for the smart contracts
    /// @param _transaction The transaction.
    /// @dev It also accepts unused _txHash and _suggestedSignedHash parameters:
    /// the unique (canonical) hash of the transaction and the suggested signed
    /// hash of the transaction.
    function prepareForPaymaster(
        bytes32, // _txHash
        bytes32, // _suggestedSignedHash
        Transaction calldata _transaction
    ) external payable ignoreNonBootloader ignoreInDelegateCall {
        _transaction.processPaymasterInput();
    }

    fallback() external payable ignoreInDelegateCall {
        // fallback of default account shouldn't be called by bootloader under no circumstances
        assert(msg.sender != BOOTLOADER_FORMAL_ADDRESS);

        // If the contract is called directly, behave like an EOA
    }

    receive() external payable {
        // If the contract is called directly, behave like an EOA
    }
}
```

# Medium Risk Findings

## <a id='M-01'> The function `MondrianWallet:tokenURI` does not provide an equal distribution among the four possible URIs. One URI is favored significantly over the others.</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/78743748751efc269d4cfb6de087f771d950d7c8/contracts/MondrianWallet.sol#L161

## Summary

The function `MondrianWallet:tokenURI` does not provide an equal distribution among the four possible URIs. One URI is favored significantly over the others.

## Vulnerability Details

The function uses the tokenId % 10 operation to determine which URI to return. This results in the following mappings:

- tokenId % 10 == 0 maps to ART_ONE
- tokenId % 10 == 1 maps to ART_TWO
- tokenId % 10 == 2 maps to ART_THREE
- All other results of tokenId % 10 (3 through 9) map to ART_FOUR.

Here’s a breakdown of the distribution:

ART_ONE is returned for 10% of token IDs (those ending in 0).
ART_TWO is returned for 10% of token IDs (those ending in 1).
ART_THREE is returned for 10% of token IDs (those ending in 2).
ART_FOUR is returned for 70% of token IDs (those ending in 3 through 9).

Clearly, the distribution is not equal among the four URIs. ART_FOUR is significantly more common than the others.

## Impact

ART_FOUR will be more distributed than other ARTs which is not compliant with the NFT part annoucement:

_You'll see the tokenURI function returns one of 4 random Mondrian art paintings. Each should have equal distribution and be random._

## Tools Used

Manual review

## Recommendations

Change the modulo operation from 10 to 4, every URI will be selected 25% of the time, assuming tokenId values are uniformly distributed. Here’s how you could adjust the function:

```cpp
function tokenURI(uint256 tokenId) public view override returns (string memory) {
    if (ownerOf(tokenId) == address(0)) {
        revert MondrainWallet__InvalidTokenId();
    }
    uint256 modNumber = tokenId % 4; // Change modulo from 10 to 4
    if (modNumber == 0) {
        return ART_ONE;
    } else if (modNumber == 1) {
        return ART_TWO;
    } else if (modNumber == 2) {
        return ART_THREE;
    } else {
        return ART_FOUR;
    }
}
```

This change ensures that each URI is equally likely to be assigned to any new tokenId minted, assuming a sequential or uniformly random distribution of tokenId values.

## <a id='M-02'> The function `MondrianWallet:tokenURI` does not introduce any randomness itself; it is deterministic based on the value of tokenId.</a>

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/78743748751efc269d4cfb6de087f771d950d7c8/contracts/MondrianWallet.sol#L161

## Summary

The function `MondrianWallet:tokenURI` does not introduce any randomness itself; it is deterministic based on the value of tokenId.

## Vulnerability Details

The randomness of the tokenURI function is directly tied to the randomness of the tokenId values. The output of this function is predictable and not random. This function does not introduce any randomness itself; it is deterministic based on the value of tokenId.

## Impact

The randomness of the tokenURI function is not compliant with the NFT part annoucement:

_You'll see the tokenURI function returns one of 4 random Mondrian art paintings. Each should have equal distribution and be random._

## Tools Used

Manual review

## Recommendations

Refactor MondrianWallet.sol by integrating Chainlink VRF (Verifiable Random Function) for genuinely random URI selection:

```cpp
import "@chainlink/contracts/src/v0.8/VRFConsumerBase.sol";

...

contract MondrianWallet is ERC721, Ownable, IAccount, VRFConsumerBase {
    bytes32 internal keyHash;
    uint256 internal fee;
    mapping(uint256 => uint256) public tokenIdToRandomNumber;
    mapping(bytes32 => uint256) private requestToTokenId;

    string[4] private uris = [
        "ar://jMRC4pksxwYIgi6vIBsMKXh3Sq0dfFFghSEqrchd_nQ",
        "ar://8NI8_fZSi2JyiqSTkIBDVWRGmHCwqHT0qn4QwF9hnPU",
        "ar://AVwp_mWsxZO7yZ6Sf3nrsoJhVnJppN02-cbXbFpdOME",
        "ar://n17SzjtRkcbHWzcPnm0UU6w1Af5N1p0LAcRUMNP-LiM"
    ];

    constructor(address entryPoint, address vrfCoordinator, address linkToken, bytes32 _keyHash, uint256 _fee)
        ERC721("MondrianWallet", "MW")
        VRFConsumerBase(vrfCoordinator, linkToken)
    {
        keyHash = 0x2ed0feb3e9f8dbb8aca68f2a6af959ac6e143cfb420e3d03285037d1eac046e8;
        fee = 0.1 * 10 ** 18; // 0.1 LINK
    }

    ...

    function requestNewRandomURI(uint256 tokenId) public returns (bytes32 requestId) {
        require(LINK.balanceOf(address(this)) >= fee, "Not enough LINK");
        requestId = requestRandomness(keyHash, fee);
        requestToTokenId[requestId] = tokenId;
    }

    function fulfillRandomness(bytes32 requestId, uint256 randomness) internal override {
        uint256 tokenId = requestToTokenId[requestId];
        uint256 randomResult = randomness % 4;
        tokenIdToRandomNumber[tokenId] = randomResult;
    }

    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        require(_exists(tokenId), "ERC721Metadata: URI query for nonexistent token");
        uint256 modNumber = tokenIdToRandomNumber[tokenId];
        return uris[modNumber];
    }

    ...

```

- VRFConsumerBase: The contract inherits from VRFConsumerBase, which is part of Chainlink's VRF functionality.

- Variables for Chainlink VRF: keyHash, fee, and mappings to store randomness requests and results.

- requestNewRandomURI(uint256 tokenId): A function that requests randomness. It requires some LINK token balance to pay for the request.

- fulfillRandomness(bytes32 requestId, uint256 randomness): An internal function that Chainlink nodes call to deliver the randomness. It maps the random number to a tokenId.

Updated tokenURI(uint256 tokenId): Now it retrieves the URI based on the stored random result.

# Low Risk Findings

## <a id='L-01'> </a>

### Relevant GitHub Links

## Summary

## Vulnerability Details

## Impact

## Tools Used

## Recommendations
