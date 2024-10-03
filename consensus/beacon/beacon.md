
# BeconConsensusEngine 구조체는 아래와 같다 

```rust
pub struct BeaconConsensusEngine<N, BT, Client>
where
    N: EngineNodeTypes,
    Client: BlockClient,
    BT: BlockchainTreeEngine
        + BlockReader
        + BlockIdReader
        + CanonChainTracker
        + StageCheckpointReader,
{
    /// Controls syncing triggered by engine updates.
    sync: EngineSyncController<N, Client>,
    /// The type we can use to query both the database and the blockchain tree.
    blockchain: BT,
    /// Used for emitting updates about whether the engine is syncing or not.
    sync_state_updater: Box<dyn NetworkSyncUpdater>,
    /// The Engine API message receiver.
    engine_message_stream: BoxStream<'static, BeaconEngineMessage<N::Engine>>,
    /// A clone of the handle
    handle: BeaconConsensusEngineHandle<N::Engine>,
    /// Tracks the received forkchoice state updates received by the CL.
    forkchoice_state_tracker: ForkchoiceStateTracker,
    /// The payload store.
    payload_builder: PayloadBuilderHandle<N::Engine>,
    /// Validator for execution payloads
    payload_validator: ExecutionPayloadValidator<N::ChainSpec>,
    /// Current blockchain tree action.
    blockchain_tree_action: Option<BlockchainTreeAction<N::Engine>>,
    /// Pending forkchoice update.
    /// It is recorded if we cannot process the forkchoice update because
    /// a hook with database read-write access is active.
    /// This is a temporary solution to always process missed FCUs.
    pending_forkchoice_update:
        Option<PendingForkchoiceUpdate<<N::Engine as PayloadTypes>::PayloadAttributes>>,
    /// Tracks the header of invalid payloads that were rejected by the engine because they're
    /// invalid.
    invalid_headers: InvalidHeaderCache,
    /// After downloading a block corresponding to a recent forkchoice update, the engine will
    /// check whether or not we can connect the block to the current canonical chain. If we can't,
    /// we need to download and execute the missing parents of that block.
    ///
    /// When the block can't be connected, its block number will be compared to the canonical head,
    /// resulting in a heuristic for the number of missing blocks, or the size of the gap between
    /// the new block and the canonical head.
    ///
    /// If the gap is larger than this threshold, the engine will download and execute the missing
    /// blocks using the pipeline. Otherwise, the engine, sync controller, and blockchain tree will
    /// be used to download and execute the missing blocks.
    pipeline_run_threshold: u64,
    hooks: EngineHooksController,
    /// Sender for engine events.
    event_sender: EventSender<BeaconConsensusEngineEvent>,
    /// Consensus engine metrics.
    metrics: EngineMetrics,
}

```


# 1. sync 
블록체인간 sync 작업을 위한 컨트롤러가 존재하고, 그 안의 메서드들은 다음과 같다. 

EngineSyncController 
1. download full block 
2. update block download metrics 
3. set pipeline 


# 2. blockchain: BT 
BlockchainTreeEngine 블록체인 트리에 접근, 블록관련 작업을 한다. 

1. insert_block_without_senders
2. insert_block_without_senders
3. insert_block
4. finalize block 


# 3. handle: BeaconConsensusEngineHandle

BeaconConsensusEngine에 대한 핸들 



# 4. payload store + validator 


# 5. engine_message_stream: The Engine API message receiver.