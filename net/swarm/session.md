#### **[Session Manager]**
- [mod.rs](#modrs)
- [active.rs](#activers)
- [conn.rs](#connrs)
- [handle.rs](#handlers)
- [counter.rs](#counterrs)

## mod.rs
- `session` 모듈의 진입점

[File : crates/net/network/src/session/mod.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/mod.rs) 
### 1. 세션 관리 기능
```Rust
impl SessionManager {
    // 세션 관리자 초기화
    pub fn new( secret_key: SecretKey,
        config: SessionsConfig,
        executor: Box<dyn TaskSpawner>,
        status: Status,
        hello_message: HelloMessageWithProtocols,
        fork_filter: ForkFilter,
        extra_protocols: RlpxSubProtocols,) -> Self {
    }

    // 새로운 세션 처리
    pub(crate) fn on_incoming(&mut self, stream: TcpStream, remote_addr: SocketAddr) -> Result<SessionId, ExceedsSessionLimit> {

    }
    // 원격 피어로 세션 설정
    pub fn dial_outbound(&mut self, remote_addr: SocketAddr, remote_peer_id: PeerId) {
    }
    
    // 특정 피어와 세션 종료 
    pub fn disconnect(&self, node: PeerId, reason: Option<DisconnectReason>) {

    }
    // 모든 활성 세션 종료 
    pub fn disconnect_all(&self, reason: Option<DisconnectReason>) {
    }

    // 모든 대기 중인 세션 종료
    pub fn disconnect_all_pending(&mut self) {
    }
}
```
### 2. 세션 이벤트 처리 기능
```Rust
impl SessionManager {
  // 세션 이벤트 처리
  pub(crate) fn poll(&mut self, cx: &mut Context<'_>) -> Poll<SessionEvent> {
  }
  // 활성 세션 폴링 처리
  fn poll_active_session_rx(&mut self, cx: &mut Context<'_>) -> Poll<ActiveSessionMessage> {
  }
}
```
### 3. 세션 인증 기능
```Rust
// 세션 인증 설정
async fn pending_session_with_timeout<F>(
    timeout: Duration,
    session_id: SessionId,
    remote_addr: SocketAddr,
    direction: Direction,
    events: mpsc::Sender<PendingSessionEvent>,
    f: F,
) where F: Future<Output = ()> {
}

// 외부에서 들어오는 세션 인증
async fn start_pending_incoming_session( /* params */ ) {
}

// 외부 피어로 세션 인증
async fn start_pending_outbound_session( /* params */ ) {
}

// 세션 인증 및 스트림 처리
async fn authenticate_stream( /* params */ ) -> PendingSessionEvent {
}
```
### 4. 세션 상태 및 이벤트 처리 
```Rust
impl SessionManager {
  // 대기 중인 세션 제거 
  fn remove_pending_session(&mut self, id: &SessionId) -> Option<PendingSessionHandle> {
  }
  // 활성 세션 제거
  fn remove_active_session(&mut self, id: &PeerId) -> Option<ActiveSessionHandle> {
  }
  // peer에게 메시지 전송
  pub fn send_message(&mut self, peer_id: &PeerId, msg: PeerMessage) {
  }
  // 상태 업데이트 처리
  pub(crate) fn on_status_update(&mut self, head: Head) -> Option<ForkTransition> {
  }
}
```
## active.rs
- `ActiveSession` : 연결된 세션 관리하는 역할
- 요청 ID 추적, 연결(conn), 수신된 명령 처리, 수신된 내부 요청 처리, 처리 중인 요청 관리, 대기 중인 메시지 전송 관리


[File : crates/net/network/src/session/active.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/active.rs) 
### 1. `ActiveSession` 구조체
```Rust
pub(crate) struct ActiveSession {
    pub(crate) next_id: u64,
    pub(crate) conn: EthRlpxConnection,
    pub(crate) remote_peer_id: PeerId,
    pub(crate) remote_addr: SocketAddr,
    pub(crate) remote_capabilities: Arc<Capabilities>,
    pub(crate) session_id: SessionId,
    pub(crate) commands_rx: ReceiverStream<SessionCommand>,
    pub(crate) to_session_manager: MeteredPollSender<ActiveSessionMessage>,
    pub(crate) pending_message_to_session: Option<ActiveSessionMessage>,
    pub(crate) internal_request_tx: Fuse<ReceiverStream<PeerRequest>>,
    pub(crate) inflight_requests: FxHashMap<u64, InflightRequest>,
    pub(crate) received_requests_from_remote: Vec<ReceivedRequest>,
    pub(crate) queued_outgoing: VecDeque<OutgoingMessage>,
    pub(crate) internal_request_timeout: Arc<AtomicU64>,
    pub(crate) internal_request_timeout_interval: Interval,
    pub(crate) protocol_breach_request_timeout: Duration,
    pub(crate) terminate_message: Option<(PollSender<ActiveSessionMessage>, ActiveSessionMessage)>,
}
```
### 2. 세션 상태 확인 및 관리 
```Rust
// 현재 세션이 연결 해제 중인지 확인
fn is_disconnecting(&self) -> bool {
    self.conn.inner().is_disconnecting()
}
// 다음 요청 ID 생성
fn next_id(&mut self) -> u64 {
    let id = self.next_id;
    self.next_id += 1;
    id
}
// 내부 버퍼 최적화
pub fn shrink_to_fit(&mut self) {
    self.received_requests_from_remote.shrink_to_fit();
    self.queued_outgoing.shrink_to_fit();
}
```
### 3. 메시지 처리 관련
```Rust
fn on_incoming_message(&mut self, msg: EthMessage) -> OnIncomingMessageOutcome {
}
fn on_internal_peer_message(&mut self, msg: PeerMessage) {
}
```
- `on_incoming_message` : 외부에서 받은 메시지 유형에 따라 처리
- `on_internal_peer_message` : 내부에서 들어온 메시지 처리

### 4. 요청 및 응답 관련
```Rust
fn on_internal_peer_request(&mut self, request: PeerRequest, deadline: Instant) {
    let request_id = self.next_id();
    let msg = request.create_request_message(request_id);
    self.queued_outgoing.push_back(msg.into());
    let req = InflightRequest {
        request: RequestState::Waiting(request),
        timestamp: Instant::now(),
        deadline,
    };
    self.inflight_requests.insert(request_id, req);
}
```
- `on_internal_peer_request` : 내부에서 발생한 요청 원격 peer에게 전달하고 요청 대기열에 추가
```Rust
fn handle_outgoing_response(&mut self, id: u64, resp: PeerResponseResult) {
    match resp.try_into_message(id) {
        Ok(msg) => {
            self.queued_outgoing.push_back(msg.into());
        }
        Err(err) => {
            debug!(target: "net", %err, "Failed to respond to received request");
        }
    }
}
```
- `handle_outgoing_resoonse` : 수신된 요청에 대한 응답 큐에 추가해 원격 peer로 전송할 수 있게 함
```Rust
// 요청 만료 시점 계산
fn request_deadline(&self) -> Instant {
    Instant::now() + Duration::from_millis(self.internal_request_timeout.load(Ordering::Relaxed))
}
// 타임아웃 발생 요청 확인 후 프로토콜 위반 시 세션 종료
fn check_timed_out_requests(&mut self, now: Instant) -> bool {
    for (id, req) in &mut self.inflight_requests {
        if req.is_timed_out(now) {
            if req.is_waiting() {
                debug!(target: "net::session", ?id, remote_peer_id=?self.remote_peer_id, "timed out outgoing request");
                req.timeout();
            } else if now - req.timestamp > self.protocol_breach_request_timeout {
                return true
            }
        }
    }
    false
}
```
```Rust
// 요청의 전송 및 수신 시간을 기준으로 타임아웃을 갱신
fn update_request_timeout(&mut self, sent: Instant, received: Instant) {
}
```
### 5. 세션 종료 및 연결 해제 
- `emit_disconnect()`
- `close_on_error`
- `try_disconnect`
- `poll_disconnect`

### 6. 메시지 전송 관련
```Rust
fn try_emit_broadcast(&self, message: PeerMessage) -> Result<(), ActiveSessionMessage> {
    let Some(sender) = self.to_session_manager.inner().get_ref() else { return Ok(()) };
    match sender.try_send(ActiveSessionMessage::ValidMessage { peer_id: self.remote_peer_id, message }) {
        Ok(_) => Ok(()),
        Err(err) => {
            trace!(target: "net", %err, "no capacity for incoming broadcast");
            match err {
              // 큐가 가득 차면 대기열에 추가
                TrySendError::Full(msg) => Err(msg),
                TrySendError::Closed(_) => Ok(()),
            }
        }
    }
}
```
- `try_emit_broadcast` : 세션 관리자에게 브로드캐스트 메시지 전송
```Rust
fn try_emit_request(&self, message: PeerMessage) -> Result<(), ActiveSessionMessage> {
    ...
}
fn poll_terminate_message(&mut self, cx: &mut Context<'_>) -> Option<Poll<()>> {
  
}
```
- `try_emit_request` : 세션 관리자에게 요청 메시지 전송
- `poll_terminate_message` : 종료 메시지 전송 및 가득차있는 큐 처리 

## conn.rs
- peer 간 세션 연결 타입 정의 및 관리
- `EthRlpxConnection` 통해 메시지 전송 및 수신

[File : crates/net/network/src/session/conn.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/conn.rs) 
### 1. `ETH 프로토콜` 관련
```Rust
pub(crate) const fn version(&self) -> EthVersion {
    match self {
        Self::EthOnly(conn) => conn.version(),
        Self::Satellite(conn) => conn.primary().version(),
    }
}
```
- `version` : 연결된 peer와 협상된 ETH 버전을 반환
- `EthOnly` 와 `Satellite` 모두 적용
```Rust
pub fn start_send_broadcast(
    &mut self,
    item: EthBroadcastMessage,
) -> Result<(), EthStreamError> {
    match self {
        Self::EthOnly(conn) => conn.start_send_broadcast(item),
        Self::Satellite(conn) => conn.primary_mut().start_send_broadcast(item),
    }
}
```
- `start_send_broadcast` : `EthBroadcastMessage`를 peer에게 전송
- `EthOnly` or `Satellite` 에 따라 다르게 처리

### 2. 스트림 관련
```Rust
// 스트림에서 EthMessage 비동기적으로 가져옴 
fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
    delegate_call!(self.poll_next(cx))
}

// peer에게 메시지 전송할 준비 되었는지 확인
fn poll_ready(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
    delegate_call!(self.poll_ready(cx))
}

// EthMessage를 peer에게 전송
fn start_send(self: Pin<&mut Self>, item: EthMessage) -> Result<(), Self::Error> {
    delegate_call!(self.start_send(item))
}

// 전송중인 데이터 모두 전송 후 스트림 비우기
fn poll_flush(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
    delegate_call!(self.poll_flush(cx))
}

// 전송 중인 모든 메시지 처리 후 스트림 종료 
fn poll_close(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
    delegate_call!(self.poll_close(cx))
}
```
> 그 외에도 내부 스트림에 접근하는 inner 메서드들과 타입 변환 관련 메서드가 있다.

## handle.rs
- 세션 관리에 관한 다양한 핸들러와 메시지 정의
- 세션 생성 / 활성화 / 종료 / 오류 처리될 때 발생하는 이벤트와 명령들을 처리

[File : crates/net/network/src/session/handle.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/handle.rs) 
### 1. 대기 중인 세션 관리
```Rust
pub struct PendingSessionHandle {
    pub(crate) disconnect_tx: Option<oneshot::Sender<()>>,
    pub(crate) direction: Direction,
}
```
- `PendingSessionHandle` : 아직 인증되지 않은 세션 관리 핸들러
```Rust
impl PendingSessionHandle {
  // 세션 해제 
  pub fn disconnect(&mut self) {
        if let Some(tx) = self.disconnect_tx.take() {
            let _ = tx.send(());
        }
  }
  // 세션 들어오고 나가는 방향 반환
  pub const fn direction(&self) -> Direction {
        self.direction
  }
}
```
### 2. 활성화된 세션 처리
```Rust
pub struct ActiveSessionHandle {
    pub(crate) direction: Direction,
    pub(crate) session_id: SessionId,
    pub(crate) version: EthVersion,
    pub(crate) remote_id: PeerId,
    pub(crate) established: Instant,
    pub(crate) capabilities: Arc<Capabilities>,
    pub(crate) commands_to_session: mpsc::Sender<SessionCommand>,
    pub(crate) client_version: Arc<str>,
    pub(crate) remote_addr: SocketAddr,
    pub(crate) local_addr: Option<SocketAddr>,
    pub(crate) status: Arc<Status>,
}
```
- `ActiveSessionHandle` : 인증된 후 활성화된 세션 관리 핸들러 
- 체인 동기화, 블록 전파, 트랜잭션 교환 
```Rust
impl ActiveSessionHandle {
    pub fn disconnect(&self, reason: Option<DisconnectReason>) {
        let _ = self.commands_to_session.clone().try_send(SessionCommand::Disconnect { reason });
    }
    ...

    pub(crate) fn peer_info(&self, record: &NodeRecord, kind: PeerKind) -> PeerInfo {
        PeerInfo {
            remote_id: self.remote_id,
            direction: self.direction,
            enode: record.to_string(),
            enr: None,
            remote_addr: self.remote_addr,
            local_addr: self.local_addr,
            capabilities: self.capabilities.clone(),
            client_version: self.client_version.clone(),
            eth_version: self.version,
            status: self.status.clone(),
            session_established: self.established,
            kind,
        }
    }
}
```
- `disconnect` : 특정 이유와 함께 활성 세션 해제
- `peer_info` : 세션 정보 바탕으로 `PeerInfo` 객체 생성 후 반환

### 3. 세션 이벤트 처리
- `PendingSessionEvent` : 세션 인증 or 연결 과정에서 발생 이벤트 열거형
```Rust
pub enum PendingSessionEvent {
    Established {
        session_id: SessionId,
        remote_addr: SocketAddr,
        local_addr: Option<SocketAddr>,
        peer_id: PeerId,
        capabilities: Arc<Capabilities>,
        status: Arc<Status>,
        conn: EthRlpxConnection,
        direction: Direction,
        client_id: String,
    }
}
```
- `Established`: 세션이 성공적으로 설정됐을 때 
```Rust
pub enum PendingSessionEvent {
    Disconnected {
        remote_addr: SocketAddr,
        session_id: SessionId,
        direction: Direction,
        error: Option<PendingSessionHandshakeError>,
    }
}
```
- `Disconnected` : 세션이 실패하거나 연결이 끊겼을 때 발생
```Rust
pub enum PendingSessionEvent {
    EciesAuthError {
        remote_addr: SocketAddr,
        session_id: SessionId,
        error: ECIESError,
        direction: Direction,
    }
}
```
- `EciesAuthError` : ECIES 인증 중에 발생한 오류 처리 이벤트

### 4. 세션 명령 처리 
```Rust
pub enum SessionCommand {
    // 세션 연결 해제
    Disconnect {
        reason: Option<DisconnectReason>,
    },
    // peer에게 메시지 전송 
    Message(PeerMessage),
}
```
- `SessionCommand` : 활성 세션에 전달되는 명령어 열거형

### 5. 활성 세션 메시지 
- `ActiveSessionMessage` : 활성 세션에서 발생하는 상태나 오류 메시지 열거형
```Rust
pub enum ActiveSessionMessage {
    Disconnected {
        peer_id: PeerId,
        remote_addr: SocketAddr,
    },
    ClosedOnConnectionError {
        peer_id: PeerId,
        remote_addr: SocketAddr,
        error: EthStreamError,
    },
    ValidMessage {
        peer_id: PeerId,
        message: PeerMessage,
    },
    InvalidMessage {
        peer_id: PeerId,
        capabilities: Arc<Capabilities>,
        message: CapabilityMessage,
    },
    BadMessage {
        peer_id: PeerId,
    },
    ProtocolBreach {
        peer_id: PeerId,
    },
}
```

## counter.rs
- `SessionCounter` : 들어오고 나가는 세션 수 추적
- 세션 상태 - 활성 / 대기 관리, 세션 제한 초과 관리 

[File : crates/net/network/src/session/counter.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/counter.rs) 
### 1. `SessionCounter` 구조체
```Rust
pub struct SessionCounter {
    limits: SessionLimits,
    pending_inbound: u32,
    pending_outbound: u32,
    active_inbound: u32,
    active_outbound: u32,
}
```

### 2. `SessionCounter` 초기화 및 상태 확인
```Rust
impl SessionCounter {
    pub(crate) const fn new(limits: SessionLimits) -> Self {
        Self {
            limits,
            pending_inbound: 0,
            pending_outbound: 0,
            active_inbound: 0,
            active_outbound: 0,
        }
    }
        pub(crate) const fn ensure_pending_outbound(&self) -> Result<(), ExceedsSessionLimit> {
        Self::ensure(self.pending_outbound, self.limits.max_pending_outbound)
    }

    pub(crate) const fn ensure_pending_inbound(&self) -> Result<(), ExceedsSessionLimit> {
        Self::ensure(self.pending_inbound, self.limits.max_pending_inbound)
    }
}
```
- `new` : 세션 제한 설정하고 카운터 초기화
- `ensure_pending_outbound` : 현재 대기 중인 나가는 세션이 최대치 초과하는지 확인
- `ensure_pending_inbound` : 현재 대기 중인 들어오는 세션이 최대치 초과하는지 확인

### 3. 세션 수 증가
```Rust
impl SessionCounter {
  pub(crate) fn inc_pending_inbound(&mut self) {
    self.pending_inbound += 1;
  }
  pub(crate) fn inc_pending_outbound(&mut self) {
    self.pending_outbound += 1;
  }
  pub(crate) fn inc_active(&mut self, direction: &Direction) {
    match direction {
        Direction::Outgoing(_) => {
            self.active_outbound += 1;
        }
        Direction::Incoming => {
            self.active_inbound += 1;
        }
    }
  }
}
- `inc_pending_inbound` : 대기 중인 들어오는 세션의 수 증가 
- `inc_pending_outbound` : 대기 중인 나가는 세션의 수 증가
- `inc_active` : 활성 세션의 수 증가 
### 4. 세션 수 감소
- 이것 역시 대기 중, 활성화 중인 세션의 수를 감소 시키는 로직으로 작성
- `dec_pending`
- `dec_active`
### 5. 세션 제한
```Rust
impl SessionCounter {
  const fn ensure(current: u32, limit: Option<u32>) -> Result<(), ExceedsSessionLimit> {
    if let Some(limit) = limit {
        if current >= limit {
            return Err(ExceedsSessionLimit(limit))
        }
    }
    Ok(())
  }
}
- `ensure` : 현재 세션 수가 설정된 제한 초과했는지 확인하는 함수