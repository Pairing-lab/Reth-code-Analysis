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

<img width="739" alt="overview" src="https://github.com/user-attachments/assets/c9be73b4-553f-49dc-b9b9-e4b466ed2ede">

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

---
### ETH Requests

` // To Do : poll, get_headers_response, get_bodies_response (+ eth_wire) `


[File: crates/net/network/src/eth_requests.rs](https://github.com/paradigmxyz/reth/blob/1563506aea09049a85e5cc72c2894f3f7a371581/crates/net/network/src/eth_requests.rs)


---
### Discovery
` // To do : discovery `

---

## Components
---

### Fetch Client

` // To Do : StateFetcher, Client(HeadersClient, BodiesClient), Downloader (Headers, Bodies), poll Action `

_Using FetchClient to Get Data in the Pipeline Stages_

<img width="608" alt="fetch client" src="https://github.com/user-attachments/assets/570099cd-7f6a-4d23-864d-d3d19d15653c">


[File: crates/net/network/src/fetch/client.rs](https://github.com/paradigmxyz/reth/blob/main/docs/crates/network.md#using-fetchclient-to-get-data-in-the-pipeline-stages)

```Rust
pub struct FetchClient {
    pub(crate) request_tx: UnboundedSender<DownloadRequest>,
    pub(crate) peers_handle: PeersHandle,
}
```
---
### Swarm
> 네트워크 연결, 세션, 상태를 관리하는 작업
![alt text](<스크린샷 2024-09-26 오후 2.29.49.png>)

#### ① [Connection Listener](#connection-listener)
Handles incoming TCP connections

#### ② [Session Manager](#session-manager)
Manages active and pending sessions

#### ③ [Network State](#network-state)
- Tracks the network state , processing session events and network commands
- Ensure efficient communication** and maintain network stability


[File: crates/net/network/src/swarm.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/swarm.rs)

### Swarm 구조체 
```Rust
pub(crate) struct Swarm<C> {
    incoming: ConnectionListener,
    sessions: SessionManager,
    state: NetworkState<C>,
}
```
- `incoming` : 들어오는 연결 수신하는 `ConnectionListener`
- `sessions` : 모든 세션 관리하는 `SessionManager`
- `state` : 네트워크 전체 상태 관리하는 `NetworkState`

### Swarm 구현
---
### 1. new
```Rust
impl Swarm {
    pub(crate) const fn new(
        incoming: ConnectionListener,
        sessions: SessionManager,
        state: NetworkState,
    ) -> Self {
        Self { incoming, sessions, state }
    }
}
```
- 새로운 `swarm` 인스턴스를 생성
- `ConnectionListener`, `SessionManager`, `NetworkState` 초기화

### 2. 네트워크 연결 관리 Method
```Rust
impl Swarm {
    fn on_connection(&mut self, event: ListenerEvent) -> Option<SwarmEvent> {
        match event {
            ListenerEvent::Error(err) => return Some(SwarmEvent::TcpListenerError(err)),
            ListenerEvent::ListenerClosed { local_address: address } => {
                return Some(SwarmEvent::TcpListenerClosed { remote_addr: address })
            }
            ListenerEvent::Incoming { stream, remote_addr } => {
                // 노드가 종료 중이면 들어오는 연결 거부 
                if self.is_shutting_down() {
                    return None;
                }
                // 들어오는 연결 가능 확인
                if let Err(err) = self.state_mut().peers_mut().on_incoming_pending_session(remote_addr.ip()) {
                    match err {
                        InboundConnectionError::IpBanned => {
                            trace!(target: "net", ?remote_addr, "The incoming ip address is in the ban list");
                        }
                        InboundConnectionError::ExceedsCapacity => {
                            trace!(target: "net", ?remote_addr, "No capacity for incoming connection");
                        }
                    }
                    return None;
                }

                // 들어오는 연결 처리 후 세션 생성
                match self.sessions.on_incoming(stream, remote_addr) {
                    Ok(session_id) => {
                        trace!(target: "net", ?remote_addr, "Incoming connection");
                        return Some(SwarmEvent::IncomingTcpConnection { session_id, remote_addr });
                    }
                    Err(err) => {
                        trace!(target: "net", %err, "Incoming connection rejected, capacity already reached.");
                        self.state_mut().peers_mut().on_incoming_pending_session_rejected_internally();
                    }
                }
            }
        }
        None
    }
}
```
- `on_connection`
    - 새로 들어오는 네트워크 연결 처리
    - 네트워크 상태 확인, 새로운 세션 생성, 오류 기록, 
```Rust
impl Swarm {
    pub(crate) fn dial_outbound(&mut self, remote_addr: SocketAddr, remote_id: PeerId) {
        self.sessions.dial_outbound(remote_addr, remote_id)
    }
}
```
- `dial_outbound` : outbound 연결을 시도하는 method로, `SessionManager`에게 연결 요청 위임 

### 3. 세션 관리 Method
```Rust
impl Swarm {
    /// 세션 이벤트를 처리하는 메서드
    fn on_session_event(&mut self, event: SessionEvent) -> Option<SwarmEvent> {
        match event {
            SessionEvent::SessionEstablished {
                peer_id,
                remote_addr,
                client_version,
                capabilities,
                version,
                status,
                messages,
                direction,
                timeout,
            } => {
                self.state.on_session_activated(
                    peer_id,
                    capabilities.clone(),
                    status.clone(),
                    messages.clone(),
                    timeout,
                );
                Some(SwarmEvent::SessionEstablished {
                    peer_id,
                    remote_addr,
                    client_version,
                    capabilities,
                    version,
                    messages,
                    status,
                    direction,
                })
            }
            SessionEvent::AlreadyConnected { peer_id, remote_addr, direction } => {
                trace!(target: "net", ?peer_id, ?remote_addr, ?direction, "already connected");
                self.state.peers_mut().on_already_connected(direction);
                None
            }
            SessionEvent::ValidMessage { peer_id, message } => {
                Some(SwarmEvent::ValidMessage { peer_id, message })
            }
            SessionEvent::InvalidMessage { peer_id, capabilities, message } => {
                Some(SwarmEvent::InvalidCapabilityMessage { peer_id, capabilities, message })
            }
            SessionEvent::IncomingPendingSessionClosed { remote_addr, error } => {
                Some(SwarmEvent::IncomingPendingSessionClosed { remote_addr, error })
            }
            SessionEvent::OutgoingPendingSessionClosed { remote_addr, peer_id, error } => {
                Some(SwarmEvent::OutgoingPendingSessionClosed { remote_addr, peer_id, error })
            }
            SessionEvent::Disconnected { peer_id, remote_addr } => {
                self.state.on_session_closed(peer_id);
                Some(SwarmEvent::SessionClosed { peer_id, remote_addr, error: None })
            }
            SessionEvent::SessionClosedOnConnectionError { peer_id, remote_addr, error } => {
                self.state.on_session_closed(peer_id);
                Some(SwarmEvent::SessionClosed { peer_id, remote_addr, error: Some(error) })
            }
            SessionEvent::OutgoingConnectionError { remote_addr, peer_id, error } => {
                Some(SwarmEvent::OutgoingConnectionError { peer_id, remote_addr, error })
            }
            SessionEvent::BadMessage { peer_id } => Some(SwarmEvent::BadMessage { peer_id }),
            SessionEvent::ProtocolBreach { peer_id } => {
                Some(SwarmEvent::ProtocolBreach { peer_id })
            }
        }
    }
}
```
- `on_session_event` : 세션과 관련된 이벤트 처리 

### 3. 네트워크 상태 관리 Method
```Rust
impl Swarm {
    /// 네트워크 상태에 따라 액션을 처리하는 메서드
    fn on_state_action(&mut self, event: StateAction) -> Option<SwarmEvent> {
        match event {
            StateAction::Connect { remote_addr, peer_id } => {
                self.dial_outbound(remote_addr, peer_id);
                Some(SwarmEvent::OutgoingTcpConnection { remote_addr, peer_id })
            }
            StateAction::Disconnect { peer_id, reason } => {
                self.sessions.disconnect(peer_id, reason);
            }
            StateAction::NewBlock { peer_id, block: msg } => {
                let msg = PeerMessage::NewBlock(msg);
                self.sessions.send_message(&peer_id, msg);
            }
            StateAction::NewBlockHashes { peer_id, hashes } => {
                let msg = PeerMessage::NewBlockHashes(hashes);
                self.sessions.send_message(&peer_id, msg);
            }
            StateAction::PeerAdded(peer_id) => Some(SwarmEvent::PeerAdded(peer_id)),
            StateAction::PeerRemoved(peer_id) => Some(SwarmEvent::PeerRemoved(peer_id)),
            StateAction::DiscoveredNode { peer_id, addr, fork_id } => {
                if self.is_shutting_down() {
                    return None;
                }
                if fork_id.map_or_else(|| true, |f| self.sessions.is_valid_fork_id(f)) {
                    self.state_mut().peers_mut().add_peer(peer_id, addr, fork_id);
                }
            }
            StateAction::DiscoveredEnrForkId { peer_id, fork_id } => {
                if self.sessions.is_valid_fork_id(fork_id) {
                    self.state_mut().peers_mut().set_discovered_fork_id(peer_id, fork_id);
                } else {
                    self.state_mut().peers_mut().remove_peer(peer_id);
                }
            }
        }
        None
    }

    /// 네트워크 종료 요청 처리
    pub(crate) fn on_shutdown_requested(&mut self) {
        self.state_mut().peers_mut().on_shutdown();
    }

    /// 네트워크가 종료 중인지 확인
    #[inline]
    pub(crate) const fn is_shutting_down(&self) -> bool {
        self.state().peers().connection_state().is_shutting_down()
    }

    /// 네트워크 상태 변경
    pub(crate) fn on_network_state_change(&mut self, network_state: NetworkConnectionState) {
        self.state_mut().peers_mut().on_network_state_change(network_state);
    }
}
```
- `on_state_action` :  네트워크 상태에 따라 새로운 연결 시도, peer 추가 및 삭제를 처리
- `on_shutdown_requested` : 네트워크가 종료될 때 호출되는 method로 새로운 연결을 받지 않도록 상태 변경
- `is_shutting_down` : 노드가 종료 중인지를 확인
- `on_network_state_change` : 네트워크 상태가 변경되었을 때 호출되는 method로 노드의 활동 상태 변경 가능 

### 4. Stream 구현
```Rust
impl Stream for Swarm {
    type Item = SwarmEvent;

    /// 네트워크 이벤트를 폴링하여 처리하는 메서드
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        let this = self.get_mut();

        // 네트워크 상태를 먼저 처리
        while let Poll::Ready(action) = this.state.poll(cx) {
            if let Some(event) = this.on_state_action(action) {
                return Poll::Ready(Some(event));
            }
        }

        // 세션 이벤트를 처리
        match this.sessions.poll(cx) {
            Poll::Pending => {}
            Poll::Ready(event) => {
                if let Some(event) = this.on_session_event(event) {
                    return Poll::Ready(Some(event));
                }
                continue;
            }
        }

        // 들어오는 연결을 처리
        match Pin::new(&mut this.incoming).poll(cx) {
            Poll::Pending => {}
            Poll::Ready(event) => {
                if let Some(event) = this.on_connection(event) {
                    return Poll::Ready(Some(event));
                }
                continue;
            }
        }

        Poll::Pending
    }
}
```
- `poll_next` : `Stream`을 구현하여 `polling`을 통해 네트워크와 세션 이벤트를 **비동기적**으로 처리


#### **[Session Manager]**
> responsible for managing peer sessions in a blockchain network

#### ① Session Creation
- When a new session is initiated, it enters the **pending** state.
- **Handshaking** : The session exchanging `Hello` messages and validating the peer's status.
- Once authenticated ...  `pending_sessions` → `active_sessions`  
  Then, it can **exchange messages.**
#### ② Sesstion Management
- The session manager tracks **pending** and **active sessions**.
- Ensures the session limit is not exceeded.
- `send_message()` : **Messages are Exchanged** using the channels set up for each session.
#### ③ Timeout and Error Handling:
- Session Failure : If the handshake is not completed within the `pending_session_timeout`
- TimeoutError : If a session fails to respond to requests within `initial_internal_request_timeout`.
- Terminate : sessions that violate protocol.
#### ④ Metrics and Monitoring
- `SessionManagerMetrics` : Tracks various events (e.g. message drops, successful connections, and timeouts.)
- Help monitor and optimize the performance of session management.

[File : crates/net/network/src/manager.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/manager.rs)

` // To Do : 기존 주석을 그대로 사용할 것인지 새로 작성할 것인지 `
```Rust
pub struct SessionManager {
}
```
```Rust
impl SessionManager {
}
```



#### **[Connection Listener]**
[File : crates/net/network/src/listener.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/listener.rs)
#### **[Network State]**
[File : crates/net/network/src/state.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/state.rs#L72)

> // To Do : Session Manager, Connection Listener, State Manager
---

### Network Manager
---

### Peer Management
---

#### **Peer
#### **P2P Protocol
---

### Eth-Wire
