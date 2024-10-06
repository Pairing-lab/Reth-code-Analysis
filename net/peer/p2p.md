# p2p
> 블록체인 노드 간에 데이터를 교환하고 동기화하는 역할

- [p2p](#p2p-1)
    - [lib.rs](#librs)
    - [download.rs](#downloadrs)
    - [sync.rs](#syncrs)
    - [full_block.rs](#full_blockrs)
    - [error.rs](#errorrs)
    - [priority.rs](#priorityrs)
    - [either.rs](#eitherrs)  
- [headers](#headers)
    - [mod.rs](#modrs)
    - [error.rs](#errors)
    - [client.rs](#clientrs)
    - [downloader.rs](#downloaderrs)
- [bodies](#bodies)
    - [mod.rs](#modrs)
    - [client.rs](#clientrs)
    - [downloader.rs](#downloaderrs)
    - [response.rs](#responsers)

## p2p
[File : crates/net/p2p/src](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src)  

### lib.rs
---
- p2p 모듈의 전체 구조 파악 

[File : crates/net/p2p/src/lib.rs](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/lib.rs) 
```Rust
pub mod download;
pub mod bodies;
pub mod either;
pub mod full_block;
pub mod headers;
pub mod error;
pub mod priority;
pub mod sync;
pub mod test_utils;
pub use bodies::client::BodiesClient;
pub use headers::client::HeadersClient;
pub trait BlockClient: HeadersClient + BodiesClient + Unpin + Clone {}
impl<T> BlockClient for T where T: HeadersClient + BodiesClient + Unpin + Clone {}
```
--- 
### download.rs
[File : crates/net/p2p/src/download.rs](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/download.rs) 

---
- 블록 데이터 다운로드 기능 
```Rust
pub trait DownloadClient: Send + Sync + Debug {
    fn report_bad_message(&self, peer_id: PeerId);
    fn num_connected_peers(&self) -> usize;
}
```
---
### sync.rs

---
- 네트워크 동기화 로직 
- 노드 간의 데이터 동기화 , 블록 및 트랜잭션 전파

[File : crates/net/p2p/src/sync.rs](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/sync.rs) 
#### ① 동기화 상태 확인
```Rust
pub trait SyncStateProvider: Send + Sync {
    // 네트워크 동기화 여부 반환
    fn is_syncing(&self) -> bool;
    // 네트워크 초기 동기화 여부 반환
    fn is_initially_syncing(&self) -> bool;
}
```
#### ② 동기화 상태 업데이트
```Rust
pub trait NetworkSyncUpdater: std::fmt::Debug + Send + Sync + 'static {
    // 네트워크 동기화 상태 변경 시 호출
    fn update_sync_state(&self, state: SyncState);
    // p2p 네트워크 노드 상태 업데이트 
    fn update_status(&self, head: Head);
}
```
#### ③ 동기화 상태 관리 
```Rust
pub enum SyncState {
    // 동기화 완료 상태
    Idle,
    // 동기화 중 
    Syncing,
}
```
#### ④ 기본 상태 반환
```Rust
pub struct NoopSyncStateUpdater;
```
--- 
### full_block.rs
---
- 블록 구조와 처리 방식 로직

[File : crates/net/p2p/src/full_block.rs](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/full_block.rs) 
#### ① `FullBlockClient` 
- 블록 헤더와 바디 요청해 완전한 블록을 가지고 오는 client
```Rust
pub struct FullBlockClient<Client> {
    client: Client,
    consensus: Arc<dyn Consensus>,
}
```
```Rust
impl<Client> FullBlockClient<Client> {
    pub fn new(client: Client, consensus: Arc<dyn Consensus>) -> Self {
        Self { client, consensus }
    }
}
```
- `new` : 새로운 client 생성
``` Rust
impl<Client> FullBlockClient<Client>
where
    Client: BlockClient,
{
    pub fn get_full_block(&self, hash: B256) -> FetchFullBlockFuture<Client> {
        ...
    }

    pub fn get_full_block_range(
        &self,
        hash: B256,
        count: u64,
    ) -> FetchFullBlockRangeFuture<Client> {
        ...
    }
}
```
- `get_full_block` : 특정 블록 해시에 대해 전체 블록 가져옴
- `get_full_block_range` : 주어진 해시와 범위에 대해 여러 블록 가져옴
#### ② `FetchFullBlockFuture`
- 단일 블록을 가져옴
- 헤더와 바디 동시에 요청하고, 응답을 합쳐서 전체 블록으로 반환한다.
```Rust
pub struct FetchFullBlockFuture<Client>
where
    Client: BlockClient,
{
    client: Client,
    hash: B256,
    request: FullBlockRequest<Client>,
    header: Option<SealedHeader>,
    body: Option<BodyResponse>,
}
```
```Rust
impl<Client> FetchFullBlockFuture<Client>
where
    Client: BlockClient,
{
    fn take_block(&mut self) -> Option<SealedBlock> {
        ...
    }

    fn on_block_response(&mut self, resp: WithPeerId<BlockBody>) {
        ...
    }
}
```
- `take_block` : 블록 바디 응답 처리 , 헤더와 일치 확인
- `on_block_response` : 블록 유효성 확인 

```Rust
impl<Client> Future for FetchFullBlockFuture<Client>
where
    Client: BlockClient + 'static,
{
    type Output = SealedBlock;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        ...
    }
}
```
- `poll` : 요청한 블록 반환 확인을 위해 지속적으로 호출
#### ③ `FetchFullBlockRangeFuture`
- `poll` : 블록 범위 요청의 진행 상태를 확인
- `on_headers_response` : 헤더 응답 처리 후 바디 요청까지
    ```Rust
    fn on_headers_response(&mut self, headers: WithPeerId<Vec<Header>>) {...}
    ```
- `take_blocks` : 유효한 블록 조합해서 반환
    ```Rust
    fn take_blocks(&mut self) -> Option<Vec<SealedBlock>> {...}
    ```
#### ④ 유효성 검사 : `ensure_valid_body_response`
#### ⑤ `FullBlockRequest`
- 헤더와 바디 요청 관리
- 응답올 때까지 대기 (비동기)
```Rust
struct FullBlockRequest<Client>
where
    Client: BlockClient,
{
    header: Option<SingleHeaderRequest<<Client as HeadersClient>::Output>>,
    body: Option<SingleBodyRequest<<Client as BodiesClient>::Output>>,
}
```
--- 

### error.rs
---
- 네트워크 통신 및 동기화 중 발생하는 에러 처리  
[File : crates/net/p2p/src/error.rs](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/error.rs) 
--- 
### priority.rs
---
- 네트워크 요청 및 처리 우선 순위 관리  

[File : crates/net/p2p/src/priority.rs](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/priority.rs) 
```Rust
pub enum Priority {
    Normal,
    High,
}

impl Priority {
    pub const fn is_high(&self) -> bool {
        matches!(self, Self::High)
    }
    pub const fn is_normal(&self) -> bool {
        matches!(self, Self::Normal)
    }
}
- 우선순위를 `High`와 `Normal`로 정의하고 확인하고 설정함

### either.rs
- 선택적 경로 처리 로직 
- `DownloadClient`
```Rust
// A, B 클라이언트가 수신한 잘못된 메시지 보고 
impl<A, B> DownloadClient for Either<A, B>
where
    A: DownloadClient,
    B: DownloadClient,
{
    // A, B 클라이언트가 수신한 잘못된 메시지 보고 
    // Left와 Right 중 선택
    fn report_bad_message(&self, peer_id: reth_network_peers::PeerId) {
        match self {
            Self::Left(a) => a.report_bad_message(peer_id),
            Self::Right(b) => b.report_bad_message(peer_id),
        }
    }
    // 연결된 Peer 수 반환 
    fn num_connected_peers(&self) -> usize {
        match self {
            Self::Left(a) => a.num_connected_peers(),
            Self::Right(b) => b.num_connected_peers(),
        }
    }
}
```
- `BodiesClient` : 블록 바디 가져오는 기능 
```Rust
impl<A, B> BodiesClient for Either<A, B>
where
    A: BodiesClient,
    B: BodiesClient,
{
    type Output = Either<A::Output, B::Output>;
    
    // 블록 해시 목록에서 블록 바디와 우선 순위를 가져옴 
    fn get_block_bodies_with_priority(
        &self,
        hashes: Vec<B256>,
        priority: Priority,
    ) -> Self::Output {
        // Left와 Right 중 선택해서 반환
        match self {
            Self::Left(a) => Either::Left(a.get_block_bodies_with_priority(hashes, priority)),
            Self::Right(b) => Either::Right(b.get_block_bodies_with_priority(hashes, priority)),
        }
    }
}
```
- `HeadersClient` : 블록 헤더 가져오는 기능 (`BodiesClient`와 같은 로직)
```Rust
impl<A, B> HeadersClient for Either<A, B>
where
    A: HeadersClient,
    B: HeadersClient,
{
    ...
}
```
---
## header
[File : crates/net/p2p/src/headers](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/headers)

### mod.rs
---
- `header` 모듈 전체 인터페이스 정의  

[File : crates/net/p2p/src/headers/mod.rs](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/headers/mod.rs)

```Rust
pub mod client;
pub mod downloader;
pub mod error;
```
---
### error.rs
---
- 헤더 관련 작업 중 발생하는 에러 처리  

[File : crates/net/p2p/src/headers/error.rs](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/headers/error.rs)
```Rust
// 결과값 타입 정의
pub type HeadersDownloaderResult<T> = Result<T, HeadersDownloaderError>;
// 헤더 다운로드 중 발생할 수 있는 에러 정의 열거형
pub enum HeadersDownloaderError {
    // 다운로드는 했으나 로컬 헤드와 연결되지 않을 때 에러 
    DetachedHead {
    local_head: Box<SealedHeader>,
    header: Box<SealedHeader>,
    error: Box<ConsensusError>,
    }
}
```
---
### client.rs
---
- 블록 헤더 클라이언트 로직 처리
- 블록 헤더 요청 및 처리   

[File : crates/net/p2p/src/headers/client.rs](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/headers/client.rs)
#### ① `HeadersRequest` 구조체 
```Rust
pub struct HeadersRequest {
    // 헤더 요청 시작할 블록의 해시 또는 번호 지정
    pub start: BlockHashOrNumber,
    // 한번에 수신 가능한 최대 헤더 개수 지정
    pub limit: u64,
    // 헤더 반환 방향 지정 (Rising, Falling ...)
    pub direction: HeadersDirection,
}
```
#### ② 헤더 요청 관련
```Rust
pub trait HeadersClient: DownloadClient {
    type Output: Future<Output = PeerRequestResult<Vec<Header>>> + Sync + Send + Unpin;

    fn get_headers(&self, request: HeadersRequest) -> Self::Output {
        self.get_headers_with_priority(request, Priority::Normal)
    }
    fn get_headers_with_priority(
        &self,
        request: HeadersRequest,
        priority: Priority,
    ) -> Self::Output;

    fn get_header(&self, start: BlockHashOrNumber) -> SingleHeaderRequest<Self::Output> {...}
    fn get_header_with_priority(
        &self,
        start: BlockHashOrNumber,
        priority: Priority,
    ) -> SingleHeaderRequest<Self::Output> {
        let req = HeadersRequest {...}
    }
}
```
- `get_headers` : `HeadersRequest` 기반으로 헤더를 가져온다. `priority` default는 `Normal`
- `get_headers_with_priority` : `priority`를 지정해서 헤더 요청을 보냄
- 단일 헤더 요청 : `get_header` , `get_header_with_priority` 
#### ③ 비동기 처리 : 단일 헤더 요청 비동기 작업 완료 확인 및 결과 반환
```Rust
pub struct SingleHeaderRequest<Fut> {
    fut: Fut,
}

impl<Fut> Future for SingleHeaderRequest<Fut>
where
    Fut: Future<Output = PeerRequestResult<Vec<Header>>> + Sync + Send + Unpin,
{
    type Output = PeerRequestResult<Option<Header>>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let resp = ready!(self.get_mut().fut.poll_unpin(cx));
        let resp = resp.map(|res| res.map(|headers| headers.into_iter().next()));
        Poll::Ready(resp)
    }
}
```
---
### downloader.rs 
---
- 블록 헤더 다운로드 로직 
- 다운로드 전략 및 우선순위 설정  

[File : crates/net/p2p/src/headers/downloader.rs](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/headers/downloader.rs)

```Rust
pub trait HeaderDownloader:
    Send + Sync + Stream<Item = HeadersDownloaderResult<Vec<SealedHeader>>> + Unpin
{
    // 로컬 헤드와 동기화 대상 사이의 동기화 갭 설정
    fn update_sync_gap(&mut self, head: SealedHeader, target: SyncTarget) {
        self.update_local_head(head);
        self.update_sync_target(target);
    }
    // 로컬 블록 헤드 업데이트
    fn update_local_head(&mut self, head: SealedHeader);
    // 동기화 대상 목표 업데이트 
    fn update_sync_target(&mut self, target: SyncTarget);
    // 다운로드할 헤더 배치 크기 설정
    fn set_batch_size(&mut self, limit: usize);
}
```
- 그 외 동기화할 블록 관련 `SyncTarget` 열거형과 `validate` 관련 코드가 있다
---
## bodies
[File : crates/net/p2p/src/bodies](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/bodies)
### mod.rs 
---
- `bodies` 모듈 전체 인터페이스 정의  

[File : crates/net/p2p/src/bodies/mod.rs](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/bodies/mod.rs)
```Rust
pub mod client;
pub mod downloader;
pub mod response;
```
---
### client.rs
---
- 블록 바디 클라이언트 로직 처리
- 블록 바디 요청 및 처리 

[File : crates/net/p2p/src/bodies/client.rs](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/bodies/client.rs)
```Rust
pub trait BodiesClient: DownloadClient {
    type Output: Future<Output = PeerRequestResult<Vec<BlockBody>>> + Sync + Send + Unpin;

    fn get_block_bodies(&self, hashes: Vec<B256>) -> Self::Output {
        self.get_block_bodies_with_priority(hashes, Priority::Normal)
    }

    fn get_block_bodies_with_priority(&self, hashes: Vec<B256>, priority: Priority)
        -> Self::Output;

    fn get_block_body(&self, hash: B256) -> SingleBodyRequest<Self::Output> {...}

    fn get_block_body_with_priority(
        &self,
        hash: B256,
        priority: Priority,
    ) -> SingleBodyRequest<Self::Output> {...}
}
```
- `get_block_bodies`: 여러 블록 바디 요청 
- `get_block_bodies_with_priority`: `priority` 지정해서 블록 바디 요청
- 단일 블록 요청 :  `get_block_body`, `get_block_body_with_priority`
`get_block_body_with_priority`
---
### downloader.rs 
---
- 블록 바디 다운로드 로직 
- 다운로드 전략 및 처리 흐름 

[File : crates/net/p2p/src/bodies/downloader.rs](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/bodies/downloader.rs)
```Rust
pub type BodyDownloaderResult = DownloadResult<Vec<BlockResponse>>;

pub trait BodyDownloader: Send + Sync + Stream<Item = BodyDownloaderResult> + Unpin {
    fn set_download_range(&mut self, range: RangeInclusive<BlockNumber>) -> DownloadResult<()>;
}
```
---
### response.rs
---
- 블록 바디 응답 처리 및 오류 관리 

[File : crates/net/p2p/src/bodies/response.rs](https://github.com/paradigmxyz/reth/tree/main/crates/net/p2p/src/bodies/response.rs)
#### ① `BlockResponse` 형태 열거형 : `Full`, `Empty`
```Rust
pub enum BlockResponse {
    // 전체 블록 : 트랜잭션이나 ommer 포함
    Full(SealedBlock),
    // 빈 블록 : 트랜잭션 없음
    Empty(SealedHeader),
}
```
#### ② `BlockResponse` 구현
```Rust
impl BlockResponse {
    // 블록 or 헤더 참조 반환
    pub const fn header(&self) -> &SealedHeader {
        match self {
            Self::Full(block) => &block.header,
            Self::Empty(header) => header,
        }
    }
    // BlockResponse 메모리 내 크기 계산
    pub fn size(&self) -> usize {
        match self {
            Self::Full(block) => SealedBlock::size(block),
            Self::Empty(header) => SealedHeader::size(header),
        }
    }
    // 블록 번호 반환
    pub fn block_number(&self) -> BlockNumber {
        self.header().number
    }
    // 블록 난이도 반환
    pub fn difficulty(&self) -> U256 {
        match self {
            Self::Full(block) => block.difficulty,
            Self::Empty(header) => header.difficulty,
        }
    }
}
```