# Fetch Client

- [mod.rs](#modrs)
- [client.rs](#clientrs)


<img width="608" alt="fetch client" src="https://github.com/user-attachments/assets/570099cd-7f6a-4d23-864d-d3d19d15653c">


## mod.rs  
- `StateFetcher` : 네트워크에서 **블록 헤드** 와 **블록 바디** 가져옴
- peer 연결 상태 관리, 다운로드 요청 큐에 저장, peer로부터 데이터 요청 후 응답 처리  
[File: crates/net/network/src/fetch/mod.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/fetch/mod.rs)

### 1. `StateFetcher` 구조체 
```Rust
pub struct StateFetcher {
    // 현재 처리 중인 블록 헤더 요청 저장
    inflight_headers_requests:
        HashMap<PeerId, Request<HeadersRequest, PeerRequestResult<Vec<Header>>>>,
    // 현재 처리 중인 블록 바디 요청 저장
    inflight_bodies_requests:
        HashMap<PeerId, Request<Vec<B256>, PeerRequestResult<Vec<BlockBody>>>>,
    // 연결된 peer 정보 저장 해시맵
    peers: HashMap<PeerId, Peer>,
    // peer 관리자 핸들 : peer와의 상호작용 제어
    peers_handle: PeersHandle,
    // 활성화된 peer 수 추적
    num_active_peers: Arc<AtomicUsize>,
    // 처리 대기 중인 다운로드 요청 큐에 저장
    queued_requests: VecDeque<DownloadRequest>,
    // 다운로드 요청 수신 스트림
    download_requests_rx: UnboundedReceiverStream<DownloadRequest>,
    // 다운로드 요청 전송 채널
    download_requests_tx: UnboundedSender<DownloadRequest>,
}
```
### 2. `Peer` 구조체
```Rust
struct Peer {
    state: PeerState,
    best_hash: B256,
    best_number: u64,
    timeout: Arc<AtomicU64>,
    last_response_likely_bad: bool,
}
```
```Rust
enum PeerState {
    // 대기 중
    Idle,
    // 블록 헤더 요청 처리 중
    GetBlockHeaders,
    // 블록 바디 요청 처리 중
    GetBlockBodies,
    // 세션 종료 대기 중
    Closing,
}
```
### 3. `new_active_peer`
```Rust
pub(crate) fn new_active_peer(
    &mut self,
    peer_id: PeerId,
    best_hash: B256,
    best_number: u64,
    timeout: Arc<AtomicU64>,
) {
    self.peers.insert(
        peer_id,
        Peer {
            state: PeerState::Idle,
            best_hash,
            best_number,
            timeout,
            last_response_likely_bad: false,
        },
    );
}
```
- 새로운 peer 연결 시 호출 
- `PeerState`를 `Idle`로 설정 후 peer 맵에 추가

### 4. `on_session_closed`
```Rust
pub(crate) fn on_session_closed(&mut self, peer: &PeerId) {
    self.peers.remove(peer);
    if let Some(req) = self.inflight_headers_requests.remove(peer) {
        let _ = req.response.send(Err(RequestError::ConnectionDropped));
    }
    if let Some(req) = self.inflight_bodies_requests.remove(peer) {
        let _ = req.response.send(Err(RequestError::ConnectionDropped));
    }
}
```
- peer 세션 종료 시 호출
- peer를 peer 맵에서 제거 후 진행 중이던 요청 취소

### 5. `poll`
```Rust
pub(crate) fn poll(&mut self, cx: &mut Context<'_>) -> Poll<FetchAction> {
    loop {
        let no_peers_available = match self.poll_action() {
            PollAction::Ready(action) => return Poll::Ready(action),
            PollAction::NoRequests => false,
            PollAction::NoPeersAvailable => true,
        };

        loop {
            match self.download_requests_rx.poll_next_unpin(cx) {
                Poll::Ready(Some(request)) => match request.get_priority() {
                    Priority::High => { ... }
                    Priority::Normal => { ... }
                },
                Poll::Ready(None) => { unreachable!("channel can't close") }
                Poll::Pending => break,
            }
        }

        if self.queued_requests.is_empty() || no_peers_available {
            return Poll::Pending
        }
    }
}
```
- 주기적으로 호출
- 대기 중인 요청 처리 및 새로운 요청 수신
- peer 상태 확인 및 요청 처리 가능 여부 결정

### 6. `on_block_headers_response`
```Rust
pub(crate) fn on_block_headers_response(
    &mut self,
    peer_id: PeerId,
    res: RequestResult<Vec<Header>>,
) -> Option<BlockResponseOutcome> {
    let is_error = res.is_err();
    let maybe_reputation_change = res.reputation_change_err();

    let resp = self.inflight_headers_requests.remove(&peer_id);
    let is_likely_bad_response = resp.as_ref().is_some_and(|r| res.is_likely_bad_headers_response(&r.request));

    if let Some(resp) = resp {
        let _ = resp.response.send(res.map(|h| (peer_id, h).into()));
    }

    if let Some(peer) = self.peers.get_mut(&peer_id) {
        peer.last_response_likely_bad = is_likely_bad_response;
        if peer.state.on_request_finished() && !is_error && !is_likely_bad_response {
            return self.followup_request(peer_id)
        }
    }

    maybe_reputation_change.map(|reputation_change| BlockResponseOutcome::BadResponse(peer_id, reputation_change))
}
```
- 블록 헤더 요청 응답 처리 및 응답 평가 
- 잘못된 응답 보낸 peer에게 패널티 부과 

## client.rs
- `FetchClient` : 네트워크에서 데이터를 다운로드하기 위한 클라이언트 구현체
- `블록 헤더`와 `블록 바디`를 peer로부터 요청
- 네트워크와 상호 작용 , peer 평판 관리, 세션 수 관리  

[File: crates/net/network/src/fetch/client.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/fetch/client.rs)

### 1. `FetchClient` 구조체
```Rust
pub struct FetchClient {
    // 다운로드 요청 전송하는 비동기 채널 송신자
    pub(crate) request_tx: UnboundedSender<DownloadRequest>,
    // peer와 상호작용하기 위한 핸들
    pub(crate) peers_handle: PeersHandle,
    // 현재 활성화된 peer 세션 수 추적 카운터
    pub(crate) num_active_peers: Arc<AtomicUsize>,
}

### 2. peer 관리
```Rust
impl DownloadClient for FetchClient {
    // peer 평판 관리 및 잘못된 메시지 보낸 peer에게 패널티 부과
    fn report_bad_message(&self, peer_id: PeerId) {
        self.peers_handle.reputation_change(peer_id, ReputationChangeKind::BadMessage);
    }
    // 현재 연결된 peer 수 반환
    fn num_connected_peers(&self) -> usize {
        self.num_active_peers.load(Ordering::Relaxed)
    }
}
```

### 3. 데이터 요청 및 응답 처리 
#### ① `get_headers_with_priority`
- 블록 헤더 요청 
- `HeadersClient` 트레이트 구현
```Rust
impl HeadersClient for FetchClient {
    type Output = HeadersClientFuture<PeerRequestResult<Vec<Header>>>;

    fn get_headers_with_priority(
        &self,
        // priority에 따라 HeaderRequest 전송
        request: HeadersRequest,
        priority: Priority,
    ) -> Self::Output {
        let (response, rx) = oneshot::channel();
        if self
            .request_tx
            .send(DownloadRequest::GetBlockHeaders { request, response, priority })
            .is_ok()
        {
            // 응답 채널이 열려 있는 경우 FlattenedResponse로 감싼 응답 대기
            Either::Left(FlattenedResponse::from(rx))
        } else {
            // 에러 반환
            Either::Right(future::err(RequestError::ChannelClosed))
        }
    }
}
```
#### ② `get_block_bodies_with_priority`
- 블록 바디 요청
- `BodiesClient` 트레이트 구현
- 블록 헤더 요청과 같은 로직
```Rust
impl BodiesClient for FetchClient {
    type Output = BodiesFut;

    fn get_block_bodies_with_priority(
        &self,
        request: Vec<B256>,
        priority: Priority,
    ) -> Self::Output {
        let (response, rx) = oneshot::channel();
        if self
            .request_tx
            .send(DownloadRequest::GetBlockBodies { request, response, priority })
            .is_ok()
        {
            Box::pin(FlattenedResponse::from(rx))
        } else {
            Box::pin(future::err(RequestError::ChannelClosed))
        }
    }
}
```