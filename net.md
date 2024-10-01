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

## Discovery
네트워크에서 노드 발견하고 관리 

## 1. `Discovery` 구조체
```Rust
pub struct Discovery {
    discovered_nodes: LruMap<PeerId, PeerAddr>,            // 발견된 노드의 캐시
    local_enr: NodeRecord,                                 // 로컬 노드의 ENR (Ethereum Node Record)
    discv4: Option<Discv4>,                                // Discovery v4 서비스 핸들러
    discv4_updates: Option<ReceiverStream<DiscoveryUpdate>>, // Discovery v4 업데이트 스트림
    _discv4_service: Option<JoinHandle<()>>,               // 비동기 처리용 v4 서비스 핸들
    discv5: Option<Discv5>,                                // Discovery v5 서비스 핸들러
    discv5_updates: Option<ReceiverStream<discv5::Event>>, // Discovery v5 업데이트 스트림
    _dns_discovery: Option<DnsDiscoveryHandle>,            // DNS 기반 노드 발견 핸들러
    dns_discovery_updates: Option<ReceiverStream<DnsNodeRecordUpdate>>, // DNS 발견 서비스의 업데이트 스트림
    _dns_disc_service: Option<JoinHandle<()>>,             // 비동기 처리용 DNS 서비스 핸들
    queued_events: VecDeque<DiscoveryEvent>,               // 대기 중인 이벤트
    discovery_listeners: Vec<mpsc::UnboundedSender<DiscoveryEvent>>, // 이벤트 리스너
}

```


## 2. `Discovery` 구현
### 1. new
- 네트워크 초기화 및 탐색 프로토콜 설정
- `discv4`, `discv5`, `DNS` 서비스를 설정하고 비동기로 실행

```Rust
    pub async fn new(
    tcp_addr: SocketAddr,
    discovery_v4_addr: SocketAddr,
    sk: SecretKey,
    discv4_config: Option<Discv4Config>,
    discv5_config: Option<reth_discv5::Config>, 
    dns_discovery_config: Option<DnsDiscoveryConfig>,) -> Result<Self, NetworkError>

```
 
- **tcp_addr** : 노드의 TCP 주소
- **discovery_v4_addr** : v4 프로토콜에 사용할 주소
- **sk** : 노드 비밀키 
- **discv4_config , discv5_config, dns_discovery_config** : discovery 설정
---
### Operations
---
#### ① discovery 4 설정
```Rust
    let discv4_future = async {
    let Some(disc_config) = discv4_config else { return Ok((None, None, None)) };
    let (discv4, mut discv4_service) =
        Discv4::bind(discovery_v4_addr, local_enr, sk, disc_config).await.map_err(
            |err| {
                NetworkError::from_io_error(err, ServiceKind::Discovery(discovery_v4_addr))
            },
        )?;
    let discv4_updates = discv4_service.update_stream();
    let discv4_service = discv4_service.spawn();
    Ok((Some(discv4), Some(discv4_updates), Some(discv4_service)))
    };
```
- `discv4_future` : discovery v4 protocol 설정 (비동기 처리)
- `NodeRecord::from_secret_key` : secret key로부터 로컬 노드 정보 생성 후 TCP 포트 설정
- `Discv4::bind` : discovery v4 서비스 시작
- `spawn` : discovery v4 비동기 실행

#### ② discovery 5 설정 
```Rust
        let discv5_future = async {
        let Some(config) = discv5_config else { return Ok::<_, NetworkError>((None, None)) };
        let (discv5, discv5_updates, _local_enr_discv5) = Discv5::start(&sk, config).await?;
        Ok((Some(discv5), Some(discv5_updates.into())))
        };
```
- `discv5_future` : discovery v5 설정 (비동기 처리)
- `Discv5::start` : discovery v5 서비스 시작

#### ③ DNS 설정 
```Rust
    pub async fn new(
    tcp_addr: SocketAddr,
    discovery_v4_addr: SocketAddr,
    sk: SecretKey,
    discv4_config: Option<Discv4Config>,
    discv5_config: Option<reth_discv5::Config>, 
    dns_discovery_config: Option<DnsDiscoveryConfig>,
    ) -> Result<Self, NetworkError>
```
- `DnsDiscoveryService::new_pair` : DNS 설정 (비동기 처리)
- `spawn` : DNS 비동기 실행  

#### ④ `tokio::try_join!`  : discovery 4, discovery 5 병렬 실행
---

```Rust
    let ((discv4, discv4_updates, _discv4_service), (discv5, discv5_updates)) = 
    tokio::try_join!(discv4_future, discv5_future)?;
```
##### ⑤ `return` : discv4, discv5, DNS 관련 객체 
---
```Rust
    Ok(Self {
    discovery_listeners: Default::default(),
    local_enr,
    discv4,
    discv4_updates,
    _discv4_service,
    discv5,
    discv5_updates,
    discovered_nodes: LruMap::new(DEFAULT_MAX_CAPACITY_DISCOVERED_PEERS_CACHE),
    queued_events: Default::default(),
    _dns_disc_service,
    _dns_discovery,
    dns_discovery_updates,
    })
```
### 2. add_listener

```Rust
pub(crate) fn add_listener(&mut self, tx: mpsc::UnboundedSender<DiscoveryEvent>) {
        self.discovery_listeners.push(tx);
    }
```
- `add_listener` : 발견 이벤트 수신 리스너 등록
### 3. notify_listeners
```Rust
fn notify_listeners(&mut self, event: &DiscoveryEvent) {
        self.discovery_listeners.retain_mut(|listener| listener.send(event.clone()).is_ok());
    }
```
- `notify_listeners` : 등록된 모든 listener에게 Event 알림
### 4. update_fork_id
```Rust
pub(crate) fn update_fork_id(&self, fork_id: ForkId) {
        if let Some(discv4) = &self.discv4 {
            discv4.set_eip868_rlp(b"eth".to_vec(), EnrForkIdEntry::from(fork_id))
        }
    }
```
-`update_fork_id` : : Discovery v4 `eth:ForkId` 업데이트
### 5. ban_ip, ban
``` Rust
// ban_ip
pub(crate) fn ban_ip(&self, ip: IpAddr) {
        if let Some(discv4) = &self.discv4 {
            discv4.ban_ip(ip)
        }
        if let Some(discv5) = &self.discv5 {
            discv5.ban_ip(ip)
        }
    }

// ban 
pub(crate) fn ban(&self, peer_id: PeerId, ip: IpAddr) {
        if let Some(discv4) = &self.discv4 {
            discv4.ban(peer_id, ip)
        }
        if let Some(discv5) = &self.discv5 {
            discv5.ban(peer_id, ip)
        }
    }
```
- IP 주소 or Peer ID 차단
### 6. poll 
- `poll` : Discovery 스트림 (v4,v5, DNS)에서 이벤트 폴링하여 처리
```Rust
pub(crate) fn poll(&mut self, cx: &mut Context<'_>) -> Poll<DiscoveryEvent>
```

- `Poll<DiscoveryEvent>` : 폴링 결과로 `DiscoveryEvent` 반환 
    - `Poll::Ready` : 이벤트 준비
    - `Poll::Pending` : 이벤트 준비 X
---
#### ① 이벤트 큐 처리 
```Rust
if let Some(event) = self.queued_events.pop_front() {
    self.notify_listeners(&event);
    return Poll::Ready(event);
}
```
- `queued_events` : 이전에 수신된 이벤트 저장 큐 
- 이벤트 발생 시, 해당 이벤트 큐에 저장 / 처리
#### ② discovery 4 업데이트 스트림 처리
```Rust
while let Some(Poll::Ready(Some(update))) =
    self.discv4_updates.as_mut().map(|updates| updates.poll_next_unpin(cx))
{
    self.on_discv4_update(update)
}
```
-`discv4_updates`가 있을 때, 스트림 폴링해 v4 업데이트 
- 업데이트는 `on_discv4_update` 함수로 전달
#### ③ discovery 5 업데이트 스트림 처리
```Rust
while let Some(Poll::Ready(Some(update))) =
    self.discv4_updates.as_mut().map(|updates| updates.poll_next_unpin(cx))
{
    self.on_discv4_update(update)
}
```
- `on_node_record_update` : 새로운 노드 발견 시 해당 함수로 노드 정보 업데이트 
#### ④ DNS 업데이트 처리 
```Rust
while let Some(Poll::Ready(Some(update))) =
    self.dns_discovery_updates.as_mut().map(|updates| updates.poll_next_unpin(cx))
{
    self.add_discv4_node(update.node_record);
    if let Err(err) = self.add_discv5_node(update.enr) {
        trace!(target: "net::discovery",
            %err,
            "failed adding node discovered by dns to discv5"
        );
    }
    self.on_node_record_update(update.node_record, update.fork_id);
}

```
- 새로운 노드 발견 시 `discv4` 및 `discv5`에 추가 
#### ⑤ 이벤트가 없는 경우
```Rust
if self.queued_events.is_empty() {
    return Poll::Pending
}
```
- 이벤트 없는 경우 : `queued_events`가 비어있으면 `Poll::Pending`반환 

### 7. Stream 
```Rust
impl Stream for Discovery {
    type Item = DiscoveryEvent;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        Poll::Ready(Some(ready!(self.get_mut().poll(cx))))
    }   
}
```
-Discovery 구조체 `Stream`으로 사용 ... event 폴링할 수 있게 한다.

---

## Components

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
