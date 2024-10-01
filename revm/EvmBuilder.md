`EvmBuilder`는 evm을 만들거나 수정할 수 있도록 해준다.
-> External Context 혹은 Custom Handler를 설정할 수 있도록 해준다. 

[[Context]]

[[Handler]]

`Builder` 는 generic한 `Database`, `External Context`  와 `Spec` 의 의존성을 묶어준다 

Evm Builder의 구조이다 
```rust 
pub struct EvmBuilder<'a, BuilderStage, EvmWiringT: EvmWiring> {
database: Option<EvmWiringT::Database>,
external_context: Option<EvmWiringT::ExternalContext>,
env: Option<Box<EnvWiring<EvmWiringT>>>,
/// Handler that will be used by EVM. It contains handle registers
handler: Handler<'a, EvmWiringT, Context<EvmWiringT>>,
/// Phantom data to mark the stage of the builder.
phantom: PhantomData<BuilderStage>,
}
```
-> `database` ,  `env` , `handler` , `phantom` 의 의존성을 묶기 위해서 field로 둔다. 
 
``` rust

use crate::evm::Evm; // build Evm with default values. 
let mut evm = Evm::builder().build(); let output = evm.transact();

```

-> Evm 빌딩 예시 

