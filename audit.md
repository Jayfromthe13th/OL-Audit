# Arbitrary Lock Duration Vulnerability in Lockbox Module: Security Audit Report




## Audit Summary

This report outlines the findings from a comprehensive security audit conducted for **StakeSphere**. The audit targeted the project's smart contract suite with the objective of identifying and mitigating potential vulnerabilities, thereby strengthening the security and robustness of the project's blockchain infrastructure.

## Findings Summary

The audit revealed findings categorized severity levels. Recommendations for remediation are provided to address these vulnerabilities effectively.

### Critical Findings

#### Arbitrary Lock Duration 

**Description:**  

The Lockbox module in the Slow Wallet v2.0 implementation allows users to lock assets for a specified period. However, a vulnerability exists where the DEFAULT LOCK DURATION constant is not enforced, allowing users to create lockboxes with arbitrary durations. This oversight can be exploited by malicious actors to create lockboxes with very short durations, leading to inflated reward distributions.The reward distribution mechanism in the Lockbox module is based on the duration of the lockbox. Specifically, the rewards are calculated using a daily drip mechanism, which gradually unlocks assets over time. The calculation of the daily drip is inversely proportional to the lock duration, meaning shorter durations result in higher daily rewards.
Here's a breakdown of the calculation issue:

1. Daily Drip Calculation: The daily drip value is calculated using the formula:
daily_drip_value=total_locked_value /duration_in_days
where duration_in_days is derived from the lock duration in months.

2. Short Duration Exploit: If a user sets a very short duration (e.g., 1 day), the duration_in_days becomes very small, leading to a large daily_drip_value. This results in a higher proportion of the locked value being unlocked daily.

3. Reward Inflation: By creating multiple lockboxes with short durations, a malicious actor can repeatedly receive high daily rewards, inflating their total rewards significantly compared to the intended design.

**POC:**  
``` rust
  #[test(framework = @0x1, bob_addr = @0x10002)]
    fun test_exploit(framework: &signer, bob_addr: address) acquires SlowWalletV2 {
        // Setup test environment
        let bob_sig = account::create_account_for_test(bob_addr);
        let coin = test_setup(framework, 1000); // Mint 1000 coins for testing

        // Execute the exploit
        exploit_short_duration(&bob_sig, coin);

        // Verify the exploit by checking the balance
        let (found, idx) = idx_by_duration(bob_addr, 1); // Check for the short duration lockbox
        assert!(found, 7357001); // Ensure the lockbox was created
        let balance = user_balance(bob_addr);
        assert!(balance > 1000, 7357002); // Check if the balance is inflated
    }
}
```

1. Exploit Function: The exploit_short_duration function simulates the creation of a lockbox with a very short duration (e.g., 1 day). This duration is not part of the predefined LOCK_DURATIONS, allowing the malicious actor to exploit the reward calculation.
   
2. Test Function: The test_exploit function sets up a test environment, creates a lockbox with the short duration, and verifies the exploit by checking if the balance is inflated.

3. Assertions: The test includes assertions to ensure that the lockbox was created and that the balance is greater than the initial amount, indicating that the rewards have been inflated.

 **Recommendation:**  
To remediate this vulnerability, enforce the use of predefined lock durations by validating the duration against the LOCK_DURATIONS constant before allowing the creation of a lockbox. This ensures that only valid durations are used, preventing the creation of lockboxes with arbitrary durations.

``` rust
 public(friend) fun new(locked_coins: Coin<LibraCoin>, duration_type: u64): Lockbox {
    // Validate that the duration is in the allowed list
    assert!(is_valid_duration(duration_type), error::invalid_argument(EINVALID_DURATION));
    let (midnight, _) = date::todays_start_seconds();
    Lockbox {
      locked_coins,
      duration_type,
      lifetime_deposited: 0,
      lifetime_unlocked: 0,
      last_unlock_timestamp: midnight,
    }
}

```
