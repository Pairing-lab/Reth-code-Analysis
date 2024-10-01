1. CallInputs
   - EVM 호출에 필요한 입력 데이터를 나타낸다.
   - 주요 필드들: input (호출 데이터), gas_limit (가스 제한), bytecode_address (실행될 바이트코드의 주소), target_address (대상 주소), caller (호출자), value (호출 값), scheme (호출 방식) 등이 있다
   - `new` 메서드: 트랜잭션 환경과 가스 제한을 받아 CallInputs를 생성한다
   - `transfers_value`, `transfer_value`, `apparent_value` 등의 메서드: 값 전송과 관련된 정보 제공

``` rust
pub struct CallInputs {
    /// The call data of the call.
    pub input: Bytes,
    /// The return memory offset where the output of the call is written.
    ///
    /// In EOF, this range is invalid as EOF calls do not write output to memory.
    pub return_memory_offset: Range<usize>,
    /// The gas limit of the call.
    pub gas_limit: u64,
    /// The account address of bytecode that is going to be executed.
    ///
    /// Previously `context.code_address`.
    pub bytecode_address: Address,
    /// Target address, this account storage is going to be modified.
    ///
    /// Previously `context.address`.
    pub target_address: Address,
    /// This caller is invoking the call.
    ///
    /// Previously `context.caller`.
    pub caller: Address,
    /// Call value.
    ///
    /// NOTE: This value may not necessarily be transferred from caller to callee, see [`CallValue`].
    ///
    /// Previously `transfer.value` or `context.apparent_value`.
    pub value: CallValue,
    /// The call scheme.
    ///
    /// Previously `context.scheme`.
    pub scheme: CallScheme,
    /// Whether the call is a static call, or is initiated inside a static call.
    pub is_static: bool,
    /// Whether the call is initiated from EOF bytecode.
    pub is_eof: bool,
}
```

2. CallScheme
   - 다양한 호출 방식을 나타낸다 (CALL, CALLCODE, DELEGATECALL ... )
   - `is_ext`와 `is_ext_delegate_call` 메서드: 특정 호출 방식인지 확인한다. -> method로 구현되어 있음 
``` rust
pub enum CallScheme {
    /// `CALL`.
    Call,
    /// `CALLCODE`
    CallCode,
    /// `DELEGATECALL`
    DelegateCall,
    /// `STATICCALL`
    StaticCall,
    /// `EXTCALL`
    ExtCall,
    /// `EXTSTATICCALL`
    ExtStaticCall,
    /// `EXTDELEGATECALL`
    ExtDelegateCall,
}
```

3. CallOutcome
   - 호출 작업의 결과를 나타냄
   - `result` (InterpreterResult)와 `memory_offset` (출력 데이터가 저장된 메모리 범위)를 포함
```rust
   pub struct CallOutcome {
    pub result: InterpreterResult,
    pub memory_offset: Range<usize>,
    }


```
4. CreateInputs
   - 컨트랙트 계정 생성 호출에 필요한 입력 데이터를 나타낸다
   - `new` 메서드: 트랜잭션 환경과 가스 제한을 받아 CreateInputs를 생성함
   - `created_address` 메서드: 생성될 컨트랙트의 주소를 계산함
```rust
impl CreateInputs {
    /// Creates new create inputs.
    pub fn new(tx_env: &impl Transaction, gas_limit: u64) -> Option<Self> {
        let TxKind::Create = tx_env.kind() else {
            return None;
        };

        Some(CreateInputs {
            caller: *tx_env.caller(),
            scheme: CreateScheme::Create,
            value: *tx_env.value(),
            init_code: tx_env.data().clone(),
            gas_limit,
        })
    }

    /// Returns boxed create inputs.
    pub fn new_boxed(tx_env: &impl Transaction, gas_limit: u64) -> Option<Box<Self>> {
        Self::new(tx_env, gas_limit).map(Box::new)
    }

    /// Returns the address that this create call will create.
    pub fn created_address(&self, nonce: u64) -> Address {
        match self.scheme {
            CreateScheme::Create => self.caller.create(nonce),
            CreateScheme::Create2 { salt } => self
                .caller
                .create2_from_code(salt.to_be_bytes(), &self.init_code),
        }
    }
}
```

6. CreateOutcome
   - 컨트랙트 생성 작업의 결과를 나타낸다.
   - `result` (InterpreterResult)와 `address` (생성된 컨트랙트의 주소)를 포함한다
```rust
pub struct CreateOutcome {
    // The result of the interpreter operation.
    pub result: InterpreterResult,
    // An optional address associated with the create operation.
    pub address: Option<Address>,
}
```

***TODO:***
EOF 설명 
/// EOF create can be called from two places:
/// * EOFCREATE opcode
/// * Creation transaction.
///
/// Creation transaction uses initdata and packs EOF and initdata inside it.
/// This eof bytecode needs to be validated.
///
/// Opcode creation uses already validated EOF bytecode, and input from Interpreter memory.
/// Address is already known and is passed as an argument.