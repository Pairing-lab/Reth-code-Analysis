Evm의 상태를 저장한다 
``` rust

pub struct Context<EvmWiringT: EvmWiring> {

/// Evm Context (internal context).

pub evm: EvmContext<EvmWiringT>,

/// External contexts.

pub external: EvmWiringT::ExternalContext,

}

```

Internal context를 기록한다.
`EvmContext` 는 다음과 같은 상태들을 기록한다  `Database`, `Environment`, `JournaledState` , `Precompiles`.
``` rust 
pub struct EvmContext<EvmWiringT: EvmWiring> {

/// Inner EVM context.

pub inner: InnerEvmContext<EvmWiringT>,

/// Precompiles that are available for evm.

pub precompiles: ContextPrecompiles<EvmWiringT>,
}
```

---> 
``` rust 

pub struct InnerEvmContext<EvmWiringT: EvmWiring> {

/// EVM Environment contains all the information about config, block and transaction that

/// evm needs.
pub env: Box<EnvWiring<EvmWiringT>>,

/// EVM State with journaling support.

pub journaled_state: JournaledState,

/// Database to load data from.

pub db: EvmWiringT::Database,

/// Inner context.

pub chain: EvmWiringT::ChainContext,

/// Error that happened during execution.

pub error: Result<(), <EvmWiringT::Database as Database>::Error>,

}

```

상단에 있는 `env` 파일에 Evm context(`block` 및 `tx` 자료구조) 가 존재한다 

``` rust

pub type EnvWiring<EvmWiringT> =

Env<<EvmWiringT as EvmWiring>::Block, <EvmWiringT as EvmWiring>::Transaction>;

```

다음과 같은 형태로 존재한다, 코드를 통해서 Env 가 `Block`과 `Transaction`을 구현한 `trait` 을 `Generic` 으로 가진다는 것을 알 수 있다.




