# eth-wire
- [protocol.rs](#protocolrs)
- [peer](#peer-1)
    - [lib.rs](#librs)
    - [node_record.rs](#node_recordrs)
    - [trusted_peer.rs](#trusted_peerrs)
## protocol.rs
[File : crates/net/network/src/protocol.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/protocol.rs) 
- `RLPx` 기반의 프로토콜을 처리하는 기능 정의
-  이더리움의 peer 간 연결을 확장해 프로토콜을 처리할 수 있도록 함

### 1. `ProtocolHandler` 관련
- `RLPx 프로토콜 핸들링`을 위한 메서드들
```Rust
pub trait ProtocolHandler: fmt::Debug + Send + Sync + 'static {
    type ConnectionHandler: ConnectionHandler;

    // 들어오는 연결에 대해 새로운 프로토콜 설정
    fn on_incoming(&self, socket_addr: SocketAddr) -> Option<Self::ConnectionHandler>;
    
    // 나가는 연결에 대해 새로운 프로토콜 설정
    fn on_outgoing(
        &self,
        socket_addr: SocketAddr,
        peer_id: PeerId,
    ) -> Option<Self::ConnectionHandler>;
}
```
### 2. `ConnectionHandler` 관련
- 프로토콜이 인증된 후 처리 방법을 정의하는 메서드들
```Rust
pub trait ConnectionHandler: Send + Sync + 'static {
    type Connection: Stream<Item = BytesMut> + Send + 'static;

    // peer와 협상할 프로토콜 반환
    fn protocol(&self) -> Protocol;
    // peer가 해당 프로토콜 지원하지 않는 경우
    fn on_unsupported_by_peer(
        self,
        supported: &SharedCapabilities,
        direction: Direction,
        peer_id: PeerId,
    ) -> OnNotSupported;
    // RLPx 연결된 후 호출되어 연결 스트림을 반환
    fn into_connection(
        self,
        direction: Direction,
        peer_id: PeerId,
        conn: ProtocolConnection,
    ) -> Self::Connection;
}
```
### 3. `RlpxSubProtocols` 관련
- `RLPx` 기반의 서브 프로토콜을 관리하는 메서드들
```Rust
pub struct RlpxSubProtocols {
    protocols: Vec<RlpxSubProtocol>,
}

impl RlpxSubProtocols {

    // 새로운 프로토콜 추가
    pub fn push(&mut self, protocol: impl IntoRlpxSubProtocol) {
        self.protocols.push(protocol.into_rlpx_sub_protocol());
    }
    // 들어오는 연결에 대해 추가 프로토콜 핸들러 반환
    pub(crate) fn on_incoming(&self, socket_addr: SocketAddr) -> RlpxSubProtocolHandlers {
        RlpxSubProtocolHandlers(
            self.protocols
                .iter()
                .filter_map(|protocol| protocol.0.on_incoming(socket_addr))
                .collect(),
        )
    }
    // 나가는 연결에 대해 추가 프로토콜 핸들러 반환
    pub(crate) fn on_outgoing(
        &self,
        socket_addr: SocketAddr,
        peer_id: PeerId,
    ) -> RlpxSubProtocolHandlers {
        RlpxSubProtocolHandlers(
            self.protocols
                .iter()
                .filter_map(|protocol| protocol.0.on_outgoing(socket_addr, peer_id))
                .collect(),
        )
    }
}
```

### 4. `DynProtocolHandler` 및 `DynConnectionHandler` 관련
- `프로토콜 핸들러`와 `연결 핸들러`의 동적 버전입니다.
```Rust
pub(crate) trait DynProtocolHandler: fmt::Debug + Send + Sync + 'static {
    // 들어오는 연결에 대해 핸들러 반환
    fn on_incoming(&self, socket_addr: SocketAddr) -> Option<Box<dyn DynConnectionHandler>>;

    // 나가는 연결에 대해 핸들러 반환
    fn on_outgoing(
        &self,
        socket_addr: SocketAddr,
        peer_id: PeerId,
    ) -> Option<Box<dyn DynConnectionHandler>>;
}
```
```Rust
pub(crate) trait DynConnectionHandler: Send + Sync + 'static {
    // 동적 연결 핸들러에서 협상할 프로토콜을 반환
    fn protocol(&self) -> Protocol;
    // 동적 연결 핸들러에서 연결 스트림을 반환
    fn into_connection(
        self: Box<Self>,
        direction: Direction,
        peer_id: PeerId,
        conn: ProtocolConnection,
    ) -> Pin<Box<dyn Stream<Item = BytesMut> + Send + 'static>>;
}
```

### 5. 기타

```Rust
// 프로토콜 핸들러를 RlpxSubProtocol로 변환하는 트레이트
fn into_rlpx_sub_protocol(self) -> RlpxSubProtocol;
```
```Rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
// 프로토콜이 지원되지 않을 때의 동작을 정의하는 열거형
pub enum OnNotSupported;
```
```Rust
// RlpxSubProtocolHandlers가 내부 벡터를 쉽게 참조할 수 있게 해줌 
impl Deref for RlpxSubProtocolHandlers;
impl DerefMut for RlpxSubProtocolHandlers;
```

## eth-wire

### lib.rs
[File : crates/net/eth-wire/src/lib.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/eth-wire/src/lib.rs) 

- eth-wire 모듈의 엔트리 포인트로 모든 기능들을 연결한다.
- 나머지 파일들에 다음 모듈들에 관한 코드가 작성되어 있다.
```Rust
// peer 간 기능 협상과 관련된 로직 처리
pub mod capability;
// peer 연결 해제 동작 처리 모듈
mod disconnect;
// 에러 정의 및 처리 모듈
pub mod errors;
// 이더리움 네트워크 상의 스트리밍 데이터 처리 모듈
mod ethstream;
// peer 연결 시 핸드쉐이크 과정 처리 모듈
mod hello;
// 멀티플렉싱 처리 통해 여러 프로토콜 동시 처리 모듈
pub mod multiplex;
// peer 간 데이터 전송 처리 모듈
mod p2pstream;
// 네트워크 상태 및 연결된 peer 상태 체크 모듈
mod pinger;
// 프로토콜 정의 및 관리 모듈
pub mod protocol;
```
- 재수출
```Rust
pub use crate::{
    disconnect::CanDisconnect,
    ethstream::{EthStream, UnauthedEthStream, MAX_MESSAGE_SIZE},
    hello::{HelloMessage, HelloMessageBuilder, HelloMessageWithProtocols},
    p2pstream::{
        DisconnectP2P, P2PMessage, P2PMessageID, P2PStream, UnauthedP2PStream,
        MAX_RESERVED_MESSAGE_ID,
    },
    Capability, ProtocolVersion,
};
```
### protocol.rs
- `RLPx` P2P 연결에서 사용할 하위 프로토콜(subprotocol)을 정의
- 프로토콜 관리 : `Capability`, 메시지 ID 멀티플렉싱 처리

```Rust
pub struct Protocol {
    // subprotocol 이름
    pub cap: Capability,
    // 메시지 ID 멀티플렉싱에 사용될 메시지 개수
    messages: u8,
}
```
#### ① 생성
```Rust
impl Protocol {
    // 새로운 protocol 객체 생성
    pub const fn new(cap: Capability, messages: u8) -> Self {
        Self { cap, messages }
    }

    // 특정 이더리움 버전 기반의 Protocol을 생성
    pub const fn eth(version: EthVersion) -> Self {
        let cap = Capability::eth(version);
        let messages = version.total_messages();
        Self::new(cap, messages)
    }

    // EthVersion::Eth66에 맞는 Protocol을 생성
    pub const fn eth_66() -> Self {
        Self::eth(EthVersion::Eth66)
    }

    // EthVersion::Eth67에 맞는 Protocol을 생성
    pub const fn eth_67() -> Self {
        Self::eth(EthVersion::Eth67)
    }

    // EthVersion::Eth68에 맞는 Protocol을 생성
    pub const fn eth_68() -> Self {
        Self::eth(EthVersion::Eth68)
    }
}
```
#### ② 내부 데이터 처리 
```Rust
impl Protocol {
    // protocol 객체 Capability와 메시지 개수로 분리
    #[inline]
    pub(crate) fn split(self) -> (Capability, u8) {
        (self.cap, self.messages)
    }

    // 해당 프로토콜이 사용하는 메시지 개수 반환
    pub fn messages(&self) -> u8 {
        if self.cap.is_eth() {
            return EthMessageID::max() + 1
        }
        self.messages
    }
}
```
#### ③ 프로토콜 버전 관리
- 프로토콜 버전과 메시지 수를 추적
```Rust
pub(crate) struct ProtoVersion {
    pub(crate) messages: u8,
    pub(crate) version: usize,
}
```

이렇게 기본 설정을 해주고 나면 파일들은 다음과 같이 동작한다.   
1 . 초기 연결 : `hello.rs`를 통해 peer 초기 연결  
2 . 기능 협상 : `capability.rs` 통해 서로 지원 기능 확인 후 사용할 프로토콜 협상  
3. 프로토콜 설정 : `protocol.rs`에서 프로토콜 연결 설정 후 사용 가능한 프로토콜 협상  
4. 데이터 전송 : `ethstream.rs` , `p2pstream.rs`에서 데이터 스트리밍 방식으로 전송하면   
               `multiplex.rs` 통해 여러 프로토콜 동시 처리
5. 연결 유지 및 종료 : `pinger.rs`에서 네트워크 상태 확인하고   
                   `disconnect.rs`에서 연결 종료
그래서 하나씩 살펴 보자면 
### hello.rs
- P2P 프로토콜에서 핸드셰이크에 사용되는 메시지를 정의하고 처리 
- builder를 통해 `HelloMessage` 빌더를 생성한다. peer 간의 프로토콜 정보를 교환하고 message()와 같은 메서드를 통해 필요한 정보를 추출해 `HelloMessage`로 변환한다.  

[File : crates/net/network/src/hello.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/hello.rs) 

#### ① `HellowMessageWithProtocols`
- 핸드셰이크 시 기본 `HellowMessage` + 사용 프로토콜 정보
```Rust
pub struct HelloMessageWithProtocols {
    // p2p 프로토콜 버전
    pub protocol_version: ProtocolVersion,
    // 클라이언트 소프트웨어 정보
    pub client_version: String,
    // peer 지원 프로토콜 목록
    pub protocols: Vec<Protocol>,
    // 클라이언트가 수신하는 포트
    pub port: u16,
    // peer 공개키 사용하는 식별자
    pub id: PeerId,
}
```
``` Rust
impl HelloMessageWithProtocols {
    pub const fn builder(id: PeerId) -> HelloMessageBuilder {
        HelloMessageBuilder::new(id)
    }

    #[inline]
    pub fn message(&self) -> HelloMessage {
        HelloMessage {
            protocol_version: self.protocol_version,
            client_version: self.client_version.clone(),
            capabilities: self.protocols.iter().map(|p| p.cap.clone()).collect(),
            port: self.port,
            id: self.id,
        }
    }

    pub fn into_message(self) -> HelloMessage {
        HelloMessage {
            protocol_version: self.protocol_version,
            client_version: self.client_version,
            capabilities: self.protocols.into_iter().map(|p| p.cap).collect(),
            port: self.port,
            id: self.id,
        }
    }

    #[inline]
    pub fn contains_protocol(&self, protocol: &Protocol) -> bool {
        self.protocols.iter().any(|p| p.cap == protocol.cap)
    }

    #[inline]
    pub fn try_add_protocol(&mut self, protocol: Protocol) -> Result<(), Protocol> {
        if self.contains_protocol(&protocol) {
            Err(protocol)
        } else {
            self.protocols.push(protocol);
            Ok(())
        }
    }
}
```
- 빌더 관련
    - `builder`
    - `message`
    - `into_message`
- 프로토콜 관리 관련
    - `contatins_protocol` : 메시지가 특정 프로토콜 포함하는지 여부 확인
    - `try_add_protocol` : 새로운 프로토콜 추가 

#### ② `HellowMessage`
- `RLPx` 핸드셰이크 과정에서 기본적인 peer 정보 / 기능 전달 메시지
- 프로토콜, 클라이언트 버전, 포트 정보 포함
```Rust 
pub struct HelloMessage {
    // P2P 프로토콜 버전
    pub protocol_version: ProtocolVersion,
    // 클라이언트 소프트웨어 정보
    pub client_version: String,
    // peer 지원 기능 목록
    pub capabilities: Vec<Capability>,
    // 수신 듣는 포트
    pub port: u16,
    // peer 공개키 사용하는 식별자
    pub id: PeerId,
}
```
```Rust
impl HelloMessage {
  ` // HelloMessage 인스턴스 생성 빌더 호출
    pub const fn builder(id: PeerId) -> HelloMessageBuilder {
        HelloMessageBuilder::new(id)
    }
}
```
#### ③ `HellowMessageBuilder`
```Rust
pub struct HelloMessageBuilder {
    pub protocol_version: Option<ProtocolVersion>,
    pub client_version: Option<String>,
    pub protocols: Option<Vec<Protocol>>,
    pub port: Option<u16>,
    pub id: PeerId,
}
```
### capability.rs
- 메시지 처리 및 협상 처리 모듈

[File : crates/net/network/src/capability.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/capability.rs) 
#### ① `CapabilityMessage` 처리
- `RawCapabiltiyMessage` : 네트워크에서 주고받는 메시지ID와 페이로드 저장
    ```Rust
    pub struct RawCapabilityMessage {
    /// Identifier of the message.
        pub id: usize,
    /// Actual payload
        pub payload: Bytes,
    }
    ```
- `CapabilityMessage` : 이더리움 네트워크의 `EthMessage`와 기타 메시지 `RawCapabilityMessage`를 구분
    ```Rust
    pub enum CapabilityMessage {
        Eth(EthMessage),
        Other(RawCapabilityMessage),
    }
    ```
#### ② `SharedCapability` 
```Rust 
// SharedCapability 구조체 정의
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum SharedCapability {
    Eth { version: EthVersion, offset: u8 },
    UnknownCapability { cap: Capability, offset: u8, messages: u8 },
}

impl SharedCapability {
    // 새 SharedCapability 생성
    pub(crate) fn new(
        name: &str,
        version: u8,
        offset: u8,
        messages: u8,
    ) -> Result<Self, SharedCapabilityError> {
        if offset <= MAX_RESERVED_MESSAGE_ID {
            return Err(SharedCapabilityError::ReservedMessageIdOffset(offset))
        }
        match name {
            "eth" => Ok(Self::eth(EthVersion::try_from(version)?, offset)),
            _ => Ok(Self::UnknownCapability { 
                cap: Capability::new(name.to_string(), version as usize), 
                offset, 
                messages 
            }),
        }
    }

    // eth SharedCapability 생성
    pub(crate) const fn eth(version: EthVersion, offset: u8) -> Self {
        Self::Eth { version, offset }
    }

    // Capability 반환
    pub const fn capability(&self) -> Cow<'_, Capability> {
        match self {
            Self::Eth { version, .. } => Cow::Owned(Capability::eth(*version)),
            Self::UnknownCapability { cap, .. } => Cow::Borrowed(cap),
        }
    }

    // Capability 이름 반환
    pub fn name(&self) -> &str {
        match self {
            Self::Eth { .. } => "eth",
            Self::UnknownCapability { cap, .. } => cap.name.as_ref(),
        }
    }

    // Capability가 eth인지 확인
    pub const fn is_eth(&self) -> bool {
        matches!(self, Self::Eth { .. })
    }

    // Capability 버전 반환
    pub const fn version(&self) -> u8 {
        match self {
            Self::Eth { version, .. } => *version as u8,
            Self::UnknownCapability { cap, .. } => cap.version as u8,
        }
    }

    // eth 버전 반환
    pub const fn eth_version(&self) -> Option<EthVersion> {
        match self {
            Self::Eth { version, .. } => Some(*version),
            _ => None,
        }
    }

    // 메시지 ID 오프셋 반환
    pub const fn message_id_offset(&self) -> u8 {
        match self {
            Self::Eth { offset, .. } | Self::UnknownCapability { offset, .. } => *offset,
        }
    }

    // 예약 메시지 ID 오프셋에서 상대적 메시지 ID 오프셋 반환
    pub const fn relative_message_id_offset(&self) -> u8 {
        self.message_id_offset() - MAX_RESERVED_MESSAGE_ID - 1
    }

    // 프로토콜 메시지 개수 반환
    pub const fn num_messages(&self) -> u8 {
        match self {
            Self::Eth { version: _version, .. } => EthMessageID::max() + 1,
            Self::UnknownCapability { messages, .. } => *messages,
        }
    }
}

```
### ethstream.rs
- 핸드셰이크: `UnauthedEthStream`에서 상태 메시지 통해 핸드셰이크 수행 후, 성공 시 `EthStream`으로 전환
- 메시지 전송/수신: `EthStream` 은 이더리움 메시지 인코딩 후 전송, 스트림에서 수신된 데이터 디코딩
- 연결 관리: `CanDisconnect` 트레이트를 통해 연결 안전하게 종료  

[File : crates/net/network/src/ethstream.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/ethstream.rs) 
#### ① `UnauthedEthStream`
- 상태 핸드셰이크 완료 X 스트림
```Rust
#[pin_project]
#[derive(Debug)]
pub struct UnauthedEthStream<S> {
    #[pin]
    inner: S,
}

impl<S> UnauthedEthStream<S> {
    // 생성자: UnauthedEthStream 생성
    pub const fn new(inner: S) -> Self { Self { inner } }

    // 내부 스트림 반환
    pub fn into_inner(self) -> S { self.inner }

    // 핸드셰이크 처리
    pub async fn handshake(
        self, status: Status, fork_filter: ForkFilter
    ) -> Result<(EthStream<S>, Status), EthStreamError> {
        self.handshake_with_timeout(status, fork_filter, HANDSHAKE_TIMEOUT).await
    }

    // 핸드셰이크 타임아웃 처리
    pub async fn handshake_with_timeout(
        self, status: Status, fork_filter: ForkFilter, timeout_limit: Duration
    ) -> Result<(EthStream<S>, Status), EthStreamError> {
        timeout(timeout_limit, Self::handshake_without_timeout(self, status, fork_filter)).await.map_err(|_| EthStreamError::StreamTimeout)?
    }

    // 타임아웃 없이 핸드셰이크
    pub async fn handshake_without_timeout(
        mut self, status: Status, fork_filter: ForkFilter
    ) -> Result<(EthStream<S>, Status), EthStreamError> {
        // 메시지 송수신 및 검증 로직
    }
}
```
#### ② `EthStream`
- 이더리움 메시지 처리
```Rust
#[pin_project]
#[derive(Debug)]
pub struct EthStream<S> {
    version: EthVersion,
    #[pin]
    inner: S,
}

impl<S> EthStream<S> {
    // 생성자: EthStream 생성
    pub const fn new(version: EthVersion, inner: S) -> Self { Self { version, inner } }

    // 이더리움 버전 반환
    pub const fn version(&self) -> EthVersion { self.version }

    // 내부 스트림 반환
    pub const fn inner(&self) -> &S { &self.inner }

    // 내부 스트림 수정 가능 반환
    pub fn inner_mut(&mut self) -> &mut S { &mut self.inner }

    // 내부 스트림으로 변환
    pub fn into_inner(self) -> S { self.inner }
}
```
#### ③ `EthStream` 메시지
```Rust
impl<S, E> EthStream<S>
where
    S: Sink<Bytes, Error = E> + Unpin, EthStreamError: From<E>
{
    // 브로드캐스트 메시지 전송
    pub fn start_send_broadcast(
        &mut self, item: EthBroadcastMessage
    ) -> Result<(), EthStreamError> {
        self.inner.start_send_unpin(Bytes::from(alloy_rlp::encode(ProtocolBroadcastMessage::from(item))))?;
        Ok(())
    }

    // 다음 메시지 폴링
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        let res = ready!(self.project().inner.poll_next(cx));
        // 디코딩 및 처리 로직
    }

    // 메시지 전송 시작
    fn start_send(self: Pin<&mut Self>, item: EthMessage) -> Result<(), Self::Error> {
        self.project().inner.start_send(Bytes::from(alloy_rlp::encode(ProtocolMessage::from(item))))?;
        Ok(())
    }
}
```
#### ④ `CanDisconnect` 트레이트
```Rust
impl<S> CanDisconnect<EthMessage> for EthStream<S>
where
    S: CanDisconnect<Bytes> + Send,
    EthStreamError: From<<S as Sink<Bytes>>::Error>,
{
    // 연결 해제
    async fn disconnect(&mut self, reason: DisconnectReason) -> Result<(), EthStreamError> {
        self.inner.disconnect(reason).await.map_err(Into::into)
    }
}
```

### p2pstream.rs
- P2P 데이터 스트림을 관리하는 모듈
- peer 간의 데이터를 효율적으로 교환

[File : crates/net/eth-wire/src/p2pstream.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/eth-wire/src/p2pstream.rs) 
#### ① 핸드셰이크 기능
```Rust
impl<S> UnauthedP2PStream<S>
where
    S: Stream<Item = io::Result<BytesMut>> + Sink<Bytes, Error = io::Error> + Unpin,
{
    pub async fn handshake(
        mut self,
        hello: HelloMessageWithProtocols,
    ) -> Result<(P2PStream<S>, HelloMessage), P2PStreamError> {
        self.inner.send(alloy_rlp::encode(P2PMessage::Hello(hello.message())).into()).await?;
        ...
        Ok((stream, their_hello))
    }

    pub async fn send_disconnect(
        &mut self,
        reason: DisconnectReason,
    ) -> Result<(), P2PStreamError> {
        self.inner.send(Bytes::from(alloy_rlp::encode(P2PMessage::Disconnect(reason)))).await?;
        Ok(())
    }
}
```
- `handshake` : `Hello` 메시지 주고 받으면서 핸드셰이크 수행 후 `P2PStream` 반환
- `send_disconnect` : 핸드셰이크 도중 문제 발생 시 `DisconnectReason` 전달해서 연결을 끊음
#### ② 메시지 송수신
```Rust
impl<S> P2PStream<S>
where
    S: Stream<Item = io::Result<BytesMut>> + Sink<Bytes, Error = io::Error> + Unpin,
{
    fn send_pong(&mut self) {
        self.outgoing_messages.push_back(Bytes::from(alloy_rlp::encode(P2PMessage::Pong)));
    }

    pub fn send_ping(&mut self) {
        self.outgoing_messages.push_back(Bytes::from(alloy_rlp::encode(P2PMessage::Ping)));
    }

    fn start_send(self: Pin<&mut Self>, item: Bytes) -> Result<(), Self::Error> {
        ...
        self.outgoing_messages.push_back(compressed.freeze());
        Ok(())
    }
}
```
- `Ping`, `Pong`, 하위 프로토콜 메시지 송수신 
#### ③ 연결 해제 및 종료 
```Rust 
impl<S> DisconnectP2P for P2PStream<S> {
    fn start_disconnect(&mut self, reason: DisconnectReason) -> Result<(), P2PStreamError> {
        ...
        self.outgoing_messages.push_back(compressed.into());
        self.disconnecting = true;
        Ok(())
    }

    pub async fn disconnect(&mut self, reason: DisconnectReason) -> Result<(), P2PStreamError> {
        self.start_disconnect(reason)?;
        self.close().await
    }
}
```
- `start_disconnect` : `DisconnectReason` 보내면서 연결 종료 준비
- `disconnect` : 메시지 송신 후 연결 종료 
#### ④

### multiplex.rs
-  `RLPx` 프로토콜에 대한 멀티플렉서 역할 : 여러 프로토콜을 관리하고 추가
- 멀티 플렉서 생성, 프로토콜 설치, 메시지 관리, 위성 스트림

[File : crates/net/eth-wire/src/multiplex.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/eth-wire/src/multiplex.rs)
#### ① 생성 및 초기화
```Rust
impl<St> RlpxProtocolMultiplexer<St> {
    // 새 P2P 스트림 생성하는 멀티플렉서 초기화
    pub fn new(conn: P2PStream<St>) -> Self

    // 프로토콜 설치해 추가적인 스트림 핸들링
    pub fn install_protocol<F, Proto>(
        &mut self, cap: &Capability, f: F
    ) -> Result<(), UnsupportedCapabilityError>

    // SharedCapabilities 반환
    pub const fn shared_capabilities(&self) -> &SharedCapabilities

    // 주 프로토콜로 위성 스트림을 변환
    pub fn into_satellite_stream<F, Primary>(
        self, cap: &Capability, primary: F
    ) -> Result<RlpxSatelliteStream<St, Primary>, P2PStreamError>

    // 핸드셰이크 후 위성 스트림으로 변환 (타임아웃 가능)
    pub async fn into_satellite_stream_with_handshake<F, Fut, Err, Primary>(
        self, cap: &Capability, handshake: F
    ) -> Result<RlpxSatelliteStream<St, Primary>, Err>

    // ETH 프로토콜을 사용해 위성 스트림으로 변환
    pub async fn into_eth_satellite_stream(
        self, status: Status, fork_filter: ForkFilter
    ) -> Result<(RlpxSatelliteStream<St, EthStream<ProtocolProxy>>, Status), EthStreamError>
}
```
#### ② `MultiplexInner`
```Rust
impl<St> MultiplexInner<St> {
    // SharedCapabilities 반환
    const fn shared_capabilities(&self) -> &SharedCapabilities

    // 받은 메시지 적절한 프로토콜로 전달
    fn delegate_message(&self, cap: &SharedCapability, msg: BytesMut) -> bool

    // 새로운 프로토콜 설치해 추가적인 메시지 스트림 관리
    fn install_protocol<F, Proto>(
        &mut self, cap: &Capability, f: F
    ) -> Result<(), UnsupportedCapabilityError>
}
```
#### ③ `ProtocolProxy` : 프로토콜에 메시지 처리 후 메시지 ID를 변환(mask/unmask)하여 송수신 처리
```Rust
impl ProtocolProxy {
    // 메시지를 송신
    fn try_send(&self, msg: Bytes) -> Result<(), io::Error>

    // 메시지 ID 마스킹
    fn mask_msg_id(&self, msg: Bytes) -> Result<Bytes, io::Error>

    // 메시지 ID 언마스킹
    fn unmask_id(&self, mut msg: BytesMut) -> Result<BytesMut, io::Error>
}
```
#### ④ `RlpxSatelliteStream` : 주 프로토콜과 추가 프로토콜 관리해 스트림과 Sink로 동작
```Rust
impl<St, Primary> RlpxSatelliteStream<St, Primary> {
    // 새로운 프로토콜 설치
    pub fn install_protocol<F, Proto>(
        &mut self, cap: &Capability, f: F
    ) -> Result<(), UnsupportedCapabilityError>

    // 주 프로토콜 반환
    pub const fn primary(&self) -> &Primary

    // 주 프로토콜 수정 가능 접근
    pub fn primary_mut(&mut self) -> &mut Primary

    // 내부 P2PStream 반환
    pub const fn inner(&self) -> &P2PStream<St>
}
```

### pinger.rs
- `Pinger`는 주기적으로 `ping` 메시지 보내고 `pong` 메시지 기다린다.

[File : crates/net/eth-wire/src/pinger.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/eth-wire/src/pinger.rs)
#### ① 구조체 및 생성
- 일정 간격으로 `ping` 메시지 보내고 `pong` 응답 처리 
    ```Rust 
    pub(crate) struct Pinger {
        ping_interval: Interval,
        timeout_timer: Pin<Box<Sleep>>,
        timeout: Duration,
        state: PingState,
    }
    ```
- `PingState`
    ```Rust
    pub(crate) enum PingState {
        // ping 보낼 준비가 된 상태 
        Ready,
        // ping 보내고 pong 응답 기다리는 상태
        WaitingForPong,
        // 시간 초과
        TimedOut,
    }
    ```
- `PingerEvent`
    ```Rust
    pub(crate) enum PingerEvent {
        // 새로운 ping 보내야함
        Ping,
        // 시간초과로 연결 종료 해야함
        Timeout,
    }
    ```
    ```Rust 
    impl Pinger {
        /// 새로운 Pinger를 생성 : ping 간격 , timeout 설정
        pub(crate) fn new(ping_interval: Duration, timeout_duration: Duration) -> Self
    }
    ```
#### ② 상태 관리 및 상태 전환
```Rust
impl Pinger {
    // pong 메시지를 받았을 때 상태를 전환
    pub(crate) fn on_pong(&mut self) -> Result<(), PingerError>

    // 현재 상태를 반환
    pub(crate) const fn state(&self) -> PingState
}
```
#### ③ `Ping` 이벤트 처리
```Rust
impl Pinger {
    // Ping 이벤트를 처리하는 상태 기반 폴링 메서드
    pub(crate) fn poll_ping(
        &mut self, cx: &mut Context<'_>
    ) -> Poll<Result<PingerEvent, PingerError>>
}
```
### disconnect.rs

[File : crates/net/eth-wire/src/disconnect.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/eth-wire/src/disconnect.rs)
#### ① `CanDisconnect` 트레이트 
```Rust
pub trait CanDisconnect<T>: Sink<T> + Unpin {
    // DisconnectReason 이용해 연결 끊기
    fn disconnect(
        &mut self,
        reason: DisconnectReason,
    ) -> impl Future<Output = Result<(), <Self as Sink<T>>::Error>> + Send;
}
```
#### ② `Framed`와  `ECIESStream` 구현
```Rust
impl<T, I, U> CanDisconnect<I> for Framed<T, U>
where
    T: AsyncWrite + Unpin + Send,
    U: Encoder<I> + Send,
{
    async fn disconnect(
        &mut self,
        _reason: DisconnectReason,
    ) -> Result<(), <Self as Sink<I>>::Error> {
        self.close().await
    }
}

impl<S> CanDisconnect<bytes::Bytes> for ECIESStream<S>
where
    S: AsyncWrite + Unpin + Send,
{
    async fn disconnect(&mut self, _reason: DisconnectReason) -> Result<(), std::io::Error> {
        self.close().await
    }
}
```
- 비동기 스트림의 연결 종료 처리
- `close()` 호출해서 처리