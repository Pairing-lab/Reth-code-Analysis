
``` rust 

pub struct Evm<'a, EvmWiringT: EvmWiring> {

/// Context of execution, containing both EVM and external context.

pub context: Context<EvmWiringT>,

/// Handler is a component of the of EVM that contains all the logic. Handler contains specification id

/// and it different depending on the specified fork.

pub handler: Handler<'a, EvmWiringT, Context<EvmWiringT>>,

}

```
EVM 구조체는 다음과 같은 field를 가진다 

[[Context]]
-> Internal 및 external context를 포함한다 

[[Handler]]
-> evm의 핵심 로직?을 포함한다 
***그말인 즉슨 Handler를 구현하는 impl을 어떻게 구현하냐에 따라서 다른 식으로 행동할 수 있을 것이다***

[[EvmBuilder]]
-> Evm을 빌드 해주거나 혹은 Evm을 수정해준다 

[[State]]



``` rust 
//! Mainnet related handlers.
mod execution;

mod post_execution;

mod pre_execution;

mod validation;

// Public exports  

pub use execution::{

call, call_return, create, create_return, eofcreate, eofcreate_return, execute_frame,

insert_call_outcome, insert_create_outcome, insert_eofcreate_outcome, last_frame_return,

};

pub use post_execution::{clear, end, output, refund, reimburse_caller, reward_beneficiary};

pub use pre_execution::{

apply_eip7702_auth_list, deduct_caller, deduct_caller_inner, load_accounts, load_precompiles,

};

pub use validation::{validate_env, validate_initial_tx_gas, validate_tx_against_state};
```

그래서 실제로 보면, `revm`은 상용화를 위해서 mainnet 전용 핸들러를 구현해서 제공한다. 우리의 경우, 이 Handler를 Spec에 따라 고칠 수 있다.
다음은 실제 `mainnet`에 `pre_execution`에 있는 `load accounts` 이다 
``` rust 
/// Main load handle
#[inline]
pub fn load_accounts<EvmWiringT: EvmWiring, SPEC: Spec>(

context: &mut Context<EvmWiringT>,

) -> EVMResultGeneric<(), EvmWiringT> {

// set journaling state flag.

context.evm.journaled_state.set_spec_id(SPEC::SPEC_ID);
// load coinbase
// EIP-3651: Warm COINBASE. Starts the `COINBASE` address warm

if SPEC::enabled(SpecId::SHANGHAI) {
let coinbase = *context.evm.inner.env.block.coinbase();
context
.evm
.journaled_state
.warm_preloaded_addresses
.insert(coinbase);
}

// Load blockhash storage address
// EIP-2935: Serve historical block hashes from state

if SPEC::enabled(SpecId::PRAGUE) {

context
.evm
.journaled_state
.warm_preloaded_addresses
.insert(BLOCKHASH_STORAGE_ADDRESS);

}
// Load access list
context.evm.load_access_list().map_err(EVMError::Database)?;

Ok(())

}
```
이것은 mainnet/pre_execution인데,  `SPEC::SPEC_ID` 는 실제 EVM의 스펙을 지정한다. 

``` rust

pub enum SpecId {

FRONTIER = 0, // Frontier 0

FRONTIER_THAWING = 1, // Frontier Thawing 200000

HOMESTEAD = 2, // Homestead 1150000

DAO_FORK = 3, // DAO Fork 1920000

TANGERINE = 4, // Tangerine Whistle 2463000

SPURIOUS_DRAGON = 5, // Spurious Dragon 2675000

BYZANTIUM = 6, // Byzantium 4370000

CONSTANTINOPLE = 7, // Constantinople 7280000 is overwritten with PETERSBURG

PETERSBURG = 8, // Petersburg 7280000

ISTANBUL = 9, // Istanbul 9069000

MUIR_GLACIER = 10, // Muir Glacier 9200000

BERLIN = 11, // Berlin 12244000

LONDON = 12, // London 12965000

ARROW_GLACIER = 13, // Arrow Glacier 13773000

GRAY_GLACIER = 14, // Gray Glacier 15050000

MERGE = 15, // Paris/Merge 15537394 (TTD: 58750000000000000000000)

SHANGHAI = 16, // Shanghai 17034870 (Timestamp: 1681338455)

CANCUN = 17, // Cancun 19426587 (Timestamp: 1710338135)

PRAGUE = 18, // Prague TBD

PRAGUE_EOF = 19, // Prague+EOF TBD

#[default]

LATEST = u8::MAX,

}
```

이것은 현재 이더리움 `mainnet`의 spec이다. 
-> ***커스터마이징 할 경우, 이 스펙을 고치거나, 따로 구현할 필요가 있다.***




