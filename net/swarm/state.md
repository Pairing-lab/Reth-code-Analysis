# Network State 
> 네트워크에서 peer 상태 추적, peer 연결 관리, block 전파  피어와의 연결을 관리하며, 블록을 전파

[File : crates/net/network/src/state.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/state.rs#L72)

### 1. `BlockNumber` 구조체 및 구현 
```Rust
pub(crate) struct BlockNumReader(Box<dyn reth_storage_api::BlockNumReader>);
```
- `BlcokNumReader` : block 번호 읽기

```Rust
impl BlockNumReader {
    pub fn new(reader: impl reth_storage_api::BlockNumReader + 'static) -> Self {
        Self(Box::new(reader))
    }
}
```
- `new` : 새로운 인스턴스 생성 

```Rust
impl fmt::Debug for BlockNumReader {
    ...
}
impl Deref for BlockNumReader {
    ...
}
```
- `Debug` 와 `Deref`를 통해 사용성 향상 

### 2. `NetworkState` 구조체 및 구현
- 네트워크의 peer 간 통신 관리
- 새로운 peer 연결 → 세션 활성화 → 새로운 블록 생성 → 네트워크에 전파 → 요청 처리 및 응답 관리
```Rust
pub struct NetworkState {
    // 활성 peer 목록 관리 (peer 상태 정보 포함)
    active_peers: HashMap<PeerId, ActivePeer>,
    // peer 연결 관리 - 새로운 연결 시도 / 해제 
    peers_manager: PeersManager,
    // 네트워크 이벤트 발생 시 메시지 저장하는 큐 
    queued_messages: VecDeque<StateAction>,
    // blcok 번호 조회하는 client 
    // 특정 peer와의 session에서 block 해시 확인
    client: BlockNumReader,
    // 새로운 peer 발견하고 관리 
    discovery: Discovery,
    // peer로부터 데이터 가져오기 
    state_fetcher: StateFetcher,
}
```
#### ① new
```Rust
impl NetworkState {
    pub(crate) fn new(
        client: BlockNumReader,
        discovery: Discovery,
        peers_manager: PeersManager,
        num_active_peers: Arc<AtomicUsize>,
    ) -> Self {
        let state_fetcher = StateFetcher::new(peers_manager.handle(), num_active_peers);
        Self {
            active_peers: Default::default(),
            peers_manager,
            queued_messages: Default::default(),
            client,
            discovery,
            state_fetcher,
        }
    }
}
```
- `new` : 새로운 인스턴스 생성
-  `StateFetcher` 초기화 후 `NetworkState` 반환

#### ② Peer 관리

```Rust
impl NetworkState {
    pub(crate) fn peers_mut(&mut self) -> &mut PeersManager {
    &mut self.peers_manager
    }
}
```
```Rust
impl NetworkState {
    pub(crate) fn discovery_mut(&mut self) -> &mut Discovery {
        &mut self.discovery
    }
}
```
```Rust
impl NetworkState {
    pub(crate) const fn peers(&self) -> &PeersManager {
        &self.peers_manager
    }
}
```

#### ③ 세션 활성화 
```Rust
impl NetworkState {
    pub(crate) fn on_session_activated(
        &mut self,
        peer: PeerId,
        capabilities: Arc<Capabilities>,
        status: Arc<Status>,
        request_tx: PeerRequestSender,
        timeout: Arc<AtomicU64>,
    ) {
        let block_number =
            self.client.block_number(status.blockhash).ok().flatten().unwrap_or_default();
        self.state_fetcher.new_active_peer(peer, status.blockhash, block_number, timeout);

        self.active_peers.insert(
            peer,
            ActivePeer {
                best_hash: status.blockhash,
                capabilities,
                request_tx,
                pending_response: None,
                blocks: LruCache::new(PEER_BLOCK_CACHE_LIMIT),
            },
        );
    }
}
```
- `on_session_activated` 
    - peer 와 새로운 session 활성화 되었을 때 호출 
    - 위에서 정의한 `BlockNumReader`를 통해  peer가 가진 블록 번호 확인 후 `active_peers`에 추가
    - `StateFetcher`에게 peer 활성화 알리고, peer가 새로운 블록 추적 설정

#### ④ 새로운 블록 전파
```Rust
impl NetworkState{
    pub(crate) fn announce_new_block(&mut self, msg: NewBlockMessage) {
        let num_propagate = (self.active_peers.len() as f64).sqrt() as u64 + 1;

        let number = msg.block.block.header.number;
        let mut count = 0;

        // Shuffle to propagate to a random sample of peers on every block announcement
        let mut peers: Vec<_> = self.active_peers.iter_mut().collect();
        peers.shuffle(&mut rand::thread_rng());

        for (peer_id, peer) in peers {
            if peer.blocks.contains(&msg.hash) {
                continue
            }
            if count < num_propagate {
                self.queued_messages
                    .push_back(StateAction::NewBlock { peer_id: *peer_id, block: msg.clone() });
                if self.state_fetcher.update_peer_block(peer_id, msg.hash, number) {
                    peer.best_hash = msg.hash;
                }
                peer.blocks.insert(msg.hash);

                count += 1;
            }

            if count >= num_propagate {
                break
            }
        }
    }
}
```
- `announce_new_block` : 네트워크에 새로운 블록 검증 후, 해당 블록을 peer들에게 전파

#### ⑤ peer 연결 해제
```Rust
impl NetworkState {
    pub(crate) fn on_session_closed(&mut self, peer: PeerId) {
        self.active_peers.remove(&peer);
        self.state_fetcher.on_session_closed(&peer);
    }
}
```
- `on_session_closed` 
    - peer와의 연결 해제 시 호출
    - `active_peers`에서 peer 상태 제거, `StateFetcher`에게 session 종료 알림
#### 요청 처리 및 응답 관리
```Rust
impl NetworkState{
    fn handle_block_request(&mut self, peer: PeerId, request: BlockRequest) {
        if let Some(ref mut peer) = self.active_peers.get_mut(&peer) {
            let (request, response) = match request {
                BlockRequest::GetBlockHeaders(request) => {
                    let (response, rx) = oneshot::channel();
                    let request = PeerRequest::GetBlockHeaders { request, response };
                    let response = PeerResponse::BlockHeaders { response: rx };
                    (request, response)
                }
                BlockRequest::GetBlockBodies(request) => {
                    let (response, rx) = oneshot::channel();
                    let request = PeerRequest::GetBlockBodies { request, response };
                    let response = PeerResponse::BlockBodies { response: rx };
                    (request, response)
                }
            };
            let _ = peer.request_tx.to_session_tx.try_send(request);
            peer.pending_response = Some(response);
        }
    }
}
```
- `handle_block_request` : peer 요청 처리, response 큐에 추가

### 3. `ActivePeer` 구조체 
```Rust
pub(crate) struct ActivePeer {
    // peer가 알고 있는 최고 블록의 해시
    pub(crate) best_hash: B256,
    #[allow(dead_code)]
    // peer가 지원하는 네트워크 기능
    pub(crate) capabilities: Arc<Capabilities>,
    // peer에게 요청 보내기 위한 통신 채널
    pub(crate) request_tx: PeerRequestSender,
    // peer로부터 대기 중인 응답 
    pub(crate) pending_response: Option<PeerResponse>,
    // peer가 보유한 블록들 추적
    pub(crate) blocks: LruCache<B256>,
}
```
- `NetworkState`에서 peer를 연결하고 해제할 때, ActivePeers에 추가하고 제거하기 위해 정의해둔다. 