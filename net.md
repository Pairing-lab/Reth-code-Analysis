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
### Network Builder
네트워크 설정을 구성하는 빌더 패턴을 구현한 것입니다. 주요 구성 요소는 `NetworkBuilder` 구조체와 그 구현입니다. 이 코드는 네트워크 매니저, 트랜잭션 매니저, 그리고 요청 핸들러를 설정하고 관리하는 기능을 제공합니다.  

[File: crates/net/network/src/builder.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/builder.rs)

#### 1. 상수 정의
```Rust
pub(crate) const ETH_REQUEST_CHANNEL_CAPACITY: usize = 256;
```
- `ETH_REQUEST_CHANNEL_CAPACITY`는 `EthRequestHandler`의 최대 채널 용량을 256으로 설정합니다. 이는 악의적인 10MB 크기의 요청이 256개일 경우 2.6GB의 메모리를 차지할 수 있음을 의미합니다.

#### 2. `NetworkBuilder` 구조체 

```Rust
pub struct NetworkBuilder<Tx, Eth> {
    pub(crate) network: NetworkManager,
    pub(crate) transactions: Tx,
    pub(crate) request_handler: Eth,
}
```
- `NetworkBuilder`는 네트워크 매니저, 트랜잭션 매니저, 요청 핸들러를 포함하는 구조체입니다. 제네릭 타입 Tx와 Eth를 사용하여 다양한 트랜잭션 매니저와 요청 핸들러를 지원합니다.

#### 3.  `NetworkBuilder` 구현
```Rust
impl<Tx, Eth> NetworkBuilder<Tx, Eth> {
    // 여러 메서드들...
}
```

- **split**: 구조체를 분해하여 개별 필드를 반환합니다.
- **network**: 네트워크 매니저에 대한 참조를 반환합니다.
- **network_mut**: 네트워크 매니저에 대한 가변 참조를 반환합니다.
- handle: 네트워크 핸들을 반환합니다.
- **split_with_handle**: 구조체를 분해하여 네트워크 핸들과 개별 필드를 반환합니다.
- **transactions**: 새로운 - TransactionsManager를 생성하고 네트워크에 연결합니다.
- **equest_handler**: 새로운 EthRequestHandler를 생성하고 네트워크에 연결합니다.

#### 사용 예제 
```Rust
let network_manager = NetworkManager::new();
let builder = NetworkBuilder {
    network: network_manager,
    transactions: (),
    request_handler: (),
};

// 트랜잭션 매니저 설정
let pool = MyTransactionPool::new();
let transactions_manager_config = TransactionsManagerConfig::default();
let builder = builder.transactions(pool, transactions_manager_config);

// 요청 핸들러 설정
let client = MyClient::new();
let builder = builder.request_handler(client);

// 네트워크 매니저와 핸들 얻기
let (network_handle, network_manager, transactions, request_handler) = builder.split_with_handle();
```

### Network Config
네트워크 초기화 설정 지원 모듈  
[File: crates/net/network/src/config.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/config.rs)

#### 1. `NetworkConfig` 구조체
- 네트워크 초기화에 필요한 모든 설정
- **client** : 체인과 상호작용하는 client
- **secret_key** : 노드의 비밀 키
- **boot_nodes**: 부트 노드의 집합; 네트워크 탐색 시 사용할 기본 노드들.
- **discovery_v4_config, dns_discovery_config**: 네트워크 디스커버리 설정.
- **peers_config, sessions_config**: 피어 및 세션 관리 설정
- **chain_spec**: 네트워크가 사용할 체인 스펙.
- **fork_filter**: 세션 인증에 사용할 포크 필터.
- **block_import**: 블록 가져오기를 처리하는 객체.
- **network_mode**: 네트워크가 사용 중인 모드(POS 또는 POW).
- **executor**: 네트워크 작업을 비동기로 처리할 실행기(executor).

#### 2. `NetworkConfigBuilder` 구조체
- `NetworkConfig` 빌드 위한 빌더 패턴 구현
- 단계별 설정 메서드 제공
- 기본값 설정 통한 초기화 편의 메서드 포함  
---  

**주요 메서드**  
- **new(secret_key)**: 새로운 빌더 인스턴스 생성 / 기본값 설정  
- **chain_spec, network_mode, set_head, hello_message** : 체인 스펙, 네트워크 모드, 헤드 정보, HelloMessage 설정  
- **set_addrs, listener_addr, discovery_addr** : 네트워크 리스너 및 디스커버리 주소 설정  
- **build(client)**: 설정을 기반으로 `NetworkConfig` 객체 생성.

#### 3. `NetworkMode` 열거형
- 네트워크 모드 정의   
: `Work` /  `Stake`
- `is_stake` 메서드로 모드 확인 가능

#### 4. 테스트 모듈 
- `NetworkConfig`와 `NetworkConfigBuilder` 기능 테스트
- `test_network_dns_defaults`, `test_network_fork_filter_default`

---
## Transactions

## config.rs
### 트랜잭션 처리 시스템 설정값 정의 
: `TransactionsManagerConfig` , `TransactionsFetcherConfig`

[File : crates/net/network/src/transactions/config.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/transactions/config.rs)

#### 1. `TransactionsManagerConfig` 구조체 및 구현
: transaction 관리하는 최상위 설정
```Rust
pub struct TransactionsManagerConfig {
    pub transaction_fetcher_config: TransactionFetcherConfig,
    pub max_transactions_seen_by_peer_history: u32,
}

impl Default for TransactionsManagerConfig {
    fn default() -> Self {
        Self {
            transaction_fetcher_config: TransactionFetcherConfig::default(),
            max_transactions_seen_by_peer_history: DEFAULT_MAX_COUNT_TRANSACTIONS_SEEN_BY_PEER,
        }
    }
}
```
- `transaction_fetcher_config` : `TransactionFetcherConfig`에 위임 ... 트랜잭션 가지고 오는 방식 정의 
- `max_transactions_seen_by_peer_history` : peer가 본 transaction의 개수 저장 

### 2. `TransactionsFetcherConfig` 구조체 및 구현
: Transaction을 가져오는 방식에 대한 세부사항을 정의 
```Rust
pub struct TransactionFetcherConfig {
    // 한번에 처리할 수 있는 트랜잭션 요청의 최대 개수 정의 
    pub max_inflight_requests: u32,
    // 각 피어가 동시에 진행할 수 있는 트랜잭션 요청의 최대 개수 
    pub max_inflight_requests_per_peer: u8,
    // 트랜잭션 응답 크기 제한 
    pub soft_limit_byte_size_pooled_transactions_response: usize,
    // 요청 시 트랜잭션 응답의 크기 제한
    pub soft_limit_byte_size_pooled_transactions_response_on_pack_request: usize,
    // 완료되지 않은 트랜잭션의 해시의 크기 정의 
    pub max_capacity_cache_txns_pending_fetch: u32,
}


impl Default for TransactionFetcherConfig {
    fn default() -> Self {
        Self {
            max_inflight_requests: DEFAULT_MAX_COUNT_CONCURRENT_REQUESTS,
            max_inflight_requests_per_peer: DEFAULT_MAX_COUNT_CONCURRENT_REQUESTS_PER_PEER,
            soft_limit_byte_size_pooled_transactions_response:
                SOFT_LIMIT_BYTE_SIZE_POOLED_TRANSACTIONS_RESPONSE,
            soft_limit_byte_size_pooled_transactions_response_on_pack_request:
                DEFAULT_SOFT_LIMIT_BYTE_SIZE_POOLED_TRANSACTIONS_RESP_ON_PACK_GET_POOLED_TRANSACTIONS_REQ,
            max_capacity_cache_txns_pending_fetch: DEFAULT_MAX_CAPACITY_CACHE_PENDING_FETCH,
        }
    }
}
```
---
## constants.rs
#### 트랜잭션과 관련된 상수값들을 미리 정의
[File : crates/net/network/src/transactions/constants.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/transactions/constants.rs)

- 브로드캐스트 관련 
- 요청 - 응답 관련
- `TransactionsManager` 관련
- `TransactionsFetcher` 관련 
---
## mod.rs

### 1. `TransactionsHandle` 구조체 및 구현 
- `TransactionsHandle` =  트랜잭션 관련 명령을  `TransactionsManager`에게 전달하는 **인터페이스** 역할 

```Rust
pub struct TransactionsHandle {
    manager_tx: mpsc::UnboundedSender<TransactionsCommand>,
}
```
- `manager_tx`: TransactionsCommand 보내는 채널 
- 이 채널을 통해 `TransactionsManager`로 보낼 수 있는 것

### ① TransactionsCommand Sending Methods
```Rust
impl TransactionsHandle {
    fn send(&self, cmd: TransactionsCommand) {
        let _ = self.manager_tx.send(cmd);
    }
}
```
- `send` : `TransactionsCommand`를 `manager_tx` 통해 비동기적 전송

```Rust
impl TransactionsHandle {
    pub fn propagate(&self, hash: TxHash) {
        self.send(TransactionsCommand::PropagateHash(hash))
    }
}
```
- `propagate` : 특정 트랜잭션 해시 → 다른 peer들에게 전파
```Rust
impl TransactionsHandle {
    pub fn propagate_hash_to(&self, hash: TxHash, peer: PeerId) {
        self.propagate_hashes_to(Some(hash), peer)
    }
}
```
- `propagate_hash_to` : 트랜잭션 해시 → 특정 peer에게 전파 (transactionPool에 해당 해시가 있을 경우에만)
```Rust
impl TransactionsHandle {
    pub fn propagate_hashes_to(&self, hash: impl IntoIterator<Item = TxHash>, peer: PeerId) {
        self.send(TransactionsCommand::PropagateHashesTo(hash.into_iter().collect(), peer))
    }
}
```
- `propagate_hashes_to` : 여러 트랜잭션 해시들 → 특정 peer에게 전파
```Rust
impl TransactionsHandle {
    pub fn propagate_transactions_to(&self, transactions: Vec<TxHash>, peer: PeerId) {
        self.send(TransactionsCommand::PropagateTransactionsTo(transactions, peer))
    }
}
```
- `propagate_transactions_to` : 전체 트랜잭션 → 특정 peer에게 전파

### ② Peer Handling Methods
```Rust
impl TransactionsHandle {
    async fn peer_handle(&self, peer_id: PeerId) -> Result<Option<PeerRequestSender>, RecvError> {
        let (tx, rx) = oneshot::channel();
        self.send(TransactionsCommand::GetPeerSender { peer_id, peer_request_sender: tx });
        rx.await
    }
}
```
- `peer_handle` 
    - 특정 peer의 `PeerRequestSender`를 가지고 온다.
    - `TransactionsCommand::GetPeerSender` 명령을 전송하면 해당 피어의 `PeerRequestSender`를 응답으로 받을 수 있다.
```Rust
impl TransactionsHandle {
    pub async fn get_pooled_transactions_from(
        &self,
        peer_id: PeerId,
        hashes: Vec<B256>,
    ) -> Result<Option<Vec<PooledTransactionsElement>>, RequestError> {
        let Some(peer) = self.peer_handle(peer_id).await? else { return Ok(None) };

        let (tx, rx) = oneshot::channel();
        let request = PeerRequest::GetPooledTransactions { request: hashes.into(), response: tx };
        peer.try_send(request).ok();

        rx.await?.map(|res| Some(res.0))
    }
}
```
- `get_pooled_transactions_from` : 특정 peer에 transaction 직접 요청

### ③ Peer and Transaction Information Requests

```Rust 
impl TransactionsHandle {
    pub async fn get_active_peers(&self) -> Result<HashSet<PeerId>, RecvError> {
        let (tx, rx) = oneshot::channel();
        self.send(TransactionsCommand::GetActivePeers(tx));
        rx.await
    }
}
```
- `get_active_peers` : 활성화된 peer ID 요청
```Rust
impl TransactionsHandle {
    pub async fn get_transaction_hashes(
        &self,
        peers: Vec<PeerId>,
    ) -> Result<HashMap<PeerId, HashSet<TxHash>>, RecvError> {
        let (tx, rx) = oneshot::channel();
        self.send(TransactionsCommand::GetTransactionHashes { peers, tx });
        rx.await
    }
}
```
- `get_transaction_hashes` : 특정 peer들에게 알고 있는 transaction 해시 목록 요청
```Rust
impl TransactionsHandle {
    pub async fn get_peer_transaction_hashes(
        &self,
        peer: PeerId,
    ) -> Result<HashSet<TxHash>, RecvError> {
        let res = self.get_transaction_hashes(vec![peer]).await?;
        Ok(res.into_values().next().unwrap_or_default())
    }
}
```
- `get_peer_transaction_hashes` : 특정 peer에게 알고 있는 transaction 해시 목록 요청

### 2. `TransactionsManager` 구조체 및 구현 

```Rust
impl<Pool: TransactionPool> TransactionsManager<Pool> {
    pub fn new(
        // 외부 네트워크와의 상호작용 , peer와 통신 기능 제공
        network: NetworkHandle,
        // transaction 유효성 검사, 새로운 transaction 추가
        pool: Pool,
        from_network: mpsc::UnboundedReceiver<NetworkTransactionEvent>,
        transactions_manager_config: TransactionsManagerConfig,
    ) -> Self {
        let (command_tx, command_rx) = mpsc::unbounded_channel();
        // transaction 가지고 오는 요청, 누락된 트랜잭션 처리 
        let transaction_fetcher = TransactionFetcher::with_transaction_fetcher_config(
            &transactions_manager_config.transaction_fetcher_config,
        );
        let pending = pool.pending_transactions_listener();
        
        Self {
            pool,
            network,
            network_events: network.event_listener(),
            transaction_fetcher,
            command_tx,
            command_rx: UnboundedReceiverStream::new(command_rx),
            //대기 transaction stream : 네트워크에 전파할 수 있는 상태
            pending_transactions: ReceiverStream::new(pending),
            transaction_events: UnboundedMeteredReceiver::new(
                from_network,
                NETWORK_POOL_TRANSACTIONS_SCOPE,
            ),
            metrics: TransactionsManagerMetrics::default(),
            // 나머지 초기화 생략
        }
    }
}

```
## Methods
### ① 기본 구성 
----

### new
- #### 네트워크 핸들링 
    ```Rust
    let network_events = network.event_listener();
    ```
    - 네트워크 이벤트 리스너 설정 
    - 트랜잭션 관련 네트워크 이벤트 수신

- #### command 채널 설정
    ```Rust
    let (command_tx, command_rx) = mpsc::unbounded_channel();
    ```
    - `command_tx` (송신자) / `command_rx`(수신자) 채널 생성
    - 외부에서 `TransactionsManager`에게 명령을 보낼 수 있고, `TransactionsManager`도 이를 처리할 준비가 됨

- #### transaction_fetch 설정
    ```Rust
    let transaction_fetcher = TransactionFetcher::with_transaction_fetcher_config(
    &transactions_manager_config.transaction_fetcher_config,
    );
    ``` 
- #### pending_transactions 처리
    ```Rust
    let pending = pool.pending_transactions_listener();
    ```
    - transactionPool 이 pending_transaction을 추가할 때 listener 설정

- #### metrics 설정
    ```Rust
    let pending_pool_imports_info = PendingPoolImportsInfo::default();
    let metrics = TransactionsManagerMetrics::default();
    metrics.capacity_pending_pool_imports.increment(pending_pool_imports_info.max_pending_pool_imports as u64);
    ```
    -`TransactionsManagerMetrics` : `TransactionsManager` 발생 이벤트 추적
    - `pending_pool_imports_info` : pending_transactioins 가져오기 설정하고 metrics로 추적

###  handle
```Rust
 pub fn handle(&self) -> TransactionsHandle {
        TransactionsHandle { manager_tx: self.command_tx.clone() }
    }
```
- `TransactionsHandle` 생성
---

### ② Transactions 전파 및 관리
---

### on_new_pending_transactions
```Rust
fn on_new_pending_transactions(&mut self, hashes: Vec<TxHash>) {
        if self.network.is_initially_syncing() || self.network.tx_gossip_disabled() {
            return;
        }
        let propagated = self.propagate_transactions(
            self.pool.get_all(hashes).into_iter().map(PropagateTransaction::new).collect(),
        );
        self.pool.on_propagated(propagated);
    }
```
- `on_new_pending_transactions` 
    - 새로 대기 중인 트랜잭션을 처리
    - 네트워크 전파 준비

### propagate_transactions
-  `propagate_transactions` : transaction을 peer에게 전파
```Rust
fn propagate_transactions(
    // 초기화 
        &mut self,
        to_propagate: Vec<PropagateTransaction>,
    ) -> PropagatedTransactions {
        let mut propagated = PropagatedTransactions::default();
        if self.network.tx_gossip_disabled() {
            return propagated;
        }
        // peer 수 계산
        let max_num_full = (self.peers.len() as f64).sqrt().round() as usize;
        // peer 순회 및 transactions 전파 준비
        for (peer_idx, (peer_id, peer)) in self.peers.iter_mut().enumerate() {
            let mut builder = if peer_idx > max_num_full {
                PropagateTransactionsBuilder::pooled(peer.version)
            } else {
                PropagateTransactionsBuilder::full(peer.version)
            };
            // transactions 확인 및 추가
            for tx in &to_propagate {
                if !peer.seen_transactions.contains(&tx.hash()) {
                    builder.push(tx);
                }
            }
            // 전파할 transaction build 및 전송 
            let PropagateTransactions { pooled, full } = builder.build();
            if let Some(new_pooled_hashes) = pooled {
                new_pooled_hashes
                    .truncate(SOFT_LIMIT_COUNT_HASHES_IN_NEW_POOLED_TRANSACTIONS_BROADCAST_MESSAGE);
                self.network.send_transactions_hashes(*peer_id, new_pooled_hashes);
            }
            if let Some(new_full_transactions) = full {
                self.network.send_transactions(*peer_id, new_full_transactions);
            }
        }
        propagated
    }
```
---
### ③ Transactions 가져오기
---
### on_get_pooled_transactions
- peer로부터 받은 transactions 요청을 가져와 반환 
```Rust
fn on_get_pooled_transactions(
        &mut self,
        peer_id: PeerId,
        request: GetPooledTransactions,
        response: oneshot::Sender<RequestResult<PooledTransactions>>,
    ) {
        // peer 존재 확인
        if let Some(peer) = self.peers.get_mut(&peer_id) {
            // 네트워크 상태 확인
            if self.network.tx_gossip_disabled() {
                let _ = response.send(Ok(PooledTransactions::default()));
                return;
            }
            // transactions 검색
            let transactions = self.pool.get_pooled_transaction_elements(
                request.0,
                GetPooledTransactionLimit::ResponseSizeSoftLimit(
                    self.transaction_fetcher.info.soft_limit_byte_size_pooled_transactions_response,
                ),
            );
            // 해당 peer가 transactions를 봤다고 기록
            peer.seen_transactions.extend(transactions.iter().map(|tx| *tx.hash()));
            // peer에게 전송 
            let resp = PooledTransactions(transactions);
            let _ = response.send(Ok(resp));
        }
    }
```
---
### ④ Metrics 업데이트
```Rust 
fn update_poll_metrics(&self, start: Instant, poll_durations: TxManagerPollDurations){
    ...
}
```
 - 메트릭 업데이트를 통해 `TransactionsManager` 동작 모니터링 

// to do : fetcher.rs, validation.rs

---
## ETH Requests

- P2P 네트워크에서 블록 및 헤더 관리를 위한 모듈
- Ethereum 블록체인 네트워크의 요청을 처리하는 역할


[File: crates/net/network/src/eth_requests.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/eth_requests.rs)

## 1. 상수 정의 
: 요청의 양이 많을  경우 대비해 크기 제한 
```Rust
const MAX_RECEIPTS_SERVE: usize = 1024;
const MAX_HEADERS_SERVE: usize = 1024;
const MAX_BODIES_SERVE: usize = 1024;
const SOFT_RESPONSE_LIMIT: usize = 2 * 1024 * 1024;
```
- `MAX_RECEIPTS_SERVE`, `MAX_HEADERS_SERVE`, `MAX_BODIES_SERVE` : 각각 최대 1024개의 영수증, 블록 헤더 , 블록 바디 제공 상수
- `SOFT_RESPONSE_LIMIT` : 응답 최대 크기 2MB로 제한 
---

## 2. `EthRequestHandler` 구조체 
```Rust
pub struct EthRequestHandler<C> {
    client: C,
    #[allow(dead_code)]
    peers: PeersHandle,
    incoming_requests: ReceiverStream<IncomingEthRequest>,
    metrics: EthRequestHandlerMetrics,
}
```
- `client` : Chain과 상호작용 할 수 있는 client 타입
- `peers` : peer 보고에 사용
- `incoming_requests` : Network Manager 요청 받는 스트림
- `metrics` : 요청 처리에 대한 metric 기록

## 3. `EthRequestHandler` 구현
### 1. new
: `EthRequestHandler` 인스턴스 생성 함수 
```Rust
impl<C> EthRequestHandler<C> {
    /// Create a new instance
    pub fn new(client: C, peers: PeersHandle, incoming: Receiver<IncomingEthRequest>) -> Self {
        Self {
            client,
            peers,
            incoming_requests: ReceiverStream::new(incoming),
            metrics: Default::default(),
        }
    }
}
```
- `client` : Ethereum 데이터 제공하는 클라이언트
- `peers` : Peer Handle ... 다른 peer와의 상호작용에 사용
- `incoming_requests` : 이더리움 요청을 처리하는 채널 .  `ReceiverStream` 으로 변환해 비동기 처리   
- `metrics` : 요청 처리 통계 기록 

### 2. get_headers_response
: `GetBlockHeaders` 요청에 따라 블록 헤더 리스트 반환   
- 블록 해시 / 블록 번호에서 시작해 블록 헤더 차례대로 조회 
- `start_block` , `limit` , `skip`, `direction` : 블록 헤더 탐색 하고 필요한 만큼 가지고 온다. 
```Rust
fn get_headers_response(&self, request: GetBlockHeaders) -> Vec<Header> {
        let GetBlockHeaders { start_block, limit, skip, direction } = request;

        let mut headers = Vec::new();

        let mut block: BlockHashOrNumber = match start_block {
            BlockHashOrNumber::Hash(start) => start.into(),
            BlockHashOrNumber::Number(num) => {
                let Some(hash) = self.client.block_hash(num).unwrap_or_default() else {
                    return headers
                };
                hash.into()
            }
        };

        let skip = skip as u64;
        let mut total_bytes = 0;

        for _ in 0..limit {
            if let Some(header) = self.client.header_by_hash_or_number(block).unwrap_or_default() {
                match direction {
                    HeadersDirection::Rising => {
                        if let Some(next) = (header.number + 1).checked_add(skip) {
                            block = next.into()
                        } else {
                            break
                        }
                    }
                    HeadersDirection::Falling => {
                        if skip > 0 {
                            // prevent under flows for block.number == 0 and `block.number - skip <
                            // 0`
                            if let Some(next) =
                                header.number.checked_sub(1).and_then(|num| num.checked_sub(skip))
                            {
                                block = next.into()
                            } else {
                                break
                            }
                        } else {
                            block = header.parent_hash.into()
                        }
                    }
                }

                total_bytes += header.length();
                headers.push(header);

                if headers.len() >= MAX_HEADERS_SERVE || total_bytes > SOFT_RESPONSE_LIMIT {
                    break
                }
            } else {
                break
            }
        }

        headers
    }
```

### 3. on_headers_request
: peer로부터 받은 블록 헤더 요청 처리 
```Rust
fn on_headers_request(
        &self,
        _peer_id: PeerId,
        request: GetBlockHeaders,
        response: oneshot::Sender<RequestResult<BlockHeaders>>,
    ) {
        self.metrics.eth_headers_requests_received_total.increment(1);
        let headers = self.get_headers_response(request);
        let _ = response.send(Ok(BlockHeaders(headers)));
    }
```
- `oneshot::Sender` :  가지고 온 헤더 목록을 피어에게 전달
- `metrics` : 헤더 요청 수 기록 

### 4. on_bodies_request
- peer로부터 받은 블록 바디 요청 처리
- 블록 해시 목록 기반으로 블록 바디 가지고 와서 반환
```Rust
fn on_bodies_request(
        &self,
        _peer_id: PeerId,
        request: GetBlockBodies,
        response: oneshot::Sender<RequestResult<BlockBodies>>,
    ) {
        self.metrics.eth_bodies_requests_received_total.increment(1);
        let mut bodies = Vec::new();

        let mut total_bytes = 0;

        for hash in request.0 {
            if let Some(block) = self.client.block_by_hash(hash).unwrap_or_default() {
                let body: BlockBody = block.into();

                total_bytes += body.length();
                bodies.push(body);

                if bodies.len() >= MAX_BODIES_SERVE || total_bytes > SOFT_RESPONSE_LIMIT {
                    break
                }
            } else {
                break
            }
        }

        let _ = response.send(Ok(BlockBodies(bodies)));
    }
```
- 요청에 포함된 블록 해시 순회해서 블록 바디 가지고 온다.
- `oneshot::Sender` : 결과 응답 
### 5. on_receipts_request
- 특정 peer로부터 받은 transaction receipt 요청 처리
- 블록 해시 기반으로 receipt 조회해서  peer에게 반환
```Rust
fn on_receipts_request(
        &self,
        _peer_id: PeerId,
        request: GetReceipts,
        response: oneshot::Sender<RequestResult<Receipts>>,
    ) {
        self.metrics.eth_receipts_requests_received_total.increment(1);

        let mut receipts = Vec::new();

        let mut total_bytes = 0;

        for hash in request.0 {
            if let Some(receipts_by_block) =
                self.client.receipts_by_block(BlockHashOrNumber::Hash(hash)).unwrap_or_default()
            {
                let receipt = receipts_by_block
                    .into_iter()
                    .map(|receipt| receipt.with_bloom())
                    .collect::<Vec<_>>();

                total_bytes += receipt.length();
                receipts.push(receipt);

                if receipts.len() >= MAX_RECEIPTS_SERVE || total_bytes > SOFT_RESPONSE_LIMIT {
                    break
                }
            } else {
                break
            }
        }

        let _ = response.send(Ok(Receipts(receipts)));
    }
```
- `reqeust.0` 에 포함된 블록 해시 목록 순회해서 블록에 포함된 receipt 조회 
- `with_bloom()` : 각 receipt 마다 호출해서 bloom 필터 포함

## 3. `Future` 구현 
- `EthRequestHandler`가 비동기로 동작할 수 있도록 해준다.
-  Ethereum 요청 스트림 폴링하며 처리
```Rust
impl<C> Future for EthRequestHandler<C>
where
    C: BlockReader + HeaderProvider + Unpin,
```
### ① poll
```Rust
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>
```
- 계속해서 새로운 요청을 폴링하고 처리하며, 처리할 새로운 요청이 없을 때까지 대기 상태
- 비동기 작업 상태 확인 : `Pending` / `Ready`

### ② 요청 스트림 처리 
```Rust 
let mut acc = Duration::ZERO;
let maybe_more_incoming_requests = metered_poll_nested_stream_with_budget!(
    acc,
    "net::eth",
    "Incoming eth requests stream",
    DEFAULT_BUDGET_TRY_DRAIN_DOWNLOADERS,
    this.incoming_requests.poll_next_unpin(cx),
    |incoming| {
        match incoming {
            IncomingEthRequest::GetBlockHeaders { peer_id, request, response } => {
                this.on_headers_request(peer_id, request, response)
            }
            IncomingEthRequest::GetBlockBodies { peer_id, request, response } => {
                this.on_bodies_request(peer_id, request, response)
            }
            IncomingEthRequest::GetNodeData { .. } => {
                this.metrics.eth_node_data_requests_received_total.increment(1);
            }
            IncomingEthRequest::GetReceipts { peer_id, request, response } => {
                this.on_receipts_request(peer_id, request, response)
            }
        }
    },
);
```
- `metered_poll_nested_stream_with_budget!` 매크로 
    - 요청 스트림을 폴링하면서 요청 처리
    - 각 요청은 미리 정해진 예산(`DEFAULT_BUDGET_TRY_DRAIN_DOWNLOADERS`) : 한 번의 폴링에서 얼마나 많은 요청을 처리할지 결정하는 기준
    - 처리 시간이 acc에 누적 → 메트릭스(`metrics.acc_duration_poll_eth_req_handler`)에 기록

- `IncomingEthRequest`: Ethereum 요청. 종류에 따라 분기 처리
    - `GetBlockHeaders`: 블록 헤더 요청을 처리하는 `on_headers_request`를 호출
    - `GetBlockBodies`: 블록 바디 요청을 처리하는 `on_bodies_request`를 호출
    - `GetNodeData`: 노드 데이터를 요청하는 경우, 단순히 요청 수를 기록
    - `GetReceipts`: 영수증 데이터를 요청하는 경우, `on_receipts_request`를 호출하여 처리

## 4. Enum IncomingEthRequest
: 네트워크에서 위임된 모든 eth 요청
```Rust
pub enum IncomingEthRequest {
    GetBlockHeaders {
        peer_id: PeerId,
        request: GetBlockHeaders,
        response: oneshot::Sender<RequestResult<BlockHeaders>>,
    },
    GetBlockBodies {
        peer_id: PeerId,
        request: GetBlockBodies,
        response: oneshot::Sender<RequestResult<BlockBodies>>,
    },
    GetNodeData {
        peer_id: PeerId,
        request: GetNodeData,
        response: oneshot::Sender<RequestResult<NodeData>>,
    },
    GetReceipts {
        peer_id: PeerId,
        request: GetReceipts,
        response: oneshot::Sender<RequestResult<Receipts>>,
    },
}
```
- `GetBlockHeaders` , `GetBlockBodies` , `GetNodeData`, `GetReceipts` 요청 

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
