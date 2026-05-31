# -Egg-Empire
 Egg Empire
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract BaseEggEmpire is ERC721, Ownable {
    uint256 public nextTokenId = 1;
    uint256 public constant MAX_PETS_PER_WALLET = 10;

    enum Rarity { COMMON, RARE, EPIC, LEGENDARY }

    struct Pet {
        Rarity rarity;
        uint8 level;
        uint256 birthTime;
        uint256 lastFeedTime;
        uint256 strength;
    }

    mapping(uint256 => Pet) public pets;

    uint256 public constant HATCH_PRICE = 0.0008 ether;
    uint256 public constant FEED_PRICE = 0.0003 ether;

    event EggHatched(address owner, uint256 tokenId, Rarity rarity);
    event PetFed(uint256 tokenId, uint8 newLevel);
    event PetsFused(uint256 newTokenId, uint256 pet1, uint256 pet2);

    constructor() ERC721("Base Egg Empire", "BEE") Ownable(msg.sender) {}

    function hatchEgg() external payable {
        require(msg.value == HATCH_PRICE, "Must pay exact 0.0008 ETH");
        require(balanceOf(msg.sender) < MAX_PETS_PER_WALLET, "Maximum pets reached");

        uint256 tokenId = nextTokenId++;
        _mint(msg.sender, tokenId);

        uint256 rand = uint256(keccak256(abi.encodePacked(block.timestamp, msg.sender, tokenId))) % 100;

        Rarity rarity = Rarity.COMMON;
        if (rand > 92) rarity = Rarity.LEGENDARY;
        else if (rand > 75) rarity = Rarity.EPIC;
        else if (rand > 40) rarity = Rarity.RARE;

        pets[tokenId] = Pet({
            rarity: rarity,
            level: 1,
            birthTime: block.timestamp,
            lastFeedTime: block.timestamp,
            strength: uint8(rarity) * 8 + 12
        });

        emit EggHatched(msg.sender, tokenId, rarity);
    }

    function feedPet(uint256 tokenId) external payable {
        require(ownerOf(tokenId) == msg.sender, "Not your pet");
        require(msg.value == FEED_PRICE, "Must pay exact 0.0003 ETH");

        Pet storage pet = pets[tokenId];
        require(block.timestamp >= pet.lastFeedTime + 1 hours, "Feed cooldown active");

        pet.lastFeedTime = block.timestamp;
        pet.level++;
        pet.strength += 6 + uint256(pet.rarity) * 3;

        // Rare evolution chance every 5 levels
        if (pet.level % 5 == 0 && uint256(keccak256(abi.encodePacked(block.timestamp, tokenId))) % 3 == 0) {
            if (uint256(pet.rarity) < 3) {
                pet.rarity = Rarity(uint256(pet.rarity) + 1);
            }
        }

        emit PetFed(tokenId, pet.level);
    }

    function fusePets(uint256 pet1, uint256 pet2) external {
        require(ownerOf(pet1) == msg.sender && ownerOf(pet2) == msg.sender, "Not your pets");
        require(pet1 != pet2, "Cannot fuse the same pet");

        Pet storage p1 = pets[pet1];
        Pet storage p2 = pets[pet2];

        uint256 newTokenId = nextTokenId++;
        _mint(msg.sender, newTokenId);

        Rarity newRarity = p1.rarity > p2.rarity ? p1.rarity : p2.rarity;

        pets[newTokenId] = Pet({
            rarity: newRarity,
            level: (p1.level + p2.level) / 2 + 3,
            birthTime: block.timestamp,
            lastFeedTime: block.timestamp,
            strength: (p1.strength + p2.strength) * 13 / 10
        });

        _burn(pet1);
        _burn(pet2);

        emit PetsFused(newTokenId, pet1, pet2);
    }

    function getPet(uint256 tokenId) external view returns (Pet memory) {
        return pets[tokenId];
    }

    function withdraw() external onlyOwner {
        (bool success, ) = payable(owner()).call{value: address(this).balance}("");
        require(success, "Withdraw failed");
    }
}
