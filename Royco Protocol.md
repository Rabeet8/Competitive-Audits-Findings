<!DOCTYPE html>
<html>
<body>

<div class="full-page">
    <div>
    <h1>Royco Protocol Audit Report</h1>
    <h3>Prepared by: Syed Rabeet</h3>
    <h5>Oct 23th,2024</h5>
    </div>

</div>

</body>
</html>


# Disclaimer

Syed Rabeet makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

# Protocol Summary

Royco allows anyone to create a market around any onchain action. Those who wish to pay users to execute an onchain action are called "Incentive Providers" (IPs) and offer token or points incentives in exchange for an "Action Provider" (APs) to take some action, be it enter a staking vault, or execute some "recipe" of one or more smart contract interactions, each having their own market types on Royco.


# Executive Summary

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 1                      |
| Medium   | 0                      |
| Low      | 0                      |
| Info     | 0                      |
| Total    | 1                      |

# Findings

### [H-1] Inability to Claim Available Rewards Due to Insufficient Balance in One Token


**Summary:**

The ERC4626i contract's `_claim::ERC4626i`  function fails to process rewards independently for multiple tokens, leading to a situation where the failure to claim one reward token prevents the claiming of other available reward tokens.

**Description:** 

In the current implementation of the `_claim::ERC4626i` function, if the contract lacks sufficient balance for one reward token, it prevents claiming all subsequent reward tokens. This occurs because the function doesn't handle each reward token independently, causing the claim process to fail completely when it encounters an issue with any single token.

```javascript

 function _claim(address reward, address from, address to, uint256 amount) internal virtual {
        _updateUserRewards(reward, from);
        rewardToUserToAR[reward][from].accumulated -= amount.toUint128();
        pushReward(reward, to, amount);
        emit Claimed(reward, from, to, amount);
    }

```

**Impact Explanation:**

This vulnerability can result in users being unable to claim rewards they've rightfully earned. If the contract runs out of one reward token, users lose access to all their accrued rewards across all tokens until the depleted token is replenished. This not only impacts user experience negatively but also locks up rewards, potentially for extended periods.

**Likelihood Explanation**

Add this test case to your ERC4626i.t.sol file.

**Objective:**

Test the behavior of the testIncentivizedVault when it has insufficient balance for one of the reward tokens during a user claim.

**Setup:**

1) Two reward tokens (rewardToken1 and rewardToken2) are configured with specific reward intervals and amounts.

2) The user deposits tokens into the vault and begins accruing rewards.

**Simulated Scenario:**

1) Time is advanced to the end of the reward interval to simulate the full accrual of rewards.

2) The contract's balance of rewardToken1 is deliberately depleted to simulate insufficient funds.

**Claim Attempt:**

1) The user attempts to claim their rewards.
2) The test expects the claim transaction to revert due to the contract's inability to pay out rewardToken1.

**Verification:**

The test ensures that no rewards are claimed when the vault lacks sufficient balance for one of the reward tokens.

In summary, if the contract lacks the balance for token A, the user is unable to claim rewards for token B,

```javascript
   function testVulnerableClaimFunction() public {
    // Setup
    uint256 initialRewards1 = 1000e18;
    uint256 initialRewards2 = 500e18;
    uint256 initialStart = uint32(block.timestamp);
    uint256 initialEnd = initialStart + 30 days;

    testIncentivizedVault.addRewardsToken(address(rewardToken1));
    testIncentivizedVault.addRewardsToken(address(rewardToken2));

    rewardToken1.mint(address(this), initialRewards1);
    rewardToken2.mint(address(this), initialRewards2);

    rewardToken1.approve(address(testIncentivizedVault), initialRewards1);
    rewardToken2.approve(address(testIncentivizedVault), initialRewards2);

    testIncentivizedVault.setRewardsInterval(address(rewardToken1), initialStart, initialEnd, initialRewards1, address(this));
    testIncentivizedVault.setRewardsInterval(address(rewardToken2), initialStart, initialEnd, initialRewards2, address(this));

    // User deposits and accrues rewards
    uint256 depositAmount = 100e18;
    MockERC20(address(token)).mint(REGULAR_USER, depositAmount);
    vm.startPrank(REGULAR_USER);
    token.approve(address(testIncentivizedVault), depositAmount);
    testIncentivizedVault.deposit(depositAmount, REGULAR_USER);
    vm.stopPrank();

    // Advance time to accrue rewards
    vm.warp(initialEnd);

    // Simulate a scenario where the contract doesn't have enough of rewardToken2
    uint256 contractBalance = rewardToken1.balanceOf(address(testIncentivizedVault));
    vm.prank(address(testIncentivizedVault));
    rewardToken1.transfer(address(this), contractBalance); // Remove all rewardToken1 from the contract

    // User tries to claim rewards
    vm.prank(REGULAR_USER);
    vm.expectRevert(); // Expect the transaction to revert due to insufficient rewardToken2
    testIncentivizedVault.claim(REGULAR_USER);

    // Verify that the user couldn't claim any rewards, including rewardToken2
    uint256 rewardForTokenA = testIncentivizedVault.currentUserRewards(address(rewardToken1), REGULAR_USER);
    uint256 rewardForTokenB = testIncentivizedVault.currentUserRewards(address(rewardToken2), REGULAR_USER);
    console.log(rewardForTokenA, "rewardForTokenA");
    console.log(rewardForTokenB, "rewardForTokenB");


    assertGt(testIncentivizedVault.currentUserRewards(address(rewardToken1), REGULAR_USER), 0);
    assertGt(testIncentivizedVault.currentUserRewards(address(rewardToken2), REGULAR_USER), 0);
}

```


**Recommended Mitigation:**
Made these changes to the `_claim` function and pasted it into the `ERC4626i.sol` file

```javascript

//Add this event at the top of the file

 event ClaimFailed(address reward, address user, address receiver, uint256 unclaimedAmount);


```

```javascript
  function _claim(address reward, address from, address to, uint256 amount) internal virtual {
    _updateUserRewards(reward, from);
    uint256 claimableAmount = amount;

    if (POINTS_FACTORY.isPointsProgram(reward)) {
        Points(reward).award(to, claimableAmount);
    } else {
        uint256 contractBalance = ERC20(reward).balanceOf(address(this));
        if (contractBalance < claimableAmount) {
            claimableAmount = contractBalance;
        }
        if (claimableAmount > 0) {
            ERC20(reward).safeTransfer(to, claimableAmount);
        }
    }

    rewardToUserToAR[reward][from].accumulated -= claimableAmount.toUint128();
    emit Claimed(reward, from, to, claimableAmount);

    if (claimableAmount < amount) {
        emit ClaimFailed(reward, from, to, amount - claimableAmount);
    }
}
```

Add this test case to the ERC4626i.t.sol file to ensure that the provided recommendation works correctly and user is able to claim reward for token2 even if the transaction for token1 fails.

```javascript
function testClaimSecondRewardWhenFirstFails() public {
    // Setup
    uint256 initialRewards1 = 1000e18;
    uint256 initialRewards2 = 500e18;
    uint256 initialStart = block.timestamp;
    uint256 initialEnd = initialStart + 30 days;

    testIncentivizedVault.addRewardsToken(address(rewardToken1));
    testIncentivizedVault.addRewardsToken(address(rewardToken2));

    rewardToken1.mint(address(this), initialRewards1);
    rewardToken2.mint(address(this), initialRewards2);

    rewardToken1.approve(address(testIncentivizedVault), initialRewards1);
    rewardToken2.approve(address(testIncentivizedVault), initialRewards2);

    testIncentivizedVault.setRewardsInterval(address(rewardToken1), initialStart, initialEnd, initialRewards1, address(this));
    testIncentivizedVault.setRewardsInterval(address(rewardToken2), initialStart, initialEnd, initialRewards2, address(this));

    // User deposits and accrues rewards
    uint256 depositAmount = 100e18;
    MockERC20(address(token)).mint(REGULAR_USER, depositAmount);
    vm.startPrank(REGULAR_USER);
    token.approve(address(testIncentivizedVault), depositAmount);
    testIncentivizedVault.deposit(depositAmount, REGULAR_USER);
    vm.stopPrank();

    // Advance time to accrue rewards
    vm.warp(initialEnd);

    // Calculate expected rewards
    uint256 expectedReward1 = testIncentivizedVault.currentUserRewards(address(rewardToken1), REGULAR_USER);
    uint256 expectedReward2 = testIncentivizedVault.currentUserRewards(address(rewardToken2), REGULAR_USER);

    // Simulate a scenario where the contract doesn't have enough of rewardToken1
    uint256 contractBalance = rewardToken1.balanceOf(address(testIncentivizedVault));
    vm.prank(address(testIncentivizedVault));
    rewardToken1.transfer(address(this), contractBalance); // Remove all rewardToken1 from the contract

    // User tries to claim rewards
    vm.prank(REGULAR_USER);
    testIncentivizedVault.claim(REGULAR_USER);

    // Verify that the user couldn't claim rewardToken1 but successfully claimed rewardToken2
    assertEq(rewardToken1.balanceOf(REGULAR_USER), 0, "User should not have received any rewardToken1");
    assertEq(rewardToken2.balanceOf(REGULAR_USER), expectedReward2, "User should have received all of rewardToken2");

    // Verify that the unclaimed rewards for rewardToken1 are still recorded
    uint256 remainingReward1 = testIncentivizedVault.currentUserRewards(address(rewardToken1), REGULAR_USER);
    assertEq(remainingReward1, expectedReward1, "Unclaimed rewardToken1 should still be recorded");

    // Verify that rewardToken2 has been fully claimed
    uint256 remainingReward2 = testIncentivizedVault.currentUserRewards(address(rewardToken2), REGULAR_USER);
    assertEq(remainingReward2, 0, "RewardToken2 should have been fully claimed");

    

}

