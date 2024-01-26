# First Flight #6: Voting Booth - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Rewards formula mistake - a big part of ETH rewards is stuck in the contract forever](#H-01)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #6

### Dates: Dec 15th, 2023 - Dec 22nd, 2023

[See more contest details here](https://www.codehawks.com/contests/clq5cx9x60001kd8vrc01dirq)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 0
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Rewards formula mistake - a big part of ETH rewards is stuck in the contract forever            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-Voting-Booth/blob/a3241b1c530529a4f739f9462f720e8561ad45ca/src/VotingBooth.sol#L172

## Summary

Due to the mistake in the rewards formula, a big part of ETH will be stuck in the contract forever.

## Vulnerability Details
This lines introduces the issue 
`uint256 totalVotes = totalVotesFor + totalVotesAgainst;`

If a voting is approved, and there was some votes against it, they will be summed with the total for-votes, but rewards will be sent only to users who voted for. For example, the reward pool is 1 ETH, there is 5 voters total, 2 votes for, 1 against, 1 ETH will be divided by 3, BUT will be sent only to 2 voters for, leaving the 1/3 of total ETH stuck in the contract forever, as it can be used only once.

The issue is present in every scenario, where a voting is accepted and there's some votes against.

## Impact
here's the test
<details>
 <summary>The Test</summary>

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import {VotingBooth} from "../src/scope/VotingBooth.sol";
import {Test} from "forge-std/Test.sol";

import {console} from "forge-std/console.sol";

contract Fuzz is Test {
	// eth reward
	uint256 constant ETH_REWARD = 1e18;
	// allowed voters
	address[] voters;

	// contracts required for test
	VotingBooth booth;

	function setUp() public virtual {
		// deal this contract the proposal reward
		deal(address(this), ETH_REWARD);

		// setup the allowed list of voters
		voters.push(address(0x1));
		voters.push(address(0x2));
		voters.push(address(0x3));
		voters.push(address(0x4));
		voters.push(address(0x5));

		// setup contract to be tested
		booth = new VotingBooth{value: ETH_REWARD}(voters);
		
        // verify setup
		//
		// proposal has rewards
		assert(address(booth).balance == ETH_REWARD);
		// proposal is active
		assert(booth.isActive());
		// proposal has correct number of allowed voters
		assert(booth.getTotalAllowedVoters() == voters.length);
		// this contract is the creator
		assert(booth.getCreator() == address(this));
	}

	// required to receive refund if proposal fails
	receive() external payable {}

	function testVotePassesAndMoneyIsStuck() public {
		vm.prank(address(0x1));
		booth.vote(true);

		vm.prank(address(0x2));
		booth.vote(true);

		vm.prank(address(0x3));
		booth.vote(false);
		
		console.log("BOOTH BALANCE", address(booth).balance);
		assert(!booth.isActive() && address(booth).balance > 0);
	}
}
```
You can see ETH left in the contract and that test passes, the contract's balance is not 0

</details>

## Tools Used

foundry

## Recommendations
I guess just deleting the votesAgainst from the formula would be enough.

<details>
<summary>Mitigations</summary>

```diff
function _distributeRewards() private {
	// get number of voters for & against
	uint256 totalVotesFor = s_votersFor.length;
	uint256 totalVotesAgainst = s_votersAgainst.length;
-   uint256 totalVotes = totalVotesFor + totalVotesAgainst;
	// rewards to distribute or refund. This is guaranteed to be
	// greater or equal to the minimum funding amount by a check
	// in the constructor, and there is intentionally by design
	// no way to decrease or increase this amount. Any findings
	// related to not being able to increase/decrease the total
	// reward amount are invalid
	uint256 totalRewards = address(this).balance;

	// if the proposal was defeated refund reward back to the creator
	// for the proposal to be successful it must have had more `For` votes
	// than `Against` votes
	if (totalVotesAgainst >= totalVotesFor) {
		// proposal creator is trusted to create a proposal from an address
		// that can receive ETH. See comment before declaration of `s_creator`
		_sendEth(s_creator, totalRewards);
	}
	// otherwise the proposal passed so distribute rewards to the `For` voters
	else {
-       uint256 rewardPerVoter = totalRewards / totalVotes;
+		uint256 rewardPerVoter = totalRewards / totalVotesFor;

		for (uint256 i; i < totalVotesFor; ++i) {
			// proposal creator is trusted when creating allowed list of voters,
			// findings related to gas griefing attacks or sending eth
			// to an address reverting thereby stopping the reward payouts are
			// invalid. Yes pull is to be preferred to push but this
			// has not been implemented in this simplified version to
			// reduce complexity & help you focus on finding the
			// harder to find bug

			// if at the last voter round up to avoid leaving dust; this means that
			// the last voter can get 1 wei more than the rest - this is not
			// a valid finding, it is simply how we deal with imperfect division
			if (i == totalVotesFor - 1) {
				rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotes, Math.Rounding.Ceil);
			}
			_sendEth(s_votersFor[i], rewardPerVoter);
		}
	}
}

```
</details>
		





