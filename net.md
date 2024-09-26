# Network

Analysis of the networking components involved in Ethereum's peer-to-peer (P2P) communication.

## Contents

### 1. [Overview](#overview)
### 2. [Network Setting](#network-setting)
### 3. [Tasks](#tasks)
   - [Network Handle and Network Inner](#network-handle-and-network-inner) 
   - [Transactions](#transactions)
   - [ETH Request](#ETH-Request)
   - [Discovery](#discovery)
        - [Discv4](#discv4)
        - [Discv5](#discv5)
        - [DNS Discovery](#dns-discovery)
### 4. [Components](#key-components)
   - [Fetch Client](#fetch-client)
   - [Swarm](#swarm)
        - [Session Manager](#session-manager)
        - [Connection Listener](#connection-listener)
        - [State Manager](#state-manager)
   - [NetworkManager](#networkmanager)
   - [Peer Management](#peer-management)
        - [Peer](#peer)
        - [P2P Protocol](#p2p-protocol)
   
   - [Eth-Wire](#eth-wire)


   ---
## Overview

![alt text](<스크린샷 2024-09-26 오전 10.31.18.png>)

Reth's P2P networking consists primarily of 4 ongoing tasks
:  `Discovery` , `Transactions` , `ETH Requests`, `Network Handle` .

The `Network Handle` manages the network state and interacts with the `Fetch Client` to send `ETH requests`, retrieve `transactions`, and manage `discovery` of peers. The `Swarm` handles **sessions**, **connections**, and **state** management, while the `NetworkManager` coordinates peer connections and P2P protocol operations. `Eth-Wire` facilitates communication between peers by encoding and decoding protocol messages.

## Network Setting

[File : docs/crates/network.md](https://github.com/paradigmxyz/reth/blob/main/docs/crates/network.md?plain=1)

[File: bin/reth/src/node/mod.rs](https://github.com/paradigmxyz/reth/blob/1563506aea09049a85e5cc72c2894f3f7a371581/bin/reth/src/node/mod.rs)

```rust
// Start network
let network = start_network(network_config(db.clone(), chain_id, genesis_hash)).await?;

// Fetch the client
let fetch_client = Arc::new(network.fetch_client().await?);

// Create a new pipeline
let mut pipeline = reth_stages::Pipeline::new()

    // Push the HeaderStage into the pipeline
    .push(HeaderStage {
        downloader: 
        headers::reverse_headers::ReverseHeadersDownloaderBuilder::default()
            .batch_size(config.stages.headers.downloader_batch_size)
            .retries(config.stages.headers.downloader_retries)
            .build(consensus.clone(), fetch_client.clone()),
        consensus: consensus.clone(),
        client: fetch_client.clone(),
        network_handle: network.clone(),
        commit_threshold: config.stages.headers.commit_threshold,
        metrics: HeaderMetrics::default(),
    })

    // Push the BodyStage into the pipeline
    .push(BodyStage {
        downloader: Arc::new(
            bodies::bodies::BodiesDownloader::new(
                fetch_client.clone(),
                consensus.clone(),
            )
            .with_batch_size(config.stages.bodies.downloader_batch_size)
            .with_retries(config.stages.bodies.downloader_retries)
            .with_concurrency(config.stages.bodies.downloader_concurrency),
        ),
        consensus: consensus.clone(),
        commit_threshold: config.stages.bodies.commit_threshold,
    })

    // Push the SenderRecoveryStage into the pipeline
    .push(SenderRecoveryStage {
        commit_threshold: config.stages.sender_recovery.commit_threshold,
    })

    // Push the ExecutionStage into the pipeline
    .push(ExecutionStage { config: ExecutorConfig::new_ethereum() });

// Check a tip (latest block)
if let Some(tip) = self.tip {
    debug!("Tip manually set: {}", tip);

     // Notify the consensus mechanism of the fork choice state
    consensus.notify_fork_choice_state(ForkchoiceState {
        head_block_hash: tip,
        safe_block_hash: tip,
        finalized_block_hash: tip,
    })?;
}

// Run pipeline
info!("Starting pipeline");
pipeline.run(db.clone()).await?;
```
Now Let's Start the Network

File: bin/reth/src/node/mod.rs
``` Rust
// Start the Network
async fn start_network<C>(config: NetworkConfig<C>) -> Result<NetworkHandle, NetworkError>
where
    C: BlockReader + HeaderProvider + 'static,
{
    // Clone the network client
    let client = config.client.clone();
    // Set up the network manager and initialize the network components
    let (handle, network, _txpool, eth) =
        NetworkManager::builder(config).await?.request_handler(client).split_with_handle();

    // Network : Background Execution (Asynchronous)
    tokio::task::spawn(network);
    // TODO: tokio::task::spawn(txpool);
    // Ethereum protocol handler : Background Execution (Asynchronous)
    tokio::task::spawn(eth);

    // Return the network handle to control the network
    Ok(handle)
}
```

## Tasks

### Network Handle and Network Inner

```Rust
pub struct NetworkHandle {
    inner: Arc<NetworkInner>,
}
```
```Rust
struct NetworkInner {
    num_active_peers: Arc<AtomicUsize>,
    to_manager_tx: UnboundedSender<NetworkHandleMessage>,
    listener_address: Arc<Mutex<SocketAddr>>,
    local_peer_id: PeerId,
    peers: PeersHandle,
    network_mode: NetworkMode,
}
```
`to_manager_tx` :  which is a handle that can be used to send messages in a channel to an instance of the NetworkManager struct.

---
### Transactions

#### Transaction Handler
① Send Commands : Send commands to the `Transaction Manager` using an `UnboundedSender`.  
② Propagate Transactions : Sending a transaction hash to the manager to propagate the transaction.

#### Transaction Manager
① Transaction Pool  
        - Manages the set of pending transactions  
        - Handles their validation and import    
② Network Handle  
        - Interacts with the network to send & receive transactions.  
③ Peer Management  
④ Command Handling


[File: crates/net/network/src/transactions.rs](https://github.com/paradigmxyz/reth/blob/1563506aea09049a85e5cc72c2894f3f7a371581/crates/net/network/src/transactions.rs)


```Rust
pub struct TransactionsHandle {
    manager_tx: mpsc::UnboundedSender<TransactionsCommand>,
}
```
```Rust
impl TransactionsHandle {
    fn send(&self, cmd: TransactionsCommand) {
        let _ = self.manager_tx.send(cmd);
    }

    pub fn propagate(&self, hash: TxHash) {
        self.send(TransactionsCommand::PropagateHash(hash))
    }
}
```

```Rust
pub struct TransactionsManager<Pool> {
    pool: Pool,
    network: NetworkHandle,
    network_events: UnboundedReceiverStream<NetworkEvent>,
    inflight_requests: FuturesUnordered<GetPooledTxRequestFut>,
    transactions_by_peers: HashMap<TxHash, Vec<PeerId>>,
    pool_imports: FuturesUnordered<PoolImportFuture>,
    peers: HashMap<PeerId, Peer>,
    command_tx: mpsc::UnboundedSender<TransactionsCommand>,
    command_rx: UnboundedReceiverStream<TransactionsCommand>,
    pending_transactions: ReceiverStream<TxHash>,
    transaction_events: UnboundedMeteredReceiver<NetworkTransactionEvent>,
    metrics: TransactionsManagerMetrics,
}
```
```Rust
impl<Pool: TransactionPool> TransactionsManager<Pool> {
    pub fn new(
        network: NetworkHandle,
        pool: Pool,
        from_network: mpsc::UnboundedReceiver<NetworkTransactionEvent>,
    ) -> Self {
        let network_events = network.event_listener();
        let (command_tx, command_rx) = mpsc::unbounded_channel();

        let pending = pool.pending_transactions_listener();

        Self {
            pool,
            network,
            network_events,
            inflight_requests: Default::default(),
            transactions_by_peers: Default::default(),
            pool_imports: Default::default(),
            peers: Default::default(),
            command_tx,
            command_rx: UnboundedReceiverStream::new(command_rx),
            pending_transactions: ReceiverStream::new(pending),
            transaction_events: UnboundedMeteredReceiver::new(
                from_network,
                NETWORK_POOL_TRANSACTIONS_SCOPE,
            ),
            metrics: Default::default(),
        }
    }
}
```
