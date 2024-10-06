# eth_requests
- P2P 네트워크에서 블록 및 헤더 관리를 위한 모듈
- Ethereum 블록체인 네트워크의 요청을 처리하는 역할

[crates/net/network/src/eth_requests.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/eth_requests.rs)
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