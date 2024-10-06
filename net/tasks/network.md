# Network
> - `NetworkHandle` : 네트워크와 상호작용하는 인터페이스
> - P2P 네트워크의 피어 관리, 트랜잭션 처리, 네트워크 이벤트 처리  

[File : crates/net/network/src/network.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/network.rs)  

### 1. `NetworkHandle` 구조체
```Rust
pub struct NetworkHandle {
    inner: Arc<NetworkInner>,
}
```
### 2. 네트워크 상태 관리 관련
```Rust
pub fn update_status(&self, head: Head) {
    self.send_message(NetworkHandleMessage::StatusUpdate { head });
}
```
- `update_status` : 네트워크 노드의 상태 업데이트
- `Head` 객체 : 최신 블록 정보 전달
```Rust
pub fn set_network_active(&self) {
    self.set_network_conn(NetworkConnectionState::Active);
}
```
- `set_network_active` : 네트워크 활성 상태로 설정해 새로운 연결 가능하도록 함
```Rust
pub fn set_network_hibernate(&self) {
    self.set_network_conn(NetworkConnectionState::Hibernate);
}
```
- `set_network_hibernate` : 네트워크 비활성 상태로 설정해 새로운 연결 막음

```Rust
pub async fn shutdown(&self) -> Result<(), oneshot::error::RecvError> {
    let (tx, rx) = oneshot::channel();
    self.send_message(NetworkHandleMessage::Shutdown(tx));
    rx.await
}
```
- `shutdown` : 네트워크 정상적 종료 

### 3. `peer`관리 관련
```Rust
pub fn add_trusted_peer_id(&self, peer: PeerId) {
    self.send_message(NetworkHandleMessage::AddTrustedPeerId(peer));
}
```
- `add_trusted_peer_kind` : 지정된 peerID를 신뢰할 수 있는 peer로 추가
```Rust
fn add_peer_kind(
    &self,
    peer: PeerId,
    kind: PeerKind,
    tcp_addr: SocketAddr,
    udp_addr: Option<SocketAddr>,
) {
    let addr = PeerAddr::new(tcp_addr, udp_addr);
    self.send_message(NetworkHandleMessage::AddPeerAddress(peer, kind, addr));
}
```
- `add_peer_kind` : 지정된 peerID 추가 후 peer 종류 및 주소 정보 설정
```Rust
async fn get_peer_by_id(&self, peer_id: PeerId) -> Result<Option<PeerInfo>, NetworkError> {
    let (tx, rx) = oneshot::channel();
    let _ = self.manager().send(NetworkHandleMessage::GetPeerInfoById(peer_id, tx));
    Ok(rx.await?)
}
```
- `get_peer_by_id` : 특정 peer 정보 가져옴
```Rust
pub fn disconnect_peer_with_reason(&self, peer: PeerId, reason: DisconnectReason) {
    self.send_message(NetworkHandleMessage::DisconnectPeer(peer, Some(reason)))
}
```
- `disconnect_peer_with_reason`: 특정 피어와 연결 해제 후 이유 반환
```Rust
pub fn reputation_change(&self, peer_id: PeerId, kind: ReputationChangeKind) {
    self.send_message(NetworkHandleMessage::ReputationChange(peer_id, kind));
}
```
- `reputation_change`: peer 평판 변경

### 4. 트랜잭션 관리 관련

```Rust
pub fn send_transactions_hashes(&self, peer_id: PeerId, msg: NewPooledTransactionHashes) {
    self.send_message(NetworkHandleMessage::SendPooledTransactionHashes { peer_id, msg })
}
```
- `send_transactions_hashes` :  특정 피어에게 트랜잭션 해시 전송
```Rust
pub fn send_transactions(&self, peer_id: PeerId, msg: Vec<Arc<TransactionSigned>>) {
    self.send_message(NetworkHandleMessage::SendTransaction {
        peer_id,
        msg: SharedTransactions(msg),
    })
}
```
- `send_transactions` :  특정 피어에게 전체 트랜잭션 데이터 전송
```Rust
pub async fn transactions_handle(&self) -> Option<TransactionsHandle> {
    let (tx, rx) = oneshot::channel();
    let _ = self.manager().send(NetworkHandleMessage::GetTransactionsHandle(tx));
    rx.await.unwrap()
}
```
- `transactions_handle` : 트랜잭션 핸들 가져옴

### 5. 네트워크 이벤트 및 통신 관리 메서든

```Rust
fn event_listener(&self) -> EventStream<NetworkEvent> {
    self.inner.event_sender.new_listener()
}
```
- `event_listener` : 네트워크 이벤트 리스너 생성
```Rust
pub fn send_request(&self, peer_id: PeerId, request: PeerRequest) {
    self.send_message(NetworkHandleMessage::EthRequest { peer_id, request })
}
```
- `send_request` : 특정 peer에게 요청 전송
```Rust
pub fn announce_block(&self, block: NewBlock, hash: B256) {
    self.send_message(NetworkHandleMessage::AnnounceBlock(block, hash))
}
```
- `announce_block` : 네트워크에게 새 블록 알림