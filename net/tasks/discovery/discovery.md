# Discovery
네트워크에서 노드 발견하고 관리 

## 1. `Discovery` 구조체
```Rust
pub struct Discovery {
    discovered_nodes: LruMap<PeerId, PeerAddr>,            // 발견된 노드의 캐시
    local_enr: NodeRecord,                                 // 로컬 노드의 ENR (Ethereum Node Record)
    discv4: Option<Discv4>,                                // Discovery v4 서비스 핸들러
    discv4_updates: Option<ReceiverStream<DiscoveryUpdate>>, // Discovery v4 업데이트 스트림
    _discv4_service: Option<JoinHandle<()>>,               // 비동기 처리용 v4 서비스 핸들
    discv5: Option<Discv5>,                                // Discovery v5 서비스 핸들러
    discv5_updates: Option<ReceiverStream<discv5::Event>>, // Discovery v5 업데이트 스트림
    _dns_discovery: Option<DnsDiscoveryHandle>,            // DNS 기반 노드 발견 핸들러
    dns_discovery_updates: Option<ReceiverStream<DnsNodeRecordUpdate>>, // DNS 발견 서비스의 업데이트 스트림
    _dns_disc_service: Option<JoinHandle<()>>,             // 비동기 처리용 DNS 서비스 핸들
    queued_events: VecDeque<DiscoveryEvent>,               // 대기 중인 이벤트
    discovery_listeners: Vec<mpsc::UnboundedSender<DiscoveryEvent>>, // 이벤트 리스너
}

```


## 2. `Discovery` 구현
### Methods
---
### 1. new
- 네트워크 초기화 및 탐색 프로토콜 설정
- `discv4`, `discv5`, `DNS` 서비스를 설정하고 비동기로 실행

```Rust
    pub async fn new(
    tcp_addr: SocketAddr,
    discovery_v4_addr: SocketAddr,
    sk: SecretKey,
    discv4_config: Option<Discv4Config>,
    discv5_config: Option<reth_discv5::Config>, 
    dns_discovery_config: Option<DnsDiscoveryConfig>,) -> Result<Self, NetworkError>

```
 
- **tcp_addr** : 노드의 TCP 주소
- **discovery_v4_addr** : v4 프로토콜에 사용할 주소
- **sk** : 노드 비밀키 
- **discv4_config , discv5_config, dns_discovery_config** : discovery 설정
---
### Operations
---
#### ① discovery 4 설정
```Rust
    let discv4_future = async {
    let Some(disc_config) = discv4_config else { return Ok((None, None, None)) };
    let (discv4, mut discv4_service) =
        Discv4::bind(discovery_v4_addr, local_enr, sk, disc_config).await.map_err(
            |err| {
                NetworkError::from_io_error(err, ServiceKind::Discovery(discovery_v4_addr))
            },
        )?;
    let discv4_updates = discv4_service.update_stream();
    let discv4_service = discv4_service.spawn();
    Ok((Some(discv4), Some(discv4_updates), Some(discv4_service)))
    };
```
- `discv4_future` : discovery v4 protocol 설정 (비동기 처리)
- `NodeRecord::from_secret_key` : secret key로부터 로컬 노드 정보 생성 후 TCP 포트 설정
- `Discv4::bind` : discovery v4 서비스 시작
- `spawn` : discovery v4 비동기 실행

#### ② discovery 5 설정 
```Rust
        let discv5_future = async {
        let Some(config) = discv5_config else { return Ok::<_, NetworkError>((None, None)) };
        let (discv5, discv5_updates, _local_enr_discv5) = Discv5::start(&sk, config).await?;
        Ok((Some(discv5), Some(discv5_updates.into())))
        };
```
- `discv5_future` : discovery v5 설정 (비동기 처리)
- `Discv5::start` : discovery v5 서비스 시작

#### ③ DNS 설정 
```Rust
    pub async fn new(
    tcp_addr: SocketAddr,
    discovery_v4_addr: SocketAddr,
    sk: SecretKey,
    discv4_config: Option<Discv4Config>,
    discv5_config: Option<reth_discv5::Config>, 
    dns_discovery_config: Option<DnsDiscoveryConfig>,
    ) -> Result<Self, NetworkError>
```
- `DnsDiscoveryService::new_pair` : DNS 설정 (비동기 처리)
- `spawn` : DNS 비동기 실행  

#### ④ `tokio::try_join!`  : discovery 4, discovery 5 병렬 실행
---

```Rust
    let ((discv4, discv4_updates, _discv4_service), (discv5, discv5_updates)) = 
    tokio::try_join!(discv4_future, discv5_future)?;
```
##### ⑤ `return` : discv4, discv5, DNS 관련 객체 
---
```Rust
    Ok(Self {
    discovery_listeners: Default::default(),
    local_enr,
    discv4,
    discv4_updates,
    _discv4_service,
    discv5,
    discv5_updates,
    discovered_nodes: LruMap::new(DEFAULT_MAX_CAPACITY_DISCOVERED_PEERS_CACHE),
    queued_events: Default::default(),
    _dns_disc_service,
    _dns_discovery,
    dns_discovery_updates,
    })
```
### 2. add_listener
: 발견 이벤트 수신 리스너 등록
```Rust
pub(crate) fn add_listener(&mut self, tx: mpsc::UnboundedSender<DiscoveryEvent>) {
        self.discovery_listeners.push(tx);
    }
```
### 3. notify_listeners
: 등록된 모든 listener에게 Event 알림
```Rust
fn notify_listeners(&mut self, event: &DiscoveryEvent) {
        self.discovery_listeners.retain_mut(|listener| listener.send(event.clone()).is_ok());
    }
```
### 4. update_fork_id
: Discovery v4 `eth:ForkId` 업데이트
```Rust
pub(crate) fn update_fork_id(&self, fork_id: ForkId) {
        if let Some(discv4) = &self.discv4 {
            discv4.set_eip868_rlp(b"eth".to_vec(), EnrForkIdEntry::from(fork_id))
        }
    }
```
### 5. ban_ip, ban
: IP 주소 or Peer ID 차단
``` Rust
// ban_ip
pub(crate) fn ban_ip(&self, ip: IpAddr) {
        if let Some(discv4) = &self.discv4 {
            discv4.ban_ip(ip)
        }
        if let Some(discv5) = &self.discv5 {
            discv5.ban_ip(ip)
        }
    }

// ban 
pub(crate) fn ban(&self, peer_id: PeerId, ip: IpAddr) {
        if let Some(discv4) = &self.discv4 {
            discv4.ban(peer_id, ip)
        }
        if let Some(discv5) = &self.discv5 {
            discv5.ban(peer_id, ip)
        }
    }
```
### 6. poll 
: Discovery 스트림 (v4,v5, DNS)에서 이벤트 폴링하여 처리
```Rust
pub(crate) fn poll(&mut self, cx: &mut Context<'_>) -> Poll<DiscoveryEvent>
```
#### Parameters 
- `&mut self` : discovery 상태 처리
- `cx: &mut Context<'_>` : 비동기 작업을 위한 폴링 관리에 사용
#### return
- `Poll<DiscoveryEvent>` : 폴링 결과로 `DiscoveryEvent` 반환 
    - `Poll::Ready` : 이벤트 준비
    - `Poll::Pending` : 이벤트 준비 X
---
#### ① 이벤트 큐 처리 
```Rust
if let Some(event) = self.queued_events.pop_front() {
    self.notify_listeners(&event);
    return Poll::Ready(event);
}
```
- `queued_events` : 이전에 수신된 이벤트 저장 큐 
- 이벤트 발생 시, 해당 이벤트 큐에 저장 / 처리
#### ② discovery 4 업데이트 스트림 처리
```Rust
while let Some(Poll::Ready(Some(update))) =
    self.discv4_updates.as_mut().map(|updates| updates.poll_next_unpin(cx))
{
    self.on_discv4_update(update)
}
```
-`discv4_updates`가 있을 때, 스트림 폴링해 v4 업데이트 
- 업데이트는 `on_discv4_update` 함수로 전달
#### ③ discovery 5 업데이트 스트림 처리
```Rust
while let Some(Poll::Ready(Some(update))) =
    self.discv4_updates.as_mut().map(|updates| updates.poll_next_unpin(cx))
{
    self.on_discv4_update(update)
}
```
- `on_node_record_update` : 새로운 노드 발견 시 해당 함수로 노드 정보 업데이트 
#### ④ DNS 업데이트 처리 
```Rust
while let Some(Poll::Ready(Some(update))) =
    self.dns_discovery_updates.as_mut().map(|updates| updates.poll_next_unpin(cx))
{
    self.add_discv4_node(update.node_record);
    if let Err(err) = self.add_discv5_node(update.enr) {
        trace!(target: "net::discovery",
            %err,
            "failed adding node discovered by dns to discv5"
        );
    }
    self.on_node_record_update(update.node_record, update.fork_id);
}

```
- 새로운 노드 발견 시 `discv4` 및 `discv5`에 추가 
#### ⑤ 이벤트가 없는 경우
```Rust
if self.queued_events.is_empty() {
    return Poll::Pending
}
```
- 이벤트 없는 경우 : `queued_events`가 비어있으면 `Poll::Pending`반환 

### 7. Stream 
: Discovery 구조체 `Stream`으로 사용 ... event 폴링할 수 있게 한다.
```Rust
impl Stream for Discovery {
    type Item = DiscoveryEvent;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        Poll::Ready(Some(ready!(self.get_mut().poll(cx))))
    }   
}
```