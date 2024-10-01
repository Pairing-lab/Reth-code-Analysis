
Host는 간단히 말해서 EVM Context와 Interpreter 간 통신하는 인터페이스라고 할 수 있다.

``` rust
pub trait Host {
    /// Chain specification.
    type EvmWiringT: EvmWiring;

    /// Returns a reference to the environment.
    fn env(&self) -> &EnvWiring<Self::EvmWiringT>;

    /// Returns a mutable reference to the environment.
    fn env_mut(&mut self) -> &mut EnvWiring<Self::EvmWiringT>;

    /// Load an account code.
    fn load_account_delegated(&mut self, address: Address) -> Option<AccountLoad>;

    /// Get the block hash of the given block `number`.
    fn block_hash(&mut self, number: u64) -> Option<B256>;

    /// Get balance of `address` and if the account is cold.
    fn balance(&mut self, address: Address) -> Option<StateLoad<U256>>;

    /// Get code of `address` and if the account is cold.
    fn code(&mut self, address: Address) -> Option<Eip7702CodeLoad<Bytes>>;

    /// Get code hash of `address` and if the account is cold.
    fn code_hash(&mut self, address: Address) -> Option<Eip7702CodeLoad<B256>>;

    /// Get storage value of `address` at `index` and if the account is cold.
    fn sload(&mut self, address: Address, index: U256) -> Option<StateLoad<U256>>;

    /// Set storage value of account address at index.
    ///
    /// Returns [`StateLoad`] with [`SStoreResult`] that contains original/new/old storage value.
    fn sstore(
        &mut self,
        address: Address,
        index: U256,
        value: U256,
    ) -> Option<StateLoad<SStoreResult>>;

    /// Get the transient storage value of `address` at `index`.
    fn tload(&mut self, address: Address, index: U256) -> U256;

    /// Set the transient storage value of `address` at `index`.
    fn tstore(&mut self, address: Address, index: U256, value: U256);

    /// Emit a log owned by `address` with given `LogData`.
    fn log(&mut self, log: Log);

    /// Mark `address` to be deleted, with funds transferred to `target`.
    fn selfdestruct(
        &mut self,
        address: Address,
        target: Address,
    ) -> Option<StateLoad<SelfDestructResult>>;
}

```