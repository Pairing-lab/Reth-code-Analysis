#Swarm
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

## Swarm 구조체 
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

## Swarm 구현

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