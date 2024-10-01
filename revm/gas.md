
가스 가격을 계산한다. 구현체는 다음과 같다. 

***각각의 Opcode마다 다르다!***
```rust 
/// `SSTORE` opcode refund calculation.
#[allow(clippy::collapsible_else_if)]
#[inline]
pub fn sstore_refund(spec_id: SpecId, vals: &SStoreResult) -> i64 {
    if spec_id.is_enabled_in(SpecId::ISTANBUL) {
        // EIP-3529: Reduction in refunds
        let sstore_clears_schedule = if spec_id.is_enabled_in(SpecId::LONDON) {
            (SSTORE_RESET - COLD_SLOAD_COST + ACCESS_LIST_STORAGE_KEY) as i64
        } else {
            REFUND_SSTORE_CLEARS
        };
        if vals.is_new_eq_present() {
            0
        } else {
            if vals.is_original_eq_present() && vals.is_new_zero() {
                sstore_clears_schedule
            } else {
                let mut refund = 0;

                if !vals.is_original_zero() {
                    if vals.is_present_zero() {
                        refund -= sstore_clears_schedule;
                    } else if vals.is_new_zero() {
                        refund += sstore_clears_schedule;
                    }
                }

                if vals.is_original_eq_new() {
                    let (gas_sstore_reset, gas_sload) = if spec_id.is_enabled_in(SpecId::BERLIN) {
                        (SSTORE_RESET - COLD_SLOAD_COST, WARM_STORAGE_READ_COST)
                    } else {
                        (SSTORE_RESET, sload_cost(spec_id, false))
                    };
                    if vals.is_original_zero() {
                        refund += (SSTORE_SET - gas_sload) as i64;
                    } else {
                        refund += (gas_sstore_reset - gas_sload) as i64;
                    }
                }

                refund
            }
        }
    } else {
        if !vals.is_present_zero() && vals.is_new_zero() {
            REFUND_SSTORE_CLEARS
        } else {
            0
        }
    }
}

/// `CREATE2` opcode cost calculation.
#[inline]
pub const fn create2_cost(len: u64) -> Option<u64> {
    CREATE.checked_add(tri!(cost_per_word(len, KECCAK256WORD)))
}

#[inline]
const fn log2floor(value: U256) -> u64 {
    let mut l: u64 = 256;
    let mut i = 3;
    loop {
        if value.as_limbs()[i] == 0u64 {
            l -= 64;
        } else {
            l -= value.as_limbs()[i].leading_zeros() as u64;
            if l == 0 {
                return l;
            } else {
                return l - 1;
            }
        }
        if i == 0 {
            break;
        }
        i -= 1;
    }
    l
}

/// `EXP` opcode cost calculation.
#[inline]
pub fn exp_cost(spec_id: SpecId, power: U256) -> Option<u64> {
    if power.is_zero() {
        Some(EXP)
    } else {
        // EIP-160: EXP cost increase
        let gas_byte = U256::from(if spec_id.is_enabled_in(SpecId::SPURIOUS_DRAGON) {
            50
        } else {
            10
        });
        let gas = U256::from(EXP)
            .checked_add(gas_byte.checked_mul(U256::from(log2floor(power) / 8 + 1))?)?;

        u64::try_from(gas).ok()
    }
}

/// `*COPY` opcodes cost calculation.
#[inline]
pub const fn copy_cost_verylow(len: u64) -> Option<u64> {
    copy_cost(VERYLOW, len)
}

/// `EXTCODECOPY` opcode cost calculation.
#[inline]
pub const fn extcodecopy_cost(spec_id: SpecId, len: u64, load: Eip7702CodeLoad<()>) -> Option<u64> {
    let base_gas = if spec_id.is_enabled_in(SpecId::BERLIN) {
        warm_cold_cost_with_delegation(load)
    } else if spec_id.is_enabled_in(SpecId::TANGERINE) {
        700
    } else {
        20
    };
    copy_cost(base_gas, len)
}

#[inline]
pub const fn copy_cost(base_cost: u64, len: u64) -> Option<u64> {
    base_cost.checked_add(tri!(cost_per_word(len, COPY)))
}

/// `LOG` opcode cost calculation.
#[inline]
pub const fn log_cost(n: u8, len: u64) -> Option<u64> {
    tri!(LOG.checked_add(tri!(LOGDATA.checked_mul(len)))).checked_add(LOGTOPIC * n as u64)
}

/// `KECCAK256` opcode cost calculation.
#[inline]
pub const fn keccak256_cost(len: u64) -> Option<u64> {
    KECCAK256.checked_add(tri!(cost_per_word(len, KECCAK256WORD)))
}

```