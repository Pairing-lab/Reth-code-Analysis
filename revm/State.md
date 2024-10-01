***State 파일에는 다음과 같은 구조체가 존재한다.*** 
-> `State` inherits the `Database` trait and implements fetching of external state and storage, 


``` rust
pub struct Account {
/// Balance, nonce, and code.
pub info: AccountInfo,
/// Storage cache
pub storage: EvmStorage,
/// Account status flags.
pub status: AccountStatus,
}
```



``` rust
/// EVM State is a mapping from addresses to accounts.
pub type EvmState = HashMap<Address, Account>;
/// Structure used for EIP-1153 transient storage.
pub type TransientStorage = HashMap<(Address, U256), U256>;
/// An account's Storage is a mapping from 256-bit integer keys to [EvmStorageSlot]s.
pub type EvmStorage = HashMap<U256, EvmStorageSlot>;
```

