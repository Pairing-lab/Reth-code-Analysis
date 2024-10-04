# peer 
- [peer.rs](#peer.rs)
- [peer](#peer)
    - [lib.rs](#lib.rs)
    - [node_record.rs](#node_record.rs)
    - [trusted_peer.rs](#trusted_peer.rs)
## peer.rs  
- **네트워크**에서 peer 관리 : 
    - peer와의 연결 및 해제
    - peer의 상태 변화
    - 평판 시스템 

[File : crates/net/network/src/peer.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/peers.rs)  
### 1. `PeersManager` 구조체 및 구현 

```Rust
pub struct PeersManager {
    // 네트워크에 알려진 모든 peer 저장 
    peers: HashMap<PeerId, Peer>,
    // 신뢰할 수 있는 peerId 저장 
    trusted_peer_ids: HashSet<PeerId>,
    // peer 명령 주고 받는 채널
    manager_tx: mpsc::UnboundedSender<PeerCommand>,
    handle_rx: UnboundedReceiverStream<PeerCommand>,
    // 관리자가 polling 될 때까지 액션 버퍼링
    queued_actions: VecDeque<PeerAction>,
    // 주기적으로 연결 시도할 때 인터벌
    refill_slots_interval: Interval,
    // 평판 변경 가중치 
    reputation_weights: ReputationChangeWeights,
    // 현재 slot 상태 추적
    connection_info: ConnectionInfo,
    // 블랙리스트 IP / peerId 추적 
    ban_list: BanList,
    // backoff된 peer 추적
    backed_off_peers: HashMap<PeerId, std::time::Instant>,
    // 블랙 리스트 해제 인터벌 
    release_interval: Interval,
    // 블랙리스트 기간
    ban_duration: Duration,
    // 백오프 기간
    backoff_durations: PeerBackoffDurations,
    // 신뢰할 수 없는 peer 연결 허용 여부 
    trusted_nodes_only: bool,
    // 마지막 tick 호출 시간
    last_tick: Instant,
    // peer 포기 전 최대 백오프 시도 횟수
    max_backoff_count: u8,
    // 노드 연결 상태 추적 
    net_connection_state: NetworkConnectionState,
}
```
#### ① 생성 및 초기화 

```Rust
impl PeersManager {
    // 주어진 설정(config)으로 새로운 PeersManager 인스턴스를 생성
    pub fn new(config: PeersConfig) -> Self {
        let PeersConfig {
            refill_slots_interval,
            connection_info,
            reputation_weights,
            ban_list,
            ban_duration,
            backoff_durations,
            trusted_nodes,
            trusted_nodes_only,
            basic_nodes,
            max_backoff_count,
        } = config;
        // 설정에 따라 PeersManager의 초기 상태를 정의
    }
}
- `new` : 
```
```Rust
impl PeersManager {
    // 이 타입에 명령을 전송할 수 있는 새로운 [`PeersHandle`]을 반환
    pub(crate) fn handle(&self) -> PeersHandle {
        PeersHandle::new(self.manager_tx.clone())
    }
}
```
```Rust
impl PeersManager {
    // 네트워크 상태를 `ShuttingDown`으로 설정
    pub fn on_shutdown(&mut self) {
        self.net_connection_state = NetworkConnectionState::ShuttingDown;
    }
}
```
```Rust
impl PeersManager {
    // 네트워크 상태 변화를 추적
    pub fn on_network_state_change(&mut self, state: NetworkConnectionState) {
        self.net_connection_state = state;
    }
}
```
```Rust
impl PeersManager {
    // 현재 네트워크 연결 상태를 반환
    pub const fn connection_state(&self) -> &NetworkConnectionState {
        &self.net_connection_state
    }
}
```

#### ② `PeersManager` 상태 관리
```Rust
impl PeersManager {
    // 피어 집합에서 피어의 수를 반환
    #[inline]
    pub(crate) fn num_known_peers(&self) -> usize {
        self.peers.len()
    }
}
```
```Rust
impl PeersManager {
    // 현재 활성화된 인바운드(inbound) 연결의 수를 반환
    #[inline]
    pub(crate) const fn num_inbound_connections(&self) -> usize {
        self.connection_info.num_inbound
    }
}
```
```Rust
impl PeersManager {
    // 현재 활성화된 아웃바운드(outbound) 연결의 수를 반환
    #[inline]
    pub(crate) const fn num_outbound_connections(&self) -> usize {
        self.connection_info.num_outbound
    }
}
```
```Rust
impl PeersManager {
    // 현재 대기 중인 아웃바운드 연결의 수를 반환
    #[inline]
    pub(crate) const fn num_pending_outbound_connections(&self) -> usize {
        self.connection_info.num_pending_out
    }
}
```
```Rust
impl PeersManager {
    // 현재 백오프된 피어의 수를 반환
    pub(crate) fn num_backed_off_peers(&self) -> usize {
        self.backed_off_peers.len()
    }
}
```

#### ③ peer 추가 및 제거 
```Rust
impl PeersManager {
    // 새로 발견된 피어를 추가
    pub(crate) fn add_peer(&mut self, peer_id: PeerId, addr: PeerAddr, fork_id: Option<ForkId>) {
        self.add_peer_kind(peer_id, PeerKind::Basic, addr, fork_id)
    }
}
```
```Rust
impl PeersManager {
    // 주어진 피어 ID를 신뢰할 수 있는 피어로 표시
    pub(crate) fn add_trusted_peer_id(&mut self, peer_id: PeerId) {
        self.trusted_peer_ids.insert(peer_id);
    }
}
```
```Rust
impl PeersManager {
    // 피어 집합에서 노드를 제거
    pub(crate) fn remove_peer(&mut self, peer_id: PeerId) {
        let Entry::Occupied(entry) = self.peers.entry(peer_id) else { return };
        if entry.get().is_trusted() {
            return;
        }
        // 피어 제거 로직 처리
    }
}
```
```Rust
impl PeersManager {
    // 신뢰할 수 있는 피어 집합에서 노드를 제거
    pub(crate) fn remove_peer_from_trusted_set(&mut self, peer_id: PeerId) {
        let Entry::Occupied(mut entry) = self.peers.entry(peer_id) else { return };
        if !entry.get().is_trusted() {
            return;
        }
        self.trusted_peer_ids.remove(&peer_id);
    }
}
```

#### ④ peer 연결 상태 및 동작
```Rust
impl PeersManager {
    // 새로운 _인바운드_ 세션이 피어와 성공적으로 설정되었을 때 호출됨
    pub(crate) fn on_incoming_session_established(&mut self, peer_id: PeerId, addr: SocketAddr) {
        self.connection_info.decr_pending_in();
        // 세션 설정 로직 처리
    }
}
```
```Rust
impl PeersManager {

    // 피어와의 활성 세션이 오류로 인해 강제로 종료되었을 때 호출됨
    pub(crate) fn on_active_session_dropped(
        &mut self,
        remote_addr: &SocketAddr,
        peer_id: &PeerId,
        err: &EthStreamError,
    ) {
        self.on_connection_failure(remote_addr, peer_id, err, ReputationChangeKind::Dropped);
    }
}
```
```Rust
impl PeersManager {
    // 아웃바운드 세션이 인증 또는 핸드셰이크 과정에서 종료되었을 때 호출됨
    pub(crate) fn on_outgoing_pending_session_dropped(
        &mut self,
        remote_addr: &SocketAddr,
        peer_id: &PeerId,
        err: &PendingSessionHandshakeError,
    ) {
        self.on_connection_failure(remote_addr, peer_id, err, ReputationChangeKind::FailedToConnect);
    }
}
```
```Rust
impl PeersManager {
    // 피어를 설정된 시간만큼 차단
    fn ban_peer(&mut self, peer_id: PeerId) {
        self.ban_list.ban_peer_until(peer_id, std::time::Instant::now() + self.ban_duration);
        self.queued_actions.push_back(PeerAction::BanPeer { peer_id });
    }
}
```

#### ⑤ peer reputation 및 backoff
```Rust
impl PeersManager {
    // 주어진 피어에 대한 평판 변화를 적용
    pub(crate) fn apply_reputation_change(&mut self, peer_id: &PeerId, rep: ReputationChangeKind) {
        if let Some(peer) = self.peers.get_mut(peer_id) {
            // 평판 변화 적용 로직 처리
        }
    }
}
```
```Rust
impl PeersManager {
    // 피어의 평판을 반환
    pub(crate) fn get_reputation(&self, peer_id: &PeerId) -> Option<i32> {
        self.peers.get(peer_id).map(|peer| peer.reputation)
    }
}
```
```Rust
impl PeersManager {
    // 피어를 백오프(backoff) 상태로 설정
    fn backoff_peer_until(&mut self, peer_id: PeerId, until: std::time::Instant) {
        if let Some(peer) = self.peers.get_mut(&peer_id) {
            peer.backed_off = true;
            self.backed_off_peers.insert(peer_id, until);
        }
    }
}
```
```Rust
impl PeersManager {
    /// 모든 연결된 피어의 평판을 업데이트하는 틱(tick) 함수
    fn tick(&mut self) {
        let now = Instant::now();
        // 틱 함수 로직 처리
    }
}
```
#### ⑥ `PeersManager` 작업 큐 및 이벤트 처리
```Rust
impl PeersManager {
    // 상태를 갱신
    pub fn poll(&mut self, cx: &mut Context<'_>) -> Poll<PeerAction> {
        loop {
            // 작업 큐에서 작업을 처리
        }
    }
}
```
```Rust
impl PeersManager {
    // 새로운 아웃바운드 연결을 위한 슬롯이 있을 경우 이를 큐에 추가
    fn fill_outbound_slots(&mut self) {
        self.tick();
        // 아웃바운드 슬롯을 채우는 로직 처리
    }
}
```
### 2. `ConnectionInfo` 구조체 및 구현 
- 네트워크 연결 상태 추적
```Rust
pub struct ConnectionInfo {
    // 현재 활성화된 아웃바운드 연결 수 
    num_outbound: usize,
    // 대기 중인 아웃바운드 연결 수
    num_pending_out: usize,
    // 현재 활성화된 인바운드 연결 수
    num_inbound: usize,
    // 대기 중인 인바운드 연결 수
    num_pending_in: usize,
    // 연결 제한 정의하는 설정 값
    config: ConnectionsConfig,
}
```

#### ① 생성 및 초기화 
``` Rust
impl ConnectionInfo {
    const fn new(config: ConnectionsConfig) -> Self {
        Self { config, num_outbound: 0, num_pending_out: 0, num_inbound: 0, num_pending_in: 0 }
    }
}
```
#### ② 상태 확인
``` Rust
impl ConnectionInfo {
    const fn has_out_capacity(&self) -> bool {
        self.num_pending_out < self.config.max_concurrent_outbound_dials &&
            self.num_outbound < self.config.max_outbound
    }
}
```
- `has_out_capacity` : 아웃바운드 연결 용량 확인
``` Rust
impl ConnectionInfo {
    const fn has_in_capacity(&self) -> bool {
        self.num_inbound < self.config.max_inbound
    }
}
```
- `has_in_capacity` : 인바운드 연결 용량 확인


#### ③ 카운터 증가/감소
- `아웃바운드` 관련

``` Rust
impl ConnectionInfo {
    // 활성화된 아웃바운드 연결 수 1 감소
    fn decr_out(&mut self) {
    self.num_outbound -= 1;
    }
}
```
``` Rust
impl ConnectionInfo {
    // 활성화된 아웃바운드 연결 수 1 증가
    fn inc_out(&mut self) {
        self.num_outbound += 1;
    }
}
```
``` Rust
impl ConnectionInfo {
    // 대기 중인 아웃바운드 연결 수 1 감소
    fn decr_pending_out(&mut self) {
        self.num_pending_out -= 1;
    }
}
``` 
``` Rust
impl ConnectionInfo {
    // 대기 중인 인바운드 연결 수 1 증가
    fn inc_pending_out(&mut self) {
        self.num_pending_out += 1;
    }
}
```

- `인바운드` 관련
``` Rust
impl ConnectionInfo {
    // 활성화된 인바운드 연결 수 1 감소
   fn decr_in(&mut self) {
    self.num_inbound -= 1;
}
}
```
``` Rust
impl ConnectionInfo {
    // 활성화된 인바운드 연결 수 1 증가
    fn inc_in(&mut self) {
    self.num_inbound += 1;
    }
}
```
``` Rust
impl ConnectionInfo {
    // 대기 중인 인바운드 연결 수 1 감소
    fn decr_pending_in(&mut self) {
    self.num_pending_in -= 1;
    }
}
```
``` Rust
impl ConnectionInfo {
    // 대기 중인 인바운드 연결 수 1 증가
    fn inc_pending_in(&mut self) {
    self.num_pending_in += 1;
    }
}
```
#### ④ 상태 변화 
``` Rust
impl ConnectionInfo {
    fn decr_state(&mut self, state: PeerConnectionState) {
    match state {
        PeerConnectionState::Idle => {}
        PeerConnectionState::DisconnectingIn | PeerConnectionState::In => self.decr_in(),
        PeerConnectionState::DisconnectingOut | PeerConnectionState::Out => self.decr_out(),
        PeerConnectionState::PendingOut => self.decr_pending_out(),
    }
    }
}
```
-   `decr_state` 
    - 피어의 연결 상태를 확인
    - 상태에 따라 인바운드 / 아웃바운드 연결 카운터 감소

### 3. `PeerAction` 열거형 
- peer와 상호작용할 때 동작 정의해둠
```Rust
pub enum PeerAction {
    // 새로운 peer와의 연결 시작
    Connect {
        peer_id: PeerId,
        remote_addr: SocketAddr,
    },
    // 기존 연결 해제 
    Disconnect {
        peer_id: PeerId,
        reason: Option<DisconnectReason>,
    },
    // peer가 `BanList`에 있을 때 수신 연결 해제 
    DisconnectBannedIncoming {
        peer_id: PeerId,
    },
    // 신뢰할 수 없는 수신 연결 해제 
    DisconnectUntrustedIncoming {

        peer_id: PeerId,
    },
    // discovery에서 특정 peerId 차단
    DiscoveryBanPeerId {

        peer_id: PeerId,

        ip_addr: IpAddr,
    },
    // discovery에서 특정 IP 차단
    DiscoveryBanIp {
        ip_addr: IpAddr,
    },
    // peer 차단
    BanPeer {
        peer_id: PeerId,
    },
    // peer 차단 해제
    UnBanPeer {
        peer_id: PeerId,
    },
    // 네트워크에 새로운 peer 추가되었을 때 이벤트 발생 
    PeerAdded(PeerId),
    // 네트워크에서 peer 제거되었을 때 이벤트 발생 
    PeerRemoved(PeerId),
}
```

## peer
- 피어 식별 / 신뢰 피어 목록 관리 
### lib.rs
- 이더리움 네트워크 엔티티 : `NodeRecord`, `PeerId`, `Enr` 관리하는 유틸리티 모듈
- 네트워크 상에서 peer 식별 , 노드 레코드 처리 
[File : crates/net/peers/src/lib.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/peers/src/lib.rs)  
#### ① `peerId` 관리 
```Rust
#[cfg(feature = "secp256k1")]
const SECP256K1_TAG_PUBKEY_UNCOMPRESSED: u8 = 4;

/// secp256k1 공개키를 PeerId로 변환하는 함수
#[cfg(feature = "secp256k1")]
#[inline]
pub fn pk2id(pk: &secp256k1::PublicKey) -> PeerId {
    PeerId::from_slice(&pk.serialize_uncompressed()[1..])
}

/// PeerId를 secp256k1 공개키로 변환하는 함수
#[cfg(feature = "secp256k1")]
#[inline]
pub fn id2pk(id: PeerId) -> Result<secp256k1::PublicKey, secp256k1::Error> {
    let mut s = [0u8; secp256k1::constants::UNCOMPRESSED_PUBLIC_KEY_SIZE];
    s[0] = SECP256K1_TAG_PUBKEY_UNCOMPRESSED;
    s[1..].copy_from_slice(id.as_slice());
    secp256k1::PublicKey::from_slice(&s)
}
```
#### ② `NodeRecord` 관리 
```Rust
impl NodeRecord {
    pub fn from_str(s: &str) -> Result<NodeRecord, NodeRecordParseError> {
        ...
    } 
}  
```
- `NodeRecord` : peer의 IP 주소, 포트 및 공개키 정보 포함하고 있음
- `from_str` : 주어진 peerId 통해 노드레코드를 반환

#### ③ AnyNode 관리 
- `peerId`, `NodeRecord`, `Enr` 중 어떤 타입도 처리 가능 
``` Rust
impl AnyNode {
    // 노드의 PeerId 반환
    pub fn peer_id(&self) -> PeerId {
        match self {
            Self::NodeRecord(record) => record.id,
            #[cfg(feature = "secp256k1")]
            Self::Enr(enr) => pk2id(&enr.public_key()),
            Self::PeerId(peer_id) => *peer_id,
        }
    }

    // 노드 레코드 반환
    pub fn node_record(&self) -> Option<NodeRecord> {
        match self {
            Self::NodeRecord(record) => Some(*record),
            #[cfg(feature = "secp256k1")]
            Self::Enr(enr) => {
                let node_record = NodeRecord {
                    address: enr.ip4().map(std::net::IpAddr::from).or_else(|| enr.ip6().map(std::net::IpAddr::from))?,
                    tcp_port: enr.tcp4().or_else(|| enr.tcp6())?,
                    udp_port: enr.udp4().or_else(|| enr.udp6())?,
                    id: pk2id(&enr.public_key()),
                }.into_ipv4_mapped();
                Some(node_record)
            }
            _ => None,
        }
    }
}
```
#### ④ `TrustedPeer` 관리
```Rust
impl TrustedPeer {
    /// 도메인을 IP 주소로 해결하는 메서드
    pub fn resolve(&self) -> Result<NodeRecord, ResolveError> {
        ...
    }
}
```
- `TrustedPeer` : 신뢰할 수 있는 peer 관리
- 도메인 이름을 IP 주소로 해결하는 것이 핵심

#### ⑤ `WithPeerId` 기능
- peerId와 데이터 함께 관리하고 변환
```Rust
impl<T> WithPeerId<T> {
    /// 새로운 PeerId와 데이터를 함께 래핑
    pub const fn new(peer: PeerId, value: T) -> Self {
        Self(peer, value)
    }

    /// PeerId 반환
    pub const fn peer_id(&self) -> PeerId {
        self.0
    }

    /// 데이터를 반환
    pub const fn data(&self) -> &T {
        &self.1
    }

    /// 데이터를 소유권과 함께 반환
    pub fn into_data(self) -> T {
        self.1
    }

    /// 데이터 변환 메서드
    pub fn transform<F: From<T>>(self) -> WithPeerId<F> {
        WithPeerId(self.0, self.1.into())
    }

    /// PeerId와 데이터를 분리하여 반환
    pub fn split(self) -> (PeerId, T) {
        (self.0, self.1)
    }
    
    /// 데이터를 매핑하여 변환
    pub fn map<U, F: FnOnce(T) -> U>(self, op: F) -> WithPeerId<U> {
        WithPeerId(self.0, op(self.1))
    }
}
```


위에서 계속 언급되는 `trusted_node`와 `node_record`는 
[File : crates/net/peers/src](https://github.com/paradigmxyz/reth/blob/main/crates/net/peers/src) 에 역시 작성되어있다.

### trusted_peer.rs
- 네트워크 노드의 신뢰할 수 있는 peer
- 도메인 기반의 peer 정보 관리 
- IP / 도메인 해석 기능 : peer의 도메인을 IP로 변환 ... 이를 통해 `NodeRecord` 생성 

#### ① 구조체 
```Rust
pub struct TrustedPeer {
    pub host: Host,        // 피어의 도메인 또는 IP 주소
    pub tcp_port: u16,     // TCP 포트
    pub udp_port: u16,     // UDP 포트
    pub id: PeerId,        // 피어의 공개 키를 나타내는 ID
}
```
#### ② `TrustedPeer` 생성 및 변환
```Rust
impl TrustedPeer {
    // peerId와 호스트 정보 기반으로 새로운 TrustedPeer 생성
    pub const fn new(host: Host, port: u16, id: PeerId) -> Self {
        Self { host, tcp_port: port, udp_port: port, id }
    }
}
```
```Rust
impl TrustedPeer {
    // 비밀 키로부터 peer의 공개 키를 유도하여 TrustedPeer를 생성
    pub fn from_secret_key(host: Host, port: u16, sk: &secp256k1::SecretKey) -> Self {
    let pk = secp256k1::PublicKey::from_secret_key(secp256k1::SECP256K1, sk);
    let id = PeerId::from_slice(&pk.serialize_uncompressed()[1..]);
    Self::new(host, port, id)
    }
}
```
#### ③ IP 주소 변환 및 도메인 해석
```Rust
impl TrustedPeer {
    const fn to_node_record(&self, ip: IpAddr) -> NodeRecord {
    NodeRecord { address: ip, id: self.id, tcp_port: self.tcp_port, udp_port: self.udp_port }
    }
}
```
- `to_node_record`
    - `NodeRecord`로 변환
    - IP 주소, 포트, peerID 포함해 더 완전한 peer 정보를 나타냄.
```Rust
impl TrustedPeer {
    fn try_node_record(&self) -> Result<NodeRecord, &str> {
    match &self.host {
        // 호스트가 IP 주소일 때 : IP 주소로 NodeRecord 생성 
        Host::Ipv4(ip) => Ok(self.to_node_record((*ip).into())),
        Host::Ipv6(ip) => Ok(self.to_node_record((*ip).into())),
        // 호스트가 도메인 일 때 : 도메인 문자열 반환
        Host::Domain(domain) => Err(domain),
    }
    }
}
```
- 도메인을 **동기적**으로 IP 주소로 변환하여  `NodeRecord`로 반환 
```Rust
impl TrustedPeer {
    pub fn resolve_blocking(&self) -> Result<NodeRecord, Error> {
    let domain = match self.try_node_record() {
        Ok(record) => return Ok(record),
        Err(domain) => domain,
    };
    let mut ips = std::net::ToSocketAddrs::to_socket_addrs(&(domain, 0))?;
    let ip = ips.next().ok_or_else(|| Error::new(std::io::ErrorKind::AddrNotAvailable, "No IP found"))?;
    Ok(self.to_node_record(ip.ip()))
    }
}
```
- 도메인을 **비동기적**으로 IP 주소로 변환하여  `NodeRecord`로 반환 
```Rust
impl TrustedPeer {
    pub async fn resolve(&self) -> Result<NodeRecord, Error> {
    let domain = match self.try_node_record() {
        Ok(record) => return Ok(record),
        Err(domain) => domain,
    };
    let mut ips = tokio::net::lookup_host(format!("{domain}:0")).await?;
    let ip = ips.next().ok_or_else(|| Error::new(std::io::ErrorKind::AddrNotAvailable, "No IP found"))?;
    Ok(self.to_node_record(ip.ip()))
    }
}
```
#### ④ 트레이트 구현 
- `fmt::Display` : `TrustedPeer`를 문자열로 출력할 수 있게 `enode://` 형식으로 포맷
- `FromStr` : 문자열을 파싱하여 `TrustedPeer` 객체로 변환

### node_record.rs
- peer의 정보를 표현하고 관리
- 노드의 IP 주소와 포트, 공개 키 포함 

#### ① 구조체 
```Rust
pub struct NodeRecord {
    pub address: IpAddr,   // 노드의 IP 주소 (IPv4 또는 IPv6)
    pub udp_port: u16,     // UDP 포트 (발견 서비스에 사용)
    pub tcp_port: u16,     // TCP 포트 (연결에 사용)
    pub id: PeerId,        // 피어의 공개 키 (PeerId 타입)
}
```
#### ② 생성 및 초기화 
```Rust
// `SocketAddr`와 `PeerId`를 이용해 `NodeRecord`를 생성
pub const fn new(addr: SocketAddr, id: PeerId) -> Self
// `IP 주소`, `TCP 포트`, `UDP 포트` 명시적으로 지정해 NodeRecord 생성
pub fn new_with_ports(ip_addr: IpAddr, tcp_port: u16, udp_port: Option<u16>, id: PeerId) -> Self
// 비밀 키로부터 `NodeRecord` 생성
pub fn from_secret_key(addr: SocketAddr, sk: &secp256k1::SecretKey) -> Self
```
#### ③ IP 주소 및 소켓 주소 관련 기능
```Rust
impl NodeRecord {
    pub fn convert_ipv4_mapped(&mut self) -> bool {
        if let IpAddr::V6(v6) = self.address {
            if let Some(v4) = v6.to_ipv4_mapped() {
                self.address = v4.into();
                return true
            }
        }
        false
    }
}

impl NodeRecord {
    pub fn into_ipv4_mapped(mut self) -> Self {
        self.convert_ipv4_mapped();
        self
    }
}
```
- `convert_ipv4_mapped`: IPv6 주소가 IPv4로 매핑된 경우, 이를 IPv4로 변환
- `into_ipv4_mapped` : `convert_ipv4_mapped`와 동일하지만, 객체를 소비하는 방식으로 변환
```Rust
impl NodeRecord {
    pub const fn tcp_addr(&self) -> SocketAddr {
        SocketAddr::new(self.address, self.tcp_port)
    }

    #[must_use]
    pub const fn udp_addr(&self) -> SocketAddr {
        SocketAddr::new(self.address, self.udp_port)
    }
}
```
- `tcp_addr` : peer의 TCP 소켓 주소 반환
- `udp_addr` : peer의 UDP 소켓 주소 반환

 #### ④ TCP / UDP 포트 설정
 ```Rust
impl NodeRecord {
    // TCP 포트 설정
    pub const fn with_tcp_port(mut self, port: u16) -> Self
    // UDP 포트 설정
    pub const fn with_udp_port(mut self, port: u16) -> Self
}
```
#### ④ 기타 
- `fmt::Display` : `NodeRecord` 객체를 `enode://` 형식으로 출력
- `FromStr` : `enode://` 형식의 문자열을 `NodeRecord` 객체로 변환
- `TryFrom<Enr> (secp256k1 feature 필요)`: `Enr` 타입을 `NodeRecord`로 변환

## bootnodes
[File : crates/net/peers/src/bootnodes](https://github.com/paradigmxyz/reth/tree/main/crates/net/peers/src/bootnodes) 

- 부트노드는 네트워크에 처음 연결할 때 사용되는 노드
- 부트노드를 정의하는 내용으로 구성되어 있다.