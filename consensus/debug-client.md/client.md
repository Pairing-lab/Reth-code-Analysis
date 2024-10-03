
# 1. Block Provider

    Consensus Client에게 블록을 공급하는 기능 
```rust
    pub trait BlockProvider: Send + Sync + 'static {
    /// Runs a block provider to send new blocks to the given sender.
    ///
    /// Note: This is expected to be spawned in a separate task.
    fn subscribe_blocks(&self, tx: mpsc::Sender<Block>) -> impl Future<Output = ()> + Send;

    /// Get a past block by number.
    fn get_block(&self, block_number: u64) -> impl Future<Output = eyre::Result<Block>> + Send;

    /// Get previous block hash using previous block hash buffer. If it isn't available (buffer
    /// started more recently than `offset`), fetch it using `get_block`.
    fn get_or_fetch_previous_block(
        &self,
        previous_block_hashes: &AllocRingBuffer<B256>,
        current_block_number: u64,
        offset: usize,
    ) -> impl Future<Output = eyre::Result<B256>> + Send {
        async move {
            let stored_hash = previous_block_hashes
                .len()
                .checked_sub(offset)
                .and_then(|index| previous_block_hashes.get(index));
            if let Some(hash) = stored_hash {
                return Ok(*hash);
            }

            // Return zero hash if the chain isn't long enough to have the block at the offset.
            let previous_block_number = match current_block_number.checked_sub(offset as u64) {
                Some(number) => number,
                None => return Ok(B256::default()),
            };
            let block = self.get_block(previous_block_number).await?;
            Ok(block.header.hash)
        }
    }


```


# 2. DebugConsensusClient

    ConsensusClient. 
```rust 
    pub struct DebugConsensusClient<P: BlockProvider> {
    /// Handle to execution client.
    auth_server: AuthServerHandle,
    /// Provider to get consensus blocks from.
    block_provider: P,
}
```
-> BlockProvider 사용 
-> fn new ConsensusCLient 생성 
-> fn run ConsensusClient 시작 

# 3. New Payload Event 

```rust

pub struct ExecutionNewPayload {
    pub execution_payload_v3: ExecutionPayloadV3,
    pub versioned_hashes: Vec<B256>,
    pub parent_beacon_block_root: B256,
}
```
