# listener
- `TcpListener`를 사용하여 `ConnectionListener` 구현  
- `ConnectionListener`는 TCP 연결을 비동기적으로 수신하고 처리   
[File : crates/net/network/src/listener.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/listener.rs)

### 1. `ConnectionListener` 구조체 및 구현

```Rust
pub struct ConnectionListener {
    local_address: SocketAddr,
    incoming: TcpListenerStream,
}
```
- `local_address` : 현재 `listener`가 수신하고 있는 주소.  `TcpListener`가 binding된 주소, 즉, TcpListener가 연결을 수신하고 있는 위치 (IP, 포트번호)라고 할 수 있다. 
- `incoming` : 들어오는 TCP 연결을 비동기 처리 

### bind
```Rust
impl ConnectionListener {
    pub async fn bind(addr: SocketAddr) -> io::Result<Self> {
        let listener = TcpListener::bind(addr).await?;
        let local_addr = listener.local_addr()?;
        Ok(Self::new(listener, local_addr))
    }
}
```
- `bind`
    - `SocketAddr`에서 TCP 리스너를 생성 후 바인딩 
    - `local_addr`로 바인딩된 주소 확인
    - 성공적 바인딩 후 `ConnectionListener`를 반환

### new
```Rust
impl ConnectionListener {
   pub(crate) const fn new(listener: TcpListener, local_address: SocketAddr) -> Self
}
```
- `new` : `TcpListener`와 `local_address`를 받아 새로운 ConnectionListener를 생성

### poll
```Rust
impl ConnectionListener {
   pub fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<ListenerEvent> {
        let this = self.project();
        // 들어오는 다음 연결 확인 
        match ready!(this.incoming.poll_next(cx)) {
            Some(Ok((stream, remote_addr))) => {
                if let Err(err) = stream.set_nodelay(true) {
                    tracing::warn!(target: "net", "set nodelay failed: {:?}", err);
                }
                Poll::Ready
                // 새로운 연결 수신 될 떄 이벤트 발생 
                (ListenerEvent::Incoming { stream, remote_addr })
            }
            // 연결 수신 중 오류 발생하면 이벤트 발생 
            Some(Err(err)) => Poll::Ready(ListenerEvent::Error(err)),
            None => {
                Poll::Ready(ListenerEvent::ListenerClosed { local_address: *this.local_address })
            }
        }
    }
}
```
- `poll` : `ConnectionListener`가 비동기적으로 진행되도록 함

### local_address
```Rust
impl ConnectionListener {
   pub const fn local_address(&self) -> SocketAddr
}
```
- `local_address` : 현재 리스너가 어떤 주소에서 수신 중인지 알 수 있는 `local_address`를 반환

### 2. `ListenerEvent` 열거형 
- TCP 리스너에서 발생할 수 있는 이벤트를 나타낸다.
- `Incoming` : 새로운 연결 수신되었을 때
    ```Rust
    pub enum ListenerEvent {
        Incoming{
            stream : TcpStream,
            remote_addr : SocketAddr,
        },
    }
    ```
- `ListenerClosed` : TCP 리스너가 종료되었을 때  
    ```Rust
    pub enum ListenerEvent {
        ListenerClosed {
            local_address: SocketAddr,
        },
    }
    ```
- `Error` : 연결 수락 중 오류 발생했을 때  
    ```Rust
    pub enum ListenerEvent {
        Error(io::Error),
    }
    ```
### 3. `TcpListenerStream` 구조체 및 구현 
- **비동기적**으로 **TCP 연결을 수신**하고, 이를 처리하는 `stream` 제공 
```Rust
struct TcpListenerStream {
    inner: TcpListener,
}
```
- `inner` : `TcpListener` 타입으로, 실제로 네트워크 소켓에서 들어오는 `TCP 연결을 수신하는 리스너` 역할

```Rust
impl Stream for TcpListenerStream {
    type Item = io::Result<(TcpStream, SocketAddr)>;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        match self.inner.poll_accept(cx) {
            Poll::Ready(Ok(conn)) => Poll::Ready(Some(Ok(conn))),
            Poll::Ready(Err(err)) => Poll::Ready(Some(Err(err))),
            Poll::Pending => Poll::Pending,
        }
    }
}
```
- `poll_next` : 새로운 연결이 있는지 확인하고 stream에서 다음 아이템 비동기적 polling해서 반환