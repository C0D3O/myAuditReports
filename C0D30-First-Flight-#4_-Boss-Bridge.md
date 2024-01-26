# First Flight #4: Boss Bridge - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)


- ## Low Risk Findings
    - ### [L-01. no check for existing token in TokenFactory deployToken function - all relative assets loss if token already exists](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #4

### Dates: Nov 9th, 2023 - Nov 15th, 2023

[See more contest details here](https://www.codehawks.com/contests/clomptuvr0001ie09bzfp4nqw)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 0
   - Low: 1



		


# Low Risk Findings

## <a id='L-01'></a>L-01. no check for existing token in TokenFactory deployToken function - all relative assets loss if token already exists            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/dad104a9f481aace15a550cf3113e81ad6bdf061/src/TokenFactory.sol#L23-L27

## Summary

Adding token with the symbol that already exists will overwrite the address of the existing one, which leads to all token's assets loss


## Vulnerability Details

Before adding a new token, the function should check if a token with the same symbol exists, otherwise it will overwrite the address of existing token leading to all token assets loss for everyone.


## Impact

Foundry test 
* deploying a token
* sending 1 ether tokens to Alice
* deploying another token with the same symbol
* assigning it to the previous token variable
* checking Alice balance to be 1 ether
* TEST FAILS - Alice has 0 balance

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import { Test } from "forge-std/Test.sol";
import { TokenFactory } from "../src/scope/TokenFactory.sol";
import { L1Token } from "../src/scope/L1Token.sol";

contract TokenFactoryTest is Test {
	TokenFactory tokenFactory;
	address owner = makeAddr("owner");
	address alice = makeAddr("Alice");
	L1Token theToken;

	function setUp() public {
		vm.prank(owner);
		tokenFactory = new TokenFactory();
	}

	function testAddToken() public {
		vm.prank(owner);
		address tokenAddress = tokenFactory.deployToken("Test", type(L1Token).creationCode);
		theToken = L1Token(tokenAddress);

		vm.prank(address(tokenFactory));
		theToken.transfer(alice, 1e18);

		vm.prank(owner);
		address tokenAddress2 = tokenFactory.deployToken("Test", type(L1Token).creationCode);
		theToken = L1Token(tokenAddress2);

		// TEST FAILS - ALice has 0 balance
		assertEq(theToken.balanceOf(alice), 1e18);
	}
}

```

## Recommendations

add check for existing symbol in the mapping

```diff
function deployToken(string memory symbol, bytes memory contractBytecode) public onlyOwner returns (address addr) {
+		require(s_tokenToAddress[symbol] == address(0), "Token exists");
		assembly {
			addr := create(0, add(contractBytecode, 0x20), mload(contractBytecode))
		}
		s_tokenToAddress[symbol] = addr;
		emit TokenDeployed(symbol, addr);
	}
```



