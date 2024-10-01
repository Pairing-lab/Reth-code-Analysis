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