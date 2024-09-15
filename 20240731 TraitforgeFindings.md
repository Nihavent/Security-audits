
# High

# Incorrect modifier in `EntropyGenerator::initializeAlphaIndices()` prevents incrementing generations

## Links to affected code

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L206-L216

## Bug Description

`EntropyGenerator::initializeAlphaIndices()` is called upon deployment of the `EntropyGenerator` contract and any time the generation is incremented in the `TraitForgeNft` contract. The `onlyOwner` modifier is applied to the `initializeAlphaIndices()` function which means it can only be called by the owner of the `EntropyGenerator` contract. This is fine for the contract deployment as the caller will be the owner, however future calls to `initializeAlphaIndices()` will only be made by the `TraitForgeNft` which via the `_incrementGeneration()` function. These calls will all revert as the `TraitForgeNft` contract is not the owner of the `EntropyGenerator` contract.

## Impact
All generation increments are prevented, therefore the game cannot progress past generation 1.

## Proof of Concept

Paste the below contents into a new file in the `test` folder. The test denomstates that the protocol is unable to advance beyond generation 1. Please note this is a foundry test and the repo must be setup as a foundry project before running.

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "../../lib/forge-std/src/Test.sol";

import { EntropyGenerator} from '../contracts/EntropyGenerator/EntropyGenerator.sol';
import { Airdrop } from '../contracts/Airdrop/Airdrop.sol';
import { TraitForgeNft } from '../contracts/TraitForgeNft/TraitForgeNft.sol';
import { EntityForging } from '../contracts/EntityForging/EntityForging.sol';
import { DevFund } from '../contracts/DevFund/DevFund.sol';
import { EntityTrading } from '../contracts/EntityTrading/EntityTrading.sol';
import { NukeFund } from '../contracts/NukeFund/NukeFund.sol';

contract TraitForgeNftTest is Test {
    address public owner = makeAddr("owner");
    address public user1 = makeAddr("user1");
    address public user2 = makeAddr("user2");

    EntropyGenerator public entropyGenerator;
    DevFund public devFund;
    Airdrop public airdrop;
    TraitForgeNft public traitForgeNft;
    EntityForging public entityForging;



    function setUp() public {
        vm.roll(42);

        vm.startPrank(owner);

        traitForgeNft = new TraitForgeNft();
        airdrop = new Airdrop();
        entropyGenerator = new EntropyGenerator(address(traitForgeNft));
        entityForging = new EntityForging(address(traitForgeNft));


        // Set contracts on the NFT contract
        traitForgeNft.setEntityForgingContract(address(entityForging));
        traitForgeNft.setEntropyGenerator(address(entropyGenerator));
        traitForgeNft.setAirdropContract(address(airdrop));

        // Transfer ownership of the airdrop contract to the NFT contract
        airdrop.transferOwnership(address(traitForgeNft));

        // Write the entropy in 3 batches
        entropyGenerator.writeEntropyBatch1();
        entropyGenerator.writeEntropyBatch2();
        entropyGenerator.writeEntropyBatch3();

        //devFund = new DevFund();
        
        vm.stopPrank();
    }


    function test_POC_incrementGenerationDOS() public {

        vm.deal(user1, 1300 ether); // enough to mint entire gen 1

        vm.warp(block.timestamp + 24 hours + 1); // Fast forward 24 hours to bypass the mint whitelist
        bytes32[] memory _whiteListProof = new bytes32[](1); // dummy calldata, not required because the call is after the whitelist timestamp

        // entire gen 1 is minted 
        for (uint256 i = 0; i < 10000; i++) {
            vm.prank(user1);
            traitForgeNft.mintToken{value: 1 ether}(_whiteListProof);
        }

        assertEq(traitForgeNft.currentGeneration(), 1);

        // The next token purchase will increment the generation, but the NFT contract is not the owner of the EntropyGenerator contract
        vm.prank(user1);
        vm.expectRevert("Ownable: caller is not the owner");
        traitForgeNft.mintToken{value: 1 ether}(_whiteListProof);   
    }

}
```

## Tools Used

Manual review

## Recommended Mitigation Steps

Replace the `onlyOwner` modifier with the `onlyAllowedCaller` modifer in the `initializeAlphaIndices()` function. Also update the `onlyAllowedCaller` to allow the owner as well. This accounts for the first call upon deployment of the `EntropyGenerator` contract. This doesn't create a new issue as the owner of the contract is already trusted.

```diff
  modifier onlyAllowedCaller() {
-   require(msg.sender == allowedCaller, 'Caller is not allowed');
+   require(msg.sender == allowedCaller || msg.sender == owner(), 'Caller is not allowed');
    _;
  }

- function initializeAlphaIndices() public whenNotPaused onlyOwner {
+ function initializeAlphaIndices() public whenNotPaused onlyAllowedCaller {
    uint256 hashValue = uint256(
      keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp))
    );

    uint256 slotIndexSelection = (hashValue % 258) + 512;
    uint256 numberIndexSelection = hashValue % 13;

    slotIndexSelectionPoint = slotIndexSelection;
    numberIndexSelectionPoint = numberIndexSelection;
  }

```


# `TraitForgeNft::_incrementGeneration()` resets `generationMintCounts` which erases previously forged token count, as a result more tokens than `maxTokensPerGen` can be minted in a given generation

## Links to affected code

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L351

## Bug Description

When two tokens are forged, the `TraitForgeNft::_mintNewEntity()` function is invoked, which increments the `generationMintCounts` mapping for the generation of the token being forged (shown below). Note that the `gen` parameter is the generation of the new token being forged, and need not equal the `currentGeneration` state parameter.

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L311-L343
```javascript
  function _mintNewEntity(
    address newOwner,
    uint256 entropy,
    uint256 gen
  ) private returns (uint256) {
    ... SKIP!...

@>  generationMintCounts[gen]++;
    
    ... SKIP!...
  }
```

When the `currentGeneration` parameter exceeds the `maxTokensPerGen`, the generation is incremented via the `_incrementGeneration()` function which increments the `currentGeneration` and then sets the `generationMintCounts[currentGeneration]` parameter to 0:

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345-L355
```javascript
  function _incrementGeneration() private {
    ... SKIP!...

@>  currentGeneration++;
@>  generationMintCounts[currentGeneration] = 0;
    
    ... SKIP!...
  }
```

This means that any previously forged tokens in the new generation will no longer be counted towards the total number of tokens minted in their generation, which means the total number of tokens in any generation can exceed `maxTokensPerGen`.

Consider the scenario where `currentGeneration` is 1 and 500 tokens are forged in generation 2. When `maxTokensPerGen` is reached in generation 1, the `currentGeneration` will increment to 2 and the `generationMintCounts[2]` is reset to 0. Now another `maxTokensPerGen` can be minted in generation 2, resulting in a total of 500 + `maxTokensPerGen` existing in generation 2.




## Impact
More than `maxTokensPerGen` tokens can exist in a given generation

This issue will occur in every generation except generation 1, and the amount of extra tokens will equal the amount forged prior to that generation being reached.


## Proof of Concept

The test `test_POC_ForgedMintCountsOverriddenUponGenerationIncrementSimple()` does the following:
- Two tokens are minted in generation 1
- These two tokens are forged together which mints the new token in generation 2
- The remaining 9998 tokens in generation 1 are minted
- The next token is minted triggers a generation increment
- The assert statements show the contradiction where two tokens exist in generation 2, but the mapping only counts one.

Please note this is a foundry test and the repo must be setup as a foundry project before running.

Before running the tests, make the below changes to the `EntropyGenerator` contract (to allow generation increments)

```diff
  modifier onlyAllowedCaller() {
-   require(msg.sender == allowedCaller, 'Caller is not allowed');
+   require(msg.sender == allowedCaller || msg.sender == owner(), 'Caller is not allowed');
    _;
  }

- function initializeAlphaIndices() public whenNotPaused onlyOwner {
+ function initializeAlphaIndices() public whenNotPaused onlyAllowedCaller {
    uint256 hashValue = uint256(
      keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp))
    );

    uint256 slotIndexSelection = (hashValue % 258) + 512;
    uint256 numberIndexSelection = hashValue % 13;

    slotIndexSelectionPoint = slotIndexSelection;
    numberIndexSelectionPoint = numberIndexSelection;
  }
```

Paste the below code into a new file in the `test` folder.

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "../../lib/forge-std/src/Test.sol";

import { EntropyGenerator} from '../contracts/EntropyGenerator/EntropyGenerator.sol';
import { Airdrop } from '../contracts/Airdrop/Airdrop.sol';
import { TraitForgeNft } from '../contracts/TraitForgeNft/TraitForgeNft.sol';
import { EntityForging } from '../contracts/EntityForging/EntityForging.sol';
import { DevFund } from '../contracts/DevFund/DevFund.sol';
import { EntityTrading } from '../contracts/EntityTrading/EntityTrading.sol';
import { NukeFund } from '../contracts/NukeFund/NukeFund.sol';

contract TraitForgeNftTest is Test {
    address public owner = makeAddr("owner");
    address public user1 = makeAddr("user1");
    address public user2 = makeAddr("user2");

    EntropyGenerator public entropyGenerator;
    DevFund public devFund;
    Airdrop public airdrop;
    TraitForgeNft public traitForgeNft;
    EntityForging public entityForging;



    function setUp() public {
        vm.roll(42);

        vm.startPrank(owner);

        traitForgeNft = new TraitForgeNft();
        airdrop = new Airdrop();
        entropyGenerator = new EntropyGenerator(address(traitForgeNft));
        entityForging = new EntityForging(address(traitForgeNft));


        // Set contracts on the NFT contract
        traitForgeNft.setEntityForgingContract(address(entityForging));
        traitForgeNft.setEntropyGenerator(address(entropyGenerator));
        traitForgeNft.setAirdropContract(address(airdrop));

        // Transfer ownership of the airdrop contract to the NFT contract
        airdrop.transferOwnership(address(traitForgeNft));

        // Write the entropy in 3 batches
        entropyGenerator.writeEntropyBatch1();
        entropyGenerator.writeEntropyBatch2();
        entropyGenerator.writeEntropyBatch3();

        //devFund = new DevFund();
        
        vm.stopPrank();
    }


    function test_POC_ForgedMintCountsOverriddenUponGenerationIncrementSimple() public {
        vm.deal(user1, 1300 ether);
        vm.deal(user2, 1 ether);

        vm.warp(block.timestamp + 24 hours + 1); // Fast forward 24 hours to bypass the mint whitelist
        bytes32[] memory _whiteListProof = new bytes32[](1); // dummy calldata, not required because the call is after the whitelist timestamp

        // Users both mint a token
        vm.prank(user1);
        traitForgeNft.mintToken{value: 1 ether}(_whiteListProof);
        assert(traitForgeNft.tokenEntropy(1) % 3 == 0); // Ensure token1  is a forger

        vm.prank(user2);
        traitForgeNft.mintToken{value: 1 ether}(_whiteListProof);

        
        uint256 forgerFee = 0.01 ether;
        vm.prank(user1);
        entityForging.listForForging(1, forgerFee); // User 1 lists their token for forging

        vm.prank(user2);
        uint256 newTokenId = entityForging.forgeWithListed{value: forgerFee}(1, 2); // User 2 forges with the listed token

        // Now lets mint out the remainder of gen 1 without any forging.
        uint256 mintsLeft = 9998; // 10,000 - 2
        for (uint256 i = 0; i < mintsLeft; i++) { 
            vm.prank(user1);
            traitForgeNft.mintToken{value: 1 ether}(_whiteListProof);
        }

        vm.prank(user1);
        traitForgeNft.mintToken{value: 1 ether}(_whiteListProof); // 1 more mint triggers a generation change

        assertEq(traitForgeNft.currentGeneration(), 2); // Now in gen 2

        // The below asserts form a contradiction because _tokenId 3 and 10002 are both in gen 2, but generationMintCounts(2) == 1. This means that more than 10,000 tokens can be minted in a given generation.
        assertEq(traitForgeNft.getTokenGeneration(3), 2);
        assertEq(traitForgeNft.getTokenGeneration(10002), 2);
        assertEq(traitForgeNft.generationMintCounts(traitForgeNft.currentGeneration()), 1); // Should be 2
    }

}
```

## Tools Used

Manual review

## Recommended Mitigation Steps

In the `_incrementGeneration()` function, do not reset the `generationMintCounts[currentGeneration]` variable.

```diff
  function _incrementGeneration() private {
    require(
      generationMintCounts[currentGeneration] >= maxTokensPerGen,
      'Generation limit not yet reached'
    );
    currentGeneration++;
-   generationMintCounts[currentGeneration] = 0;
    priceIncrement = priceIncrement + priceIncrementByGen;
    entropyGenerator.initializeAlphaIndices();
    emit GenerationIncremented(currentGeneration);
  }
```



# Medium

# `TraitForgeNft::mintWithBudget()` is DOSed after `maxTokensPerGen` are minted across all generations

## Links to affected code

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L202-L225

## Bug Description

The `mintWithBudget()` function allows users to mint multiple tokens until either the generation is incremented or their budget is spent. The issue with the implementation is it requires `_tokenIds` to be less than `maxTokensPerGen`. `_tokenIds` increments every time a token is minted across all generations and is never reset, meaning as soon as `maxTokensPerGen` tokens are minted, the `mintWithBudget()` function is DOSed forever.

```javascript
  function mintWithBudget(
    bytes32[] calldata proof
  )
    public
    payable
    whenNotPaused
    nonReentrant
    onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
  {

   ... SKIP!...

@>  while (budgetLeft >= mintPrice && _tokenIds < maxTokensPerGen) { // _tokenIds can exceed maxTokensPerGen DOSing this function
      _mintInternal(msg.sender, mintPrice);
      amountMinted++;
      budgetLeft -= mintPrice;
      mintPrice = calculateMintPrice();
    }
    if (budgetLeft > 0) {
      (bool refundSuccess, ) = msg.sender.call{ value: budgetLeft }('');
      require(refundSuccess, 'Refund failed.');
    }
  }
```

## Impact
When a total of `maxTokensPerGen` tokens are minted, the `mintWithBudget()` function is DOSed forever. This impact is guaranteed to occur once the protocol reaches generation 2, but will occur earlier assuming there are forged tokens.

## Proof of Concept

The test `test_POC_MintWithBudgetDOSedAfterGenerationOne()` does the following:
- User1 mints all generation 1 tokens
- User1 mints another token is minted to increment the generation
- User1 attempts to mint several tokens in generation 2 using `mintWithBudget()`, no tokens are minted due to this issue

Please note this is a foundry test and the repo must be setup as a foundry project before running.

Before running the tests, make the below changes to the `EntropyGenerator` contract (to allow generation increments):

```diff
  modifier onlyAllowedCaller() {
-   require(msg.sender == allowedCaller, 'Caller is not allowed');
+   require(msg.sender == allowedCaller || msg.sender == owner(), 'Caller is not allowed');
    _;
  }

- function initializeAlphaIndices() public whenNotPaused onlyOwner {
+ function initializeAlphaIndices() public whenNotPaused onlyAllowedCaller {
    uint256 hashValue = uint256(
      keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp))
    );

    uint256 slotIndexSelection = (hashValue % 258) + 512;
    uint256 numberIndexSelection = hashValue % 13;

    slotIndexSelectionPoint = slotIndexSelection;
    numberIndexSelectionPoint = numberIndexSelection;
  }
```

Paste the below code into a new file in the `test` folder.

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "../../lib/forge-std/src/Test.sol";

import { EntropyGenerator} from '../contracts/EntropyGenerator/EntropyGenerator.sol';
import { Airdrop } from '../contracts/Airdrop/Airdrop.sol';
import { TraitForgeNft } from '../contracts/TraitForgeNft/TraitForgeNft.sol';
import { EntityForging } from '../contracts/EntityForging/EntityForging.sol';
import { DevFund } from '../contracts/DevFund/DevFund.sol';
import { EntityTrading } from '../contracts/EntityTrading/EntityTrading.sol';
import { NukeFund } from '../contracts/NukeFund/NukeFund.sol';

contract TraitForgeNftTest is Test {
    address public owner = makeAddr("owner");
    address public user1 = makeAddr("user1");
    address public user2 = makeAddr("user2");

    EntropyGenerator public entropyGenerator;
    DevFund public devFund;
    Airdrop public airdrop;
    TraitForgeNft public traitForgeNft;
    EntityForging public entityForging;



    function setUp() public {
        vm.roll(42);

        vm.startPrank(owner);

        traitForgeNft = new TraitForgeNft();
        airdrop = new Airdrop();
        entropyGenerator = new EntropyGenerator(address(traitForgeNft));
        entityForging = new EntityForging(address(traitForgeNft));


        // Set contracts on the NFT contract
        traitForgeNft.setEntityForgingContract(address(entityForging));
        traitForgeNft.setEntropyGenerator(address(entropyGenerator));
        traitForgeNft.setAirdropContract(address(airdrop));

        // Transfer ownership of the airdrop contract to the NFT contract
        airdrop.transferOwnership(address(traitForgeNft));

        // Write the entropy in 3 batches
        entropyGenerator.writeEntropyBatch1();
        entropyGenerator.writeEntropyBatch2();
        entropyGenerator.writeEntropyBatch3();

        //devFund = new DevFund();
        
        vm.stopPrank();
    }

    function test_POC_MintWithBudgetDOSedAfterGenerationOne() public {
        vm.deal(user1, 1300 ether); // give user 1 enough ETH to mint out gen 1
        vm.warp(block.timestamp + 24 hours + 1); // Fast forward 24 hours to bypass the mint whitelist
        bytes32[] memory _whiteListProof = new bytes32[](1); // dummy calldata, not required because the call is after the whitelist timestamp

        // User 1 mints out Gen 1
        vm.startPrank(user1);
        for (uint256 i = 0; i < 10000; i++) { 
            traitForgeNft.mintToken{value: 1 ether}(_whiteListProof);
        }
        
        // 1 more mint triggers a generation change
        traitForgeNft.mintToken{value: 1 ether}(_whiteListProof);

        assertEq(traitForgeNft.currentGeneration(), 2); // We're now in gen 2

        
        // User attempts to use the mintWithBudget function to mint a few tokens
        uint256 pre_user1TokenBalance = traitForgeNft.balanceOf(user1);
        traitForgeNft.mintWithBudget{value: 1 ether}(_whiteListProof);
        uint256 post_user1TokenBalance = traitForgeNft.balanceOf(user1);

        assertEq(pre_user1TokenBalance, post_user1TokenBalance); // User 1's token balance did not change
    }

}
```

## Tools Used

Manual review

## Recommended Mitigation Steps

Replace `_tokenIds` with `generationMintCounts[currentGeneration]` as shown below. This allows `mintWithBudget()` to execute until the budget is depleted or the entire generation is minted.

```diff
  function mintWithBudget(
    bytes32[] calldata proof
  )
    public
    payable
    whenNotPaused
    nonReentrant
    onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
  {
    uint256 mintPrice = calculateMintPrice();
    uint256 amountMinted = 0;
    uint256 budgetLeft = msg.value;


-   while (budgetLeft >= mintPrice && _tokenIds < maxTokensPerGen) {
+   while (budgetLeft >= mintPrice && generationMintCounts[currentGeneration] < maxTokensPerGen)
      _mintInternal(msg.sender, mintPrice);
      amountMinted++;
      budgetLeft -= mintPrice;
      mintPrice = calculateMintPrice();
    }
    if (budgetLeft > 0) {
      (bool refundSuccess, ) = msg.sender.call{ value: budgetLeft }('');
      require(refundSuccess, 'Refund failed.');
    }
  }
```


# Due to an incorrect `require` statement in the `writeEntropyBatch()` functions, there is no protection against multiple 'Golden God' tokens apprearing in a single generation

## Links to affected code

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L47-L98

## Bug Description

As confirmed by the docs and the protocol sponsor, there should be 1 'Golden God' entropy/token per generation. However the code does not enforce this correctly.

See below in the `EntropyGenerator::writeEntropyBatch1()` function (note this is the same in `writeEntropyBatch2()` and `writeEntropyBatch3()`) there is an attempt to ensure that the entropy in any given slot and index does not equal 999999 which represents the entropy of a 'Golden God' token. However this checks if the entire `pseudoRandomValue` is equal to 999999, not whether a specific 6-digit number within the `pseudoRandomValue` is equal to 999999.

```javascript
  function writeEntropyBatch1() public {
    require(lastInitializedIndex < batchSize1, 'Batch 1 already initialized.');


    uint256 endIndex = lastInitializedIndex + batchSize1; // calculate the end index for the batch
    unchecked {
      for (uint256 i = lastInitializedIndex; i < endIndex; i++) {
        uint256 pseudoRandomValue = uint256(
          keccak256(abi.encodePacked(block.number, i))
        ) % uint256(10) ** 78; // generate a  pseudo-random value using block number and index
@>      require(pseudoRandomValue != 999999, 'Invalid value, retry.');
        entropySlots[i] = pseudoRandomValue; // store the value in the slots array
      }
    }
    lastInitializedIndex = endIndex;
  }
```

## Impact
- As a result of random chance during deploymenet, the entropy batches written may contain an entropy of 999999. When this occurs more than one 'Golden God' can be minted in a generation.
- Additionally, due to the above random chance, the 'Golden God' token can occur earlier the third batch in a generation.

## Proof of Concept

The below coded POC shows a deployment where there are two 'Golden God' entropies (slot 32 position 3, and slot 754 position 1). This demonstrates both impacts: 
1. In generation 1 if there is less than ~ 350 forges then there will be two 'Golden God' tokens in generation 1. However if there are more than ~350 forges, generation 1 will contain 1 'Golden God' and generation 2 will contain at least 2 and sometimes 3 'Golden God' tokens (depending on where the `slotIndexSelectionPoint` and `numberIndexSelectionPoint` are initialized at generation increment). 
2. The first 'Golden God' in generation 1 will occur much earlier than desired (at slot 32 position 3) instead of between the desired slots of 512 and 769.

Please note this is a foundry test and the repo must be setup as a foundry project before running.

Paste the below code into a new file in the `test` folder.

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "../../lib/forge-std/src/Test.sol";

import { EntropyGenerator} from '../contracts/EntropyGenerator/EntropyGenerator.sol';
import { Airdrop } from '../contracts/Airdrop/Airdrop.sol';
import { TraitForgeNft } from '../contracts/TraitForgeNft/TraitForgeNft.sol';
import { EntityForging } from '../contracts/EntityForging/EntityForging.sol';
import { DevFund } from '../contracts/DevFund/DevFund.sol';
import { EntityTrading } from '../contracts/EntityTrading/EntityTrading.sol';
import { NukeFund } from '../contracts/NukeFund/NukeFund.sol';

contract TraitForgeNftTest is Test {
    address public owner = makeAddr("owner");
    address public user1 = makeAddr("user1");
    address public user2 = makeAddr("user2");

    EntropyGenerator public entropyGenerator;
    DevFund public devFund;
    Airdrop public airdrop;
    TraitForgeNft public traitForgeNft;
    EntityForging public entityForging;



    function setUp() public {
        vm.roll(42);

        vm.startPrank(owner);

        traitForgeNft = new TraitForgeNft();
        airdrop = new Airdrop();
        entropyGenerator = new EntropyGenerator(address(traitForgeNft));
        entityForging = new EntityForging(address(traitForgeNft));


        // Set contracts on the NFT contract
        traitForgeNft.setEntityForgingContract(address(entityForging));
        traitForgeNft.setEntropyGenerator(address(entropyGenerator));
        traitForgeNft.setAirdropContract(address(airdrop));

        // Transfer ownership of the airdrop contract to the NFT contract
        airdrop.transferOwnership(address(traitForgeNft));

        // Write the entropy in 3 batches
        entropyGenerator.writeEntropyBatch1();
        entropyGenerator.writeEntropyBatch2();
        entropyGenerator.writeEntropyBatch3();

        //devFund = new DevFund();
        
        vm.stopPrank();
    }

    function test_POC_MultipleAlphaEntropiesUponDeployment2() public {
            vm.roll(2627);

            // re deploy entropyGenerator contract
            entropyGenerator = new EntropyGenerator(address(traitForgeNft));
            // Write the entropy in 3 batches
            entropyGenerator.writeEntropyBatch1();
            entropyGenerator.writeEntropyBatch2();
            entropyGenerator.writeEntropyBatch3();

            // show there is two 'gold god' entropies in this deployment 
            assertEq(entropyGenerator.getPublicEntropy(32, 3), 999999); // In addition to there being 2 god NFTs, the first one is at slot 32 which is well before the intended slots of between 512 and 769
            assertEq(entropyGenerator.getPublicEntropy(754, 1), 999999);
    }

}
```

## Tools Used

Manual review

## Recommended Mitigation Steps

When writing the entropy batches, iterate through each `pseudoRandomValue` to ensure that no entropy is equal to 999999. For example the following change could be made to `writeEntropyBatch1()`:

```diff
  function writeEntropyBatch1() public {
    require(lastInitializedIndex < batchSize1, 'Batch 1 already initialized.');
+   uint256 entropy;
+   uint256 paddedEntropy;
+   uint256 position;
    uint256 endIndex = lastInitializedIndex + batchSize1; // calculate the end index for the batch
    unchecked {
      for (uint256 i = lastInitializedIndex; i < endIndex; i++) {
        uint256 pseudoRandomValue = uint256(
          keccak256(abi.encodePacked(block.number, i))
        ) % uint256(10) ** 78; // generate a  pseudo-random value using block number and index
-       require(pseudoRandomValue != 999999, 'Invalid value, retry.');
+       for (uint256 j = 0; j < maxNumberIndex; j++) {
+           position = j * 6;
+           entropy = (pseudoRandomValue / (10 ** (72 - position))) % 1000000;
+           paddedEntropy = entropy * (10 ** (6 - numberOfDigits(entropy))); 
+           require(paddedEntropy != 999999, 'Invalid value, retry.');
+       }
        entropySlots[i] = pseudoRandomValue; // store the value in the slots array
      }
    }
    lastInitializedIndex = endIndex;
  }
```


# Due to the the `entropySlots` array containing 10,010 slots, it is possible a generation will not contain a 'Golden God' token

## Links to affected code

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L47-L61
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L64-L81
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L84-L98
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L206-L216

## Bug Description

As confirmed by the docs and the protocol sponsor, there should be 1 'Golden God' entropy/token per generation. However it is possible a generation will not contain any 'Golden God' NFTs due to the size of the `entropySlots` array.

The `entropySlots` array will be filled upon deployment with pseudo random uint256 values. The array contains 770 slots, each containing 13 entropies resulting in 10,010 total entropies. This total number of entropies is larger than the expected value of `TraitForgeNft::maxTokensPerGen` which is 10,000. Due to this difference, the 'Golden God' entropy may not be minted in every generation.

Upon deploymenet of the `EntropyGenerator` contract, `initializeAlphaIndices()` is called which selects the first `slotIndexSelectionPoint` and `numberIndexSelectionPoint` values. 

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L206-L216
```javascript
  function initializeAlphaIndices() public whenNotPaused onlyOwner {
    uint256 hashValue = uint256(
      keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp))
    );

    uint256 slotIndexSelection = (hashValue % 258) + 512; // value between 512 and 769 inclusive
    uint256 numberIndexSelection = hashValue % 13; // value between 0 and 12 inclusive

    slotIndexSelectionPoint = slotIndexSelection;
    numberIndexSelectionPoint = numberIndexSelection;
  }
```

The bug occurs in generation 1 when, due to random chance, the `slotIndexSelectionPoint` value is initialised with a value of 769, and `numberIndexSelectionPoint` is initialized with a value between 3 and 12 inclusive. Under these circumstances, generation 1 and will not have a 'Golden God' token minted.

The bug occurs in future generations when, certain values of `slotIndexSelectionPoint` and `numberIndexSelectionPoint` are set upon generation increment. The required values for this to occur vary depending on the present values of `currentSlotIndex` and `currentNumberIndex`. 

## Impact
- With current implementation it is possible that generations may not contain a 'Golden God' token.
- When this occurs, it is likely that the 'Golden God' token will occur earlier than expected in the following generation (earlier than the first 66% of mints).

Note this has the same impact as another issue I submitted, however both issues have distinct root causes.

## Proof of Concept
The below coded POC shows a deployment where generation 1 will not contain a 'Golden God' NFT.

Please note this is a foundry test and the repo must be setup as a foundry project before running.

Paste the below code into a new file in the `test` folder.

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "../../lib/forge-std/src/Test.sol";

import { EntropyGenerator} from '../contracts/EntropyGenerator/EntropyGenerator.sol';
import { Airdrop } from '../contracts/Airdrop/Airdrop.sol';
import { TraitForgeNft } from '../contracts/TraitForgeNft/TraitForgeNft.sol';
import { EntityForging } from '../contracts/EntityForging/EntityForging.sol';
import { DevFund } from '../contracts/DevFund/DevFund.sol';
import { EntityTrading } from '../contracts/EntityTrading/EntityTrading.sol';
import { NukeFund } from '../contracts/NukeFund/NukeFund.sol';

contract TraitForgeNftTest is Test {
    address public owner = makeAddr("owner");
    address public user1 = makeAddr("user1");
    address public user2 = makeAddr("user2");

    EntropyGenerator public entropyGenerator;
    DevFund public devFund;
    Airdrop public airdrop;
    TraitForgeNft public traitForgeNft;
    EntityForging public entityForging;


    function setUp() public {
        vm.roll(42);

        vm.startPrank(owner);

        traitForgeNft = new TraitForgeNft();
        airdrop = new Airdrop();
        entropyGenerator = new EntropyGenerator(address(traitForgeNft));
        entityForging = new EntityForging(address(traitForgeNft));


        // Set contracts on the NFT contract
        traitForgeNft.setEntityForgingContract(address(entityForging));
        traitForgeNft.setEntropyGenerator(address(entropyGenerator));
        traitForgeNft.setAirdropContract(address(airdrop));

        // Transfer ownership of the airdrop contract to the NFT contract
        airdrop.transferOwnership(address(traitForgeNft));

        // Write the entropy in 3 batches
        entropyGenerator.writeEntropyBatch1();
        entropyGenerator.writeEntropyBatch2();
        entropyGenerator.writeEntropyBatch3();

        //devFund = new DevFund();
        
        vm.stopPrank();
    }

    function test_POC_DeploymentWithNoAlphaEntropyInGen1() public {
        vm.roll(305);

        // re deploy entropyGenerator contract
        entropyGenerator = new EntropyGenerator(address(traitForgeNft));
        // Write the entropy in 3 batches
        entropyGenerator.writeEntropyBatch1();
        entropyGenerator.writeEntropyBatch2();
        entropyGenerator.writeEntropyBatch3();

        // Show that the first 10,000 mints will not contain a 'Golden God' entropy of 999,999
        // The first 10,000 mints occur on slots 0-768 and slot 769 indexes 0-2 inclusive.
        for (uint256 i = 0; i < 770; i++) {
            for (uint256 j = 0; j < 13; j++) {
                if (i == 769 && j > 2) {
                    continue;
                }
                assert(entropyGenerator.getPublicEntropy(i, j) != 999999);
            }
        }
    }
}
```

## Tools Used

Manual review

## Recommended Mitigation Steps

Without re-designing the way entropy is initalized, three changes are required:

1. Ignore the last 10 values in the final slot of `entropySlots`, and
2. Upon deployment or generation increment, if `slotIndexSelectionPoint` is 769 and `numberIndexSelectionPoint` is between 3 and 12 inclusive, chose new values.
3. Upon generation increment, reset the value of `currentSlotIndex` and `currentNumberIndex` back to 0.




# Due to forging, it is possible a generation will not contain a 'Golden God' token

## Links to affected code

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L293
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L328
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L288
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L101-L120
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L176

## Bug Description

As confirmed by the docs and the protocol sponsor, there should be 1 'Golden God' entropy/token per generation. However this is unlikely to be the case due to forging. 

After the generation has incremented, forges to the current generation and 'standard' mints both contribute towards the total tokens minted in the current generation. This is because [`TraitForgeNft::_mintInternal()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L293) and [`TraitForgeNft::_mintNewEntity()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L328) both increment the `generationMintCounts` mapping variable.

However, [`TraitForgeNft::_mintInternal()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L288) fetches the entropy from the newly minted token via the [`EntropyGenerator::getNextEntropy()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L101-L120) call and [`TraitForgeNft::_mintNewEntity()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L176) recieves the entropy of the newly minted token as calculated in the ['TraitForgeNft::forge()'](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L173) function.

The important difference between the entropy retrieval between the above two types of minting is `_mintInternal()` increments the `EntropyGenerator::currentSlotIndex` and `EntropyGenerator::currentNumberIndex` state variables whilst `_mintNewEntity()` does not. 
The result is the 'forges' contribute to exceeding the `generationMintCounts`, whilst not incrementing the entropy indicies, so it is possible a whole generation gets minted without reaching the 'Golden God' entropy index.

Imagine the following scenario which will also be demonstrated below with a POC:
- Generation 1 is minted out with 'standard' mints
- The next mint increments the `TraitForgeNft::currentGeneration()` to 2
- The call to `entropyGenerator.initializeAlphaIndices()` initializes the indicies such that the 'Golden God' will be minted at slot 769, index 3.
- 30 tokens from gen 1 are now forged to gen 2 (importantly this occurs after the generation has incremented to 2). This results in 15 tokens minted in gen 2 via the `TraitForgeNft::_mintNewEntity()` method.
- After a total of 986 gen 1 tokens are minted via `TraitForgeNft::_mintInternal()`, the generation increments to 3.
- There is no 'Golden God' in generation 2.

## Impact
- It is possible that generations contains no 'Golden God'
- When this occurs, it is likely that the 'Golden God' token will occur earlier than expected in the following generation (earlier than the first 66% of mints).

Note this has the same impact as another issue I submitted, however both issues have distinct root causes.

## Proof of Concept

The test `test_POC_NoGoldenGodEntropyDueToForging()` does the following:
- User1 mints 5000x gen 1 tokens
- User2 mints 5000 gen 1 tokens and 1x gen 2 tokens
- User2 lists 40x of their gen 1 forgers for forging
- User1 matches the above 40x forgers with mergers, this mints 40 tokens in gen 2
- User1 mints out the remainder of gen 2
- No 'Golden God' exists in gen 2

Please note this is a foundry test and the repo must be setup as a foundry project before running.

Before running the tests, make the below changes to the `EntropyGenerator` contract (to allow generation increments):

```diff
  modifier onlyAllowedCaller() {
-   require(msg.sender == allowedCaller, 'Caller is not allowed');
+   require(msg.sender == allowedCaller || msg.sender == owner(), 'Caller is not allowed');
    _;
  }

- function initializeAlphaIndices() public whenNotPaused onlyOwner {
+ function initializeAlphaIndices() public whenNotPaused onlyAllowedCaller {
    uint256 hashValue = uint256(
      keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp))
    );

    uint256 slotIndexSelection = (hashValue % 258) + 512;
    uint256 numberIndexSelection = hashValue % 13;

    slotIndexSelectionPoint = slotIndexSelection;
    numberIndexSelectionPoint = numberIndexSelection;
  }
```

Paste the below code into a new file in the `test` folder.


```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "../../lib/forge-std/src/Test.sol";

import { EntropyGenerator} from '../contracts/EntropyGenerator/EntropyGenerator.sol';
import { Airdrop } from '../contracts/Airdrop/Airdrop.sol';
import { TraitForgeNft } from '../contracts/TraitForgeNft/TraitForgeNft.sol';
import { EntityForging } from '../contracts/EntityForging/EntityForging.sol';
import { DevFund } from '../contracts/DevFund/DevFund.sol';
import { EntityTrading } from '../contracts/EntityTrading/EntityTrading.sol';
import { NukeFund } from '../contracts/NukeFund/NukeFund.sol';

contract TraitForgeNftTest is Test {
    address public owner = makeAddr("owner");
    address public user1 = makeAddr("user1");
    address public user2 = makeAddr("user2");

    EntropyGenerator public entropyGenerator;
    DevFund public devFund;
    Airdrop public airdrop;
    TraitForgeNft public traitForgeNft;
    EntityForging public entityForging;



    function setUp() public {
        vm.roll(42);

        vm.startPrank(owner);

        traitForgeNft = new TraitForgeNft();
        airdrop = new Airdrop();
        entropyGenerator = new EntropyGenerator(address(traitForgeNft));
        entityForging = new EntityForging(address(traitForgeNft));


        // Set contracts on the NFT contract
        traitForgeNft.setEntityForgingContract(address(entityForging));
        traitForgeNft.setEntropyGenerator(address(entropyGenerator));
        traitForgeNft.setAirdropContract(address(airdrop));

        // Transfer ownership of the airdrop contract to the NFT contract
        airdrop.transferOwnership(address(traitForgeNft));

        // Write the entropy in 3 batches
        entropyGenerator.writeEntropyBatch1();
        entropyGenerator.writeEntropyBatch2();
        entropyGenerator.writeEntropyBatch3();

        //devFund = new DevFund();
        
        vm.stopPrank();
    }


    function test_POC_NoGoldenGodEntropyDueToForging() public { 

        vm.warp(block.timestamp + 24 hours + 1); // Fast forward 24 hours to bypass the mint whitelist
        bytes32[] memory _whiteListProof = new bytes32[](1); // dummy calldata, not required because the call is after the whitelist timestamp

        // user 1 mints 5000x gen 1 tokens
        vm.deal(user1, 5000 ether);
        vm.startPrank(user1);
        for (uint256 i = 0; i < 5000; i++) {
            traitForgeNft.mintToken{value: 1 ether}(_whiteListProof);
        }

        // user 2 mints 5000x gen 1 tokens and 1x gen 2 token
        vm.deal(user2, 5000 ether);
        vm.stopPrank();
        vm.startPrank(user2);
        for (uint256 i = 0; i < 5000; i++) {
            traitForgeNft.mintToken{value: 1 ether}(_whiteListProof);
        }
        
        // set seed for gen 2 alpha indicies
        vm.roll(1044); // note this is not highly constrained due to the potential of thousands of forges

        // mint a token triggering a generation increment
        traitForgeNft.mintToken{value: 1 ether}(_whiteListProof);

        // Check alpha indicies in gen 2
        assertEq(traitForgeNft.currentGeneration(), 2);

        // The alpha entropy is at slot 768, number 0. So to demonstrate this issue we need to forge > 10 + 2*13 ~ 36 tokens in gen 2.

        // User 2 lists 40 tokens for forging
        uint256 numListed = 0;
        uint256 tokenId = 5000;
        uint256 forgerFee = 0.01 ether;
        uint256[] memory listedTokenIds = new uint256[](40);

        while (numListed < 40) {
            if (traitForgeNft.isForger(tokenId) && (uint8((traitForgeNft.tokenEntropy(tokenId) / 10) % 10)) > 0) {
                entityForging.listForForging(tokenId, forgerFee);

                listedTokenIds[numListed] = tokenId;
                numListed++;
            }
            tokenId++;
        }

        // User 1 matches the 40 listed tokens
        vm.stopPrank();
        vm.startPrank(user1);

        uint256 numMatched = 0;
        tokenId = 0;

        while (numMatched < 40) {
            if (!traitForgeNft.isForger(tokenId) && (uint8((traitForgeNft.tokenEntropy(tokenId) / 10) % 10)) > 0) {
                entityForging.forgeWithListed{value: forgerFee}(listedTokenIds[numMatched], tokenId);

                numMatched++;
            }
            tokenId++;
        }

        assertEq(traitForgeNft.generationMintCounts(2), 41); // 41 tokens minted in gen 2

        // User 1 mints out the remainder of gen 2
        for (uint256 j = 0; j < 10000 - 41; j++) {
            traitForgeNft.mintToken{value: 1 ether}(_whiteListProof);
        }

        assertEq(traitForgeNft.generationMintCounts(2), 10000); // Gen 2 minted out

        // None of the 10,000 tokens in gen 2 are a 'Golden God'
        for (uint256 z = 10000; z < 2*10000; z++) { // in this case, we minted the tokens in order, so the tokenIds 10000 -> 19999 are all gen 2
            assert(traitForgeNft.tokenEntropy(z) != 999999);
        }
    }

}
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

One solution is to increment the `EntropyGenerator::currentSlotIndex` and `EntropyGeneratior::currentNumberIndex` parameters when a token is minted due to forging. In addition, if the forged token is on the current generations's 'alpha indicies' (`slotIndexSelectionPoint`, `numberIndexSelectionPoint`) set the entropy for this token to 999,999. 





# Due to the maximum value of a uint256 and the modulo in the `writeEntropyBatch()` functions, the first entropy per slot can never be a 'forger' and will always have a `performanceFactor` of 0

## Links to affected code

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L47-L61
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L64-L81
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L84-L98
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L164-L185
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L128-L132
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L143-L148
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L92-L93


## Bug Description

The entropy values are stored in the `EntropyGenerator::entropySlots` state variable which is an array of uint256. The assumption made by the protocol is each uint256 is a 78 digit number and therefore we can index 13 non-overlapping 6-digit values from each uint256 value resulting in pseudo-random entropy values.

In reality, uint256 values range between 0 and ~1.1579e77. A randomly selected uint256 has a 1e77 / 1.1579e77 (~86%) chance to be 77 digits in length. Thereofore we can expect, on average, 86% of slots will contain a 77 digit (or less) uint256. When this occurs, the returned entropy via the `getEntropy()` function will be padded with zeros up to 6 total digits:

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L164-L185
```javascript
  function getEntropy(
    uint256 slotIndex,
    uint256 numberIndex
  ) private view returns (uint256) {
    require(slotIndex <= maxSlotIndex, 'Slot index out of bounds.');


    if (
      slotIndex == slotIndexSelectionPoint &&
      numberIndex == numberIndexSelectionPoint
    ) {
      return 999999;
    }


    uint256 position = numberIndex * 6; // calculate the position for slicing the entropy value
    require(position <= 72, 'Position calculation error');


    uint256 slotValue = entropySlots[slotIndex]; // slice the required [art of the entropy value
    uint256 entropy = (slotValue / (10 ** (72 - position))) % 1000000; // adjust the entropy value based on the number of digits
@>  uint256 paddedEntropy = entropy * (10 ** (6 - numberOfDigits(entropy)));


    return paddedEntropy; // return the caculated entropy value
  }
```

This issue worsened due to the modulo in the three `writeEntropyBatch()` functions (`writeEntropyBatch1()`, `writeEntropyBatch2()`, `writeEntropyBatch3()`). As shown below, the `pseudoRandomValue` which gets written to a slot is the result of a 'random number' mod `uint256(10) ** 78` wrapped by `unchecked` to ignore overflows:

```javascript
  function writeEntropyBatch1() public {
    require(lastInitializedIndex < batchSize1, 'Batch 1 already initialized.');

    uint256 endIndex = lastInitializedIndex + batchSize1; // calculate the end index for the batch
    unchecked {
      for (uint256 i = lastInitializedIndex; i < endIndex; i++) {
        uint256 pseudoRandomValue = uint256(
          keccak256(abi.encodePacked(block.number, i))
@>      ) % uint256(10) ** 78; // generate a  pseudo-random value using block number and index
        require(pseudoRandomValue != 999999, 'Invalid value, retry.');
        entropySlots[i] = pseudoRandomValue; // store the value in the slots array
      }
    }
    lastInitializedIndex = endIndex;
  }
```

The value of `uint256(10) ** 78` is around ~8.64 times larger than the max uint256 value. Therefore, when each of the `writeEntropyBatch()` functions are executed, the value of `uint256(10) ** 78` overflows 8 times and becomes the remainder which is ~ 7.37e76. So the final value written to each `entropySlot[i]` is the 'random' uint256 value mod ~7.37e76. There are two possibilities here:
  1. If the original 'random' uint256 was less than ~7.37e76, then it is unchanged and contains 77 digits
  2. If the original 'random' uint256 was greater than ~7.37e76, it is reduced to a maximum value of 1.1579e77 - ~7.37e76 ~ 4.21e76, which also contains 77 digits.

Either way, all entropies written to the `entropySlot` array will have 77 digits, and therefore the first entropy extracted from each slot will have 6th digit 0 (ie. will be evenly divisible by 10).


## Impact
With the exception of the 'Golden God' slot, the first entropy per slot will always have a 6th digit of 0, resulting in two direct impacts:

1. They always have a `performanceFactor` of 0, meaning they do not [age](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L128-L132) for the purposes of the `finalNukeFactor` calculation. To be specific, tokens with a `performanceFactor` of 0 will have a `finalNukeFactor` [equal to](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L143-L148) an `initialNukeFactor`, so they cannot accrue 'potential nuke value' over time. 
2. Entropies that have a 6th digit of 0 can never be 'forgers', because 'forgers' are defined by having an [entropy which is evenly divisible by 3](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L92-L93). 

The above direct impacts also have two implied impacts:
1. Tokens minted with the first entropy from any slot are on average, less valuable than other tokens.
2. Less than 2/3 of all tokens will be 'forgers'


## Proof of Concept

The following coded POC shows that for a deploymenet (the seed doesn't matter this is always true), the first entropy per slot will always be divisible by 10.

Please note this is a foundry test and the repo must be setup as a foundry project before running.

Paste the below code into a new file in the `test` folder.

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "../../lib/forge-std/src/Test.sol";

import { EntropyGenerator} from '../contracts/EntropyGenerator/EntropyGenerator.sol';
import { Airdrop } from '../contracts/Airdrop/Airdrop.sol';
import { TraitForgeNft } from '../contracts/TraitForgeNft/TraitForgeNft.sol';
import { EntityForging } from '../contracts/EntityForging/EntityForging.sol';
import { DevFund } from '../contracts/DevFund/DevFund.sol';
import { EntityTrading } from '../contracts/EntityTrading/EntityTrading.sol';
import { NukeFund } from '../contracts/NukeFund/NukeFund.sol';

contract TraitForgeNftTest is Test {
    address public owner = makeAddr("owner");
    address public user1 = makeAddr("user1");
    address public user2 = makeAddr("user2");

    EntropyGenerator public entropyGenerator;
    DevFund public devFund;
    Airdrop public airdrop;
    TraitForgeNft public traitForgeNft;
    EntityForging public entityForging;



    function setUp() public {
        vm.roll(42);

        vm.startPrank(owner);

        traitForgeNft = new TraitForgeNft();
        airdrop = new Airdrop();
        entropyGenerator = new EntropyGenerator(address(traitForgeNft));
        entityForging = new EntityForging(address(traitForgeNft));


        // Set contracts on the NFT contract
        traitForgeNft.setEntityForgingContract(address(entityForging));
        traitForgeNft.setEntropyGenerator(address(entropyGenerator));
        traitForgeNft.setAirdropContract(address(airdrop));

        // Transfer ownership of the airdrop contract to the NFT contract
        airdrop.transferOwnership(address(traitForgeNft));

        // Write the entropy in 3 batches
        entropyGenerator.writeEntropyBatch1();
        entropyGenerator.writeEntropyBatch2();
        entropyGenerator.writeEntropyBatch3();

        //devFund = new DevFund();
        
        vm.stopPrank();
    }

    function test_POC_FirstEntropyPerSlotAlwaysDivisibleByTen() public {
        for (uint256 i = 0; i < 770; i++) {
            // ignore the alpha NFT, this is manually set to 999999 at the specified index
            if (entropyGenerator.slotIndexSelectionPoint() == i && entropyGenerator.numberIndexSelectionPoint() == 0) {
                continue;
            }
            assertEq(entropyGenerator.getPublicEntropy(i, 0) % 10, 0); // First entropy per slot is always divisible by 10 meaning it has 0 performanceFactor and can never be a forger.
        }
    }

}
```

## Tools Used

Manual review

## Recommended Mitigation Steps

Due to the limitation of the size of a uint256, it is not possible to get split it into 13 fairly random numbers as the first 6 digits will bias towards a lower value.
I suggest using only the last 72 digits of the uint256 whilst ignoring other digits. This would require also increasing the array size from 770 to 834 to ensure at least 10,000 entropies can be stored.
In addition, remove the modulo operation in `writeEntropyBatch1()`, `writeEntropyBatch2()`, and `writeEntropyBatch3()`

```diff
  function writeEntropyBatch1() public {
    require(lastInitializedIndex < batchSize1, 'Batch 1 already initialized.');

    uint256 endIndex = lastInitializedIndex + batchSize1; // calculate the end index for the batch
    unchecked {
      for (uint256 i = lastInitializedIndex; i < endIndex; i++) {
        uint256 pseudoRandomValue = uint256(
          keccak256(abi.encodePacked(block.number, i))
-       ) % uint256(10) ** 78; // generate a  pseudo-random value using block number and index
+       );
        require(pseudoRandomValue != 999999, 'Invalid value, retry.');
        entropySlots[i] = pseudoRandomValue; // store the value in the slots array
      }
    }
    lastInitializedIndex = endIndex;
  }

  // second batch initialization
  function writeEntropyBatch2() public {
    require(
      lastInitializedIndex >= batchSize1 && lastInitializedIndex < batchSize2,
      'Batch 2 not ready or already initialized.'
    );

    uint256 endIndex = lastInitializedIndex + batchSize1;
    unchecked {
      for (uint256 i = lastInitializedIndex; i < endIndex; i++) {
        uint256 pseudoRandomValue = uint256(
          keccak256(abi.encodePacked(block.number, i))
-       ) % uint256(10) ** 78;
+       );
        require(pseudoRandomValue != 999999, 'Invalid value, retry.');
        entropySlots[i] = pseudoRandomValue;
      }
    }
    lastInitializedIndex = endIndex;
  }

  // allows setting a specific entropy slot with a value
  function writeEntropyBatch3() public {
    require(
      lastInitializedIndex >= batchSize2 && lastInitializedIndex < maxSlotIndex,
      'Batch 3 not ready or already completed.'
    );
    unchecked {
      for (uint256 i = lastInitializedIndex; i < maxSlotIndex; i++) {
        uint256 pseudoRandomValue = uint256(
          keccak256(abi.encodePacked(block.number, i))
-       ) % uint256(10) ** 78;
+       );
        entropySlots[i] = pseudoRandomValue;
      }
    }
    lastInitializedIndex = maxSlotIndex;
  }
```


# Tokens can be forged an extra time beyond `forgePotential` due to an incorrect inequality check

## Links to affected code

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L86-L90
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L138-L144

## Bug Description

According to the [docs](https://docs.google.com/document/d/1pihtkKyyxobFWdaNU4YfAy56Q7WIMbFJjSHUAfRm6BA/edit) 

>"Forge Potential dictates how many times an Entity can forge before becoming infertile for the rest of the year (10% of the population is infertile from birth)"

The `EntityForging::listForForging()` and `EntityForging::forgeWithListed()` have checks to ensure that the respective token's `forgingCounts[tokenId]` does not exceed their `forgePotential`. The inequality check is incorrect as it does not revert when `forgingCounts[tokenId]` is equal to `forgePotential`.

```javascript
  function listForForging(
    uint256 tokenId,
    uint256 fee
  ) public whenNotPaused nonReentrant {

    ... SKIP!..

    uint8 forgePotential = uint8((entropy / 10) % 10); // Extract the 5th digit from the entropy
    require(
      forgePotential > 0 && forgingCounts[tokenId] <= forgePotential, // incorrect inequality
      'Entity has reached its forging limit'
    );

    ... SKIP!..
  }

  function forgeWithListed(
    uint256 forgerTokenId,
    uint256 mergerTokenId
  ) external payable whenNotPaused nonReentrant returns (uint256) {
    
    ... SKIP!..
    
    uint8 mergerForgePotential = uint8((mergerEntropy / 10) % 10); // Extract the 5th digit from the entropy
    forgingCounts[mergerTokenId]++;
    require(
      mergerForgePotential > 0 &&
        forgingCounts[mergerTokenId] <= mergerForgePotential, // incorrect inequality
      'forgePotential insufficient'
    );

    ... SKIP!..

}
```


## Impact
Tokens with a non-zero `forgePotential` can forge one extra time than expected.

## Proof of Concept
- Imagine a forger token has a `forgePotential` of 1 
- The said token forges once, forgingCounts[tokenId] is now equal to 1
- In the same year, this token is listed again for forging, this should revert because the token has reached it's `forgePotential`
- This call does not revert because `forgingCounts[tokenId]` == `forgePotential`

## Tools Used

Manual review

## Recommended Mitigation Steps

Require `forgingCounts[tokenId]` to be strictly less than forgePotential:

```diff
  function listForForging(
    uint256 tokenId,
    uint256 fee
  ) public whenNotPaused nonReentrant {

    ... SKIP!..

    uint8 forgePotential = uint8((entropy / 10) % 10); // Extract the 5th digit from the entropy
    require(
-     forgePotential > 0 && forgingCounts[tokenId] <= forgePotential, // incorrect inequality
+     forgePotential > 0 && forgingCounts[tokenId] < forgePotential,
      'Entity has reached its forging limit'
    );

    ... SKIP!..
  }

  function forgeWithListed(
    uint256 forgerTokenId,
    uint256 mergerTokenId
  ) external payable whenNotPaused nonReentrant returns (uint256) {
    
    ... SKIP!..
    
    uint8 mergerForgePotential = uint8((mergerEntropy / 10) % 10); // Extract the 5th digit from the entropy
    forgingCounts[mergerTokenId]++;
    require(
      mergerForgePotential > 0 &&
-       forgingCounts[mergerTokenId] <= mergerForgePotential, // incorrect inequality
+       forgingCounts[mergerTokenId] < mergerForgePotential,
      'forgePotential insufficient'
    );

    ... SKIP!..

}
```


# The `lastForgeResetTimestamp` mapping gets first set upon first listing or first forging, not token minting, this results in less forges than expected in first year after token minting

## Links to affected code

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L83
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L128-L129
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L199-L207

## Bug Description

According to the [docs](https://docs.google.com/document/d/1pihtkKyyxobFWdaNU4YfAy56Q7WIMbFJjSHUAfRm6BA/edit):

>"Forge Potential dictates how many times an Entity can forge before becoming infertile for the rest of the year "

However the `lastForgeResetTimestamp[tokenId]` variable gets set upon the first [listing](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L83) or [forging](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L128-L129), not minting. This results in unexpected behaviour for users.

There is no other place in the code where `lastForgeResetTimestamp[tokenId]` gets set, so to begin their 'first year of forging' they must forge.

## Impact
- Users unable to forge their tokens when they expect.
- As shown by the below example, a user may be able to only forge half the number of expected times in the first ~2 years since minting the token.

## Proof of Concept
Consider the following example where a user can only forge twice (but they have a `forgePotential` = 2) in the first ~23 months since minting.
- A user mints a token on January 1st 2024 which has `forgePotential` of 2, they expect to be able to forge twice in the 2024 and twice in the 2025 calendar years. 
- They forge this token twice on December 1st 2024
- They attempt to forge again on January 2nd 2025, but cannot until December 1st 2025.


## Tools Used

Manual Review

## Recommended Mitigation Steps
- Set the `lastForgeResetTimestamp[tokenId]` parameter upon minting a token
- Alternatively, the first time `_resetForgingCountIfNeeded()` is called, update `lastForgeResetTimestamp[tokenId]` to `nftContract::getTokenCreationTimestamp(tokenId)`:

```diff
  function _resetForgingCountIfNeeded(uint256 tokenId) private {
    uint256 oneYear = oneYearInDays;
    if (lastForgeResetTimestamp[tokenId] == 0) {
-     lastForgeResetTimestamp[tokenId] = block.timestamp;
+     lastForgeResetTimestamp[tokenId] = nftContract.getTokenCreationTimestamp(tokenId);
    } else if (block.timestamp >= lastForgeResetTimestamp[tokenId] + oneYear) {
      forgingCounts[tokenId] = 0; // Reset to the forge potential
      lastForgeResetTimestamp[tokenId] = block.timestamp;
    }
  }
```


# Using ERC721::transferFrom() instead of ERC721::safeTransfer() might cause the user's NFT to be frozen in if the receiver contract does not support ERC721 tokens

## Links to affected code
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityTrading/EntityTrading.sol#L81
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityTrading/EntityTrading.sol#L53

## Bug Description

The `EntityTrading` contract makes transfers ERC721 tokens to `msg.sender` in the `buyNft()` and `cancelListing()` functions. 

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityTrading/EntityTrading.sol#L81
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityTrading/EntityTrading.sol#L53

```javascript
  function buyNFT(uint256 tokenId) external payable whenNotPaused nonReentrant {
    ... SKIP!...

    nftContract.transferFrom(address(this), msg.sender, tokenId); // transfer NFT to the buyer

    ... SKIP!...
  }


  function cancelListing(uint256 tokenId) public whenNotPaused nonReentrant {
    ... SKIP!...

    nftContract.transferFrom(address(this), msg.sender, tokenId); // transfer the nft back to seller

    ... SKIP!...
  }
```

There are no checks in the code to ensure the receive of the tokens are not smart contracts.

If the `msg.sender` happens to be a contract which has not implemented the `onERC721Received()` function, the NFT will be [stuck in the contract](https://eips.ethereum.org/EIPS/eip-721).

>"A wallet/broker/auction application MUST implement the wallet interface if it will accept safe transfers."


## Impact
Tokens can be stuck in contracts due to the use of `.transferFrom()` instead of `.safeTransferFrom()`.


## Tools Used

Manual Review

## Recommended Mitigation Steps

Use `.safeTransferFrom()` instead of `.transferFrom()`:


safeTransferFrom
```diff
  function buyNFT(uint256 tokenId) external payable whenNotPaused nonReentrant {
    ... SKIP!...

-   nftContract.transferFrom(address(this), msg.sender, tokenId); // transfer NFT to the buyer
+   nftContract.safeTransferFrom(address(this), msg.sender, tokenId); // transfer NFT to the buyer

    ... SKIP!...
  }


  function cancelListing(uint256 tokenId) public whenNotPaused nonReentrant {
    ... SKIP!...

-    nftContract.transferFrom(address(this), msg.sender, tokenId); // transfer the nft back to seller
+    nftContract.safeTransferFrom(address(this), msg.sender, tokenId); // transfer the nft back to seller

    ... SKIP!...
  }
```



# Low / QA issues

# Zero address check missing in `TraitForgeNft::setNukeFundContract()`

This is inconsistant with `setEntityForgingContract()`, `setEntropyGenerator()`, and `setAirdropContract()` from the same contract which all revert if the zero address is provided.

```javascript
  function setNukeFundContract(
    address payable _nukeFundAddress
  ) external onlyOwner {
    nukeFundAddress = _nukeFundAddress;
    emit NukeFundContractUpdated(_nukeFundAddress);
  }

  // Function to set the entity merging (breeding) contract address, restricted to the owner
  function setEntityForgingContract(
    address entityForgingAddress_
  ) external onlyOwner {
@>  require(entityForgingAddress_ != address(0), 'Invalid address');

    entityForgingContract = IEntityForging(entityForgingAddress_);
  }

  function setEntropyGenerator(
    address entropyGeneratorAddress_
  ) external onlyOwner {
@>  require(entropyGeneratorAddress_ != address(0), 'Invalid address');

    entropyGenerator = IEntropyGenerator(entropyGeneratorAddress_);
  }

  function setAirdropContract(address airdrop_) external onlyOwner {
@>  require(airdrop_ != address(0), 'Invalid address');

    airdropContract = IAirdrop(airdrop_);
  }
```


# In `TraitForgeNft::mintWithBudget()`, the `amountMinted` variable is initialized and updated but not used

Consider removing this variable as it is a waste of gas to updating it in memory without serving any purpose.

```javascript
  function mintWithBudget( // Allows a user to mint several NFTs until a provided mint budget is reached
    bytes32[] calldata proof
  )
    public
    payable
    whenNotPaused
    nonReentrant
    onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
  {
    uint256 mintPrice = calculateMintPrice();
@>  uint256 amountMinted = 0;
    uint256 budgetLeft = msg.value; 

    while (budgetLeft >= mintPrice && _tokenIds < maxTokensPerGen) {
      _mintInternal(msg.sender, mintPrice);
@>    amountMinted++;
      budgetLeft -= mintPrice;
      mintPrice = calculateMintPrice();
    }
    if (budgetLeft > 0) {
      (bool refundSuccess, ) = msg.sender.call{ value: budgetLeft }('');
      require(refundSuccess, 'Refund failed.');
    }
  }
```


# Owner is able to update `TraitForgeNft::maxGeneration` to the same generation which is a waste of gas

In `TraitForgeNft::setMaxGeneration()` consider requiring `maxGeneration_` to strictly exceed `currentGeneration` to avoid writing to storage for no reason.

```javascript

  function setMaxGeneration(uint maxGeneration_) external onlyOwner {
    require(
@>    maxGeneration_ >= currentGeneration,
      "can't below than current generation"
    );
    maxGeneration = maxGeneration_;
  }
```


# Remove unused input argument in `TraitForgeNft::forge()`

```javascript
  function forge(
    address newOwner,
    uint256 parent1Id,
    uint256 parent2Id,
@>  string memory 
  ) external whenNotPaused nonReentrant returns (uint256) {

    ... SKIP!...

    }
```


# It is recommended to standardize return variable declarations for consistancy

Some functions explicitly declare the return variable with a `return` function, others implicitly return variables by declaring it initially. Standardization is recommended for readability and consistancy.

Example 1:
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L227C1-L232C4
```javascript
  function calculateMintPrice() public view returns (uint256) {
    uint256 currentGenMintCount = generationMintCounts[currentGeneration];
    uint256 priceIncrease = priceIncrement * currentGenMintCount;
    uint256 price = startPrice + priceIncrease;
    return price;
  }
```

Example 2:
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L246-L252
```javascript
  function getEntropiesForTokens(
    uint256 forgerTokenId,
    uint256 mergerTokenId
  ) public view returns (uint256 forgerEntropy, uint256 mergerEntropy) {
    forgerEntropy = getTokenEntropy(forgerTokenId);
    mergerEntropy = getTokenEntropy(mergerTokenId);
  }
```


# Simplify `TraitForgeNft::isForger()` to save gas by avoiding writing to memory

```diff
  function isForger(uint256 tokenId) public view returns (bool) {
-   uint256 entropy = tokenEntropy[tokenId];
-   uint256 roleIndicator = entropy % 3;
+   return tokenEntropy[tokenId] % 3 == 0;
-    return roleIndicator == 0;
  }
```


# Inconsistancy between calculation of `forgePotential` parameter between functions

Consider the below two functions from the `EntityForging` contract, which calculate `forgePotential` as the 5th digit from the entropy. Note how these definitions differ from `EntropyGenerator::deriveTokenParameters()`.

```javascript
  function listForForging(
    uint256 tokenId,
    uint256 fee
  ) public whenNotPaused nonReentrant {
    ... SKIP!...

    uint8 forgePotential = uint8((entropy / 10) % 10); // Extract the 5th digit from the entropy
    
    ... SKIP!...
  }

  function forgeWithListed(
    uint256 forgerTokenId,
    uint256 mergerTokenId 
  ) external payable whenNotPaused nonReentrant returns (uint256) {
    ... SKIP!...

    uint8 mergerForgePotential = uint8((mergerEntropy / 10) % 10); // Extract the 5th digit from the entropy
    
    ... SKIP!...
  }
```

```javascript
  function deriveTokenParameters(
    uint256 slotIndex,
    uint256 numberIndex
  )
    public
    view
    returns (
      uint256 nukeFactor,
      uint256 forgePotential,
      uint256 performanceFactor,
      bool isForger
    )
  {
    ... SKIP!...

    forgePotential = getFirstDigit(entropy); // inconsistancy between forgePotential here and in the `EntityForging::listForForging()
    
    ... SKIP!...
  }
```



# Remove redundant `require()` statement in `EntityTrading::buyNFT()` to save gas

The second `require()` statement below can never revert if the first `require()` statement does not. This is because the `listing` variable needs to exist for the first `require()` statement to pass, and the listing cannot be created with a zero address.

```javascript
  function buyNFT(uint256 tokenId) external payable whenNotPaused nonReentrant {

    Listing memory listing = listings[listedTokenIds[tokenId]];
    require(
      msg.value == listing.price, 
      'ETH sent does not match the listing price'
    );
@>  require(listing.seller != address(0), 'NFT is not listed for sale.'); // redundant check

    ... SKIP!...
  }
```


# In the `NukeFund` contract, `nukeFactorMaxParam` should be a function of `maxAllowedClaimDivisor` but they have individual setter functions

This opens up the possibility of a mistake in setting these values resulting in unexpected nuke payouts from the `NukeFund`. For example if `setMaxAllowedClaimDivisor()` is called without calling `setNukeFactorMaxParam()` the maximum amount claimable from the `NukeFund` would behave unexpectedly.

Currently there are two separate setter functions for these state variables, it is recommended one of these is removed and is derived when the `maxAllowedClaimDivisor` is set. You may want to also consider changing the input to `setMaxAllowedClaimDivisor()` to be a uint8 or adding a require state to ensure the input contains reasonable values.

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L75-L81
```diff
  function setMaxAllowedClaimDivisor(uint256 value) external onlyOwner {
    maxAllowedClaimDivisor = value;
+   nukeFactorMaxParam = MAX_DENOMINATOR / maxAllowedClaimDivisor;
  }

- function setNukeFactorMaxParam(uint256 value) external onlyOwner {
-   nukeFactorMaxParam = value;
- }
```