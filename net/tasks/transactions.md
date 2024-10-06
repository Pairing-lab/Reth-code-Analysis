# Transactions
- [config.rs](#configrs)
- [contants.rs](#constantsrs)
- [mod.rs](#modrs)
## config.rs
### 트랜잭션 처리 시스템 설정값 정의 
: `TransactionsManagerConfig` , `TransactionsFetcherConfig`

[File : crates/net/network/src/transactions/config.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/transactions/config.rs)

### 1. `TransactionsManagerConfig` 구조체 및 구현
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

