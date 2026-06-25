# Committee Reorg Fix - Quick Start Guide

## Overview

This guide helps you understand and use the committee reorganization fix that resolves attestation verification failures during mid-epoch validator set changes.

## What Problem Does This Fix?

When validators exit or join mid-epoch, the committee composition changes, creating different committee roots. Previously, attestations created before the change would fail verification after the change, even for honest validators. This fix allows attestations to verify correctly during the transition period.

## How to Use

### Basic Usage Example

```rust
use sorosusu_contracts::validator::committee_assignment::CommitteeAssignment;
use sorosusu_contracts::validator::committee_assignment::CommitteeView;
use sorosusu_contracts::attestation::verifier::verify_attestation_with_committee_view;

// Initialize committee with validators
let mut assignment = CommitteeAssignment::new(vec![10, 20, 30, 40]);

// Get committee view for normal operation
let view = assignment.get_committee_view(slot);
match view {
    CommitteeView::Stable(root) => {
        // Single root - normal operation
        println!("Committee root: {:?}", root);
    }
    CommitteeView::Ambiguous { old_root, new_root } => {
        // Dual roots - during reorg
        println!("Old root: {:?}, New root: {:?}", old_root, new_root);
    }
}
```

### Handling Validator Exit Mid-Epoch

```rust
// Validator 40 exits irregularly at slot 3203
let exit_slot = 3203;

// Step 1: Trigger reorg (captures current state as "old")
assignment.trigger_reorg(exit_slot);

// Step 2: Update validator set (validator 40 exits, 50 joins)
assignment.update_validator_set(vec![10, 20, 30, 50]);

// Step 3: Verify attestations during reorg window (slots 3203-3206)
let view = assignment.get_committee_view(exit_slot + 1);

// Attestations with either old or new root will verify
let result = verify_attestation_with_committee_view(
    &bitfield,
    &keys,
    &domain,
    &data,
    &signatures,
    &view,
    &committee_root,  // Can be old OR new root
);

// Step 4: Finalize reorg after window closes (slot 3207+)
assignment.finalize_reorg(exit_slot + 4);

// Now only new root is accepted
let stable_view = assignment.get_committee_view(exit_slot + 5);
assert!(matches!(stable_view, CommitteeView::Stable(_)));
```

### Using Committee Cache

```rust
use sorosusu_contracts::db::committee_cache::CommitteeCache;

// Create cache with default capacity (256 epochs)
let mut cache = CommitteeCache::new();

// Store stable committee root for an epoch
cache.store_stable(epoch, committee_root);

// Store ambiguous root during reorg
cache.store_ambiguous(epoch, old_root, new_root, reorg_end_slot);

// Retrieve committee view for attestation verification
let view = cache.get_committee_view(epoch, current_slot);

// Finalize reorg in cache
cache.finalize_reorg(epoch, finalization_slot);
```

## API Reference

### CommitteeAssignment

#### Methods

- **`new(validator_indices: Vec<u64>) -> Self`**
  - Creates new committee assignment with given validators

- **`trigger_reorg(slot: u64)`**
  - Initiates a reorganization at specified slot
  - Captures current validator set as "old"
  - Sets up 4-slot reorg window

- **`update_validator_set(new_indices: Vec<u64>)`**
  - Updates to new validator set after reorg trigger

- **`finalize_reorg(current_slot: u64)`**
  - Finalizes reorg after window closes
  - Transitions to stable state with new root only

- **`get_committee_view(slot: u64) -> CommitteeView`**
  - Returns appropriate committee view for given slot
  - `Stable` during normal operation
  - `Ambiguous` during reorg window

- **`validator_indices() -> &[u64]`**
  - Returns current validator indices

### CommitteeView

#### Variants

- **`Stable(Hash256)`**
  - Normal operation with single committee root
  
- **`Ambiguous { old_root: Hash256, new_root: Hash256 }`**
  - During reorg with both roots valid

#### Methods

- **`matches(candidate: &Hash256) -> bool`**
  - Checks if candidate root matches this view
  - For `Stable`: matches only the single root
  - For `Ambiguous`: matches either old or new root

### CommitteeCache

#### Methods

- **`new() -> Self`**
  - Creates cache with default capacity (256 epochs)

- **`with_capacity(max_epochs: usize) -> Self`**
  - Creates cache with specified capacity

- **`store_stable(epoch: u64, root: Hash256)`**
  - Stores stable committee root for epoch

- **`store_ambiguous(epoch: u64, old_root: Hash256, new_root: Hash256, reorg_end_slot: u64)`**
  - Stores ambiguous entry during reorg

- **`get_committee_view(epoch: u64, current_slot: u64) -> Option<CommitteeView>`**
  - Retrieves committee view for verification

- **`finalize_reorg(epoch: u64, current_slot: u64)`**
  - Converts ambiguous entry to stable

- **`len() -> usize`** / **`is_empty() -> bool`**
  - Query cache size

## Testing Your Integration

### Run All Committee Tests

```bash
# Run integration tests
cargo test --test committee_reorg_test

# Run unit tests
cargo test --lib committee

# Run all tests
cargo test
```

### Expected Output

```
running 11 tests
test test_stable_committee_verification ... ok
test test_mid_epoch_exit_creates_ambiguous_view ... ok
test test_cross_boundary_attestation_verification ... ok
test test_late_inclusion_activation ... ok
test test_committee_cache_reorg_handling ... ok
test test_attestation_verification_fails_with_wrong_root ... ok
test test_multiple_reorgs_in_epoch ... ok
test test_reorg_window_boundaries ... ok
test test_validator_set_integration ... ok
test test_epoch_boundary_reorg ... ok
test test_attestation_partial_committee ... ok

test result: ok. 11 passed; 0 failed; 0 ignored
```

## Common Scenarios

### Scenario 1: Irregular Validator Exit

```rust
// Validator exits mid-epoch without proper warning
let exit_slot = calculate_exit_slot();
committee.trigger_reorg(exit_slot);
committee.update_validator_set(remaining_validators);

// Attestations work during transition
// Finalize after 4 slots
committee.finalize_reorg(exit_slot + 4);
```

### Scenario 2: Late Validator Activation

```rust
// New validator activates later than expected
let activation_slot = validator.actual_activation_slot();
committee.trigger_reorg(activation_slot);
committee.update_validator_set(updated_validator_list);

// Handle transition period
// Finalize when stable
committee.finalize_reorg(activation_slot + 4);
```

### Scenario 3: Multiple Validators Change

```rust
// Several validators exit/join simultaneously
let change_slot = epoch_start + offset;
committee.trigger_reorg(change_slot);

// Update with all changes at once
let new_validators = calculate_new_committee();
committee.update_validator_set(new_validators);

// Single 4-slot window for all changes
committee.finalize_reorg(change_slot + 4);
```

## Performance Tips

1. **Cache Management**: Use appropriate cache capacity for your network size
   ```rust
   // For small networks
   let cache = CommitteeCache::with_capacity(100);
   
   // For large networks
   let cache = CommitteeCache::with_capacity(500);
   ```

2. **Batch Finalization**: Finalize reorgs in batches when possible
   ```rust
   for epoch in epochs_to_finalize {
       cache.finalize_reorg(epoch, current_slot);
   }
   ```

3. **Precompute Roots**: Cache committee roots to avoid recomputation
   ```rust
   let root = assignment.get_committee_view(slot);
   // Store root for reuse
   ```

## Troubleshooting

### Issue: Attestations Still Failing

**Check:**
1. Is reorg triggered before validator set change?
2. Is finalization called too early (before window closes)?
3. Is committee root correctly computed?

```rust
// Debug logging
println!("View: {:?}", committee.get_committee_view(slot));
println!("Reorg state: {:?}", committee.pending_reorg());
```

### Issue: High Memory Usage

**Solution:** Reduce cache capacity

```rust
// Reduce from 256 to 128 epochs
let cache = CommitteeCache::with_capacity(128);
```

### Issue: Slow Verification

**Solution:** Precompute and cache committee roots

```rust
// Compute once, use many times
let view = cache.get_committee_view(epoch, slot)
    .expect("Committee view not found");
```

## Security Considerations

1. **Always use ambiguous views during reorg windows**
   - Never force single-root verification during transitions
   
2. **Finalize reorgs promptly**
   - Don't leave committee in ambiguous state longer than necessary
   
3. **Validate committee roots**
   - Ensure roots are properly computed from sorted validator lists
   
4. **Monitor reorg frequency**
   - Excessive reorgs may indicate network issues

## Additional Resources

- **Full Implementation Report**: See `COMMITTEE_REORG_FIX_REPORT.md`
- **Implementation Summary**: See `IMPLEMENTATION_SUMMARY.md`
- **Test Examples**: See `tests/committee_reorg_test.rs`
- **Source Code**: See `src/validator/committee_assignment.rs`

## Support

For issues or questions:
1. Check test cases in `tests/committee_reorg_test.rs` for examples
2. Review implementation in `src/validator/committee_assignment.rs`
3. Consult documentation in `COMMITTEE_REORG_FIX_REPORT.md`

---
**Last Updated**: June 25, 2026  
**Version**: 1.0.0  
**Status**: Production Ready ✅
