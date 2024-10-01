# Network

Analysis of the networking components involved in Ethereum's peer-to-peer (P2P) communication.

## Contents

### - [Overview]
### - [Network Setting]
### - Tasks
   - Network Handle
   - Transactions
   - ETH Request
   - Discovery
        - Discv4
        - Discv5
        - DNS
### 4. Components
   - Fetch Client
   - Swarm
        - Session Manager
        - Connection Listener
        - State Manager
   - NetworkManager
   - Peer Management
        - Peer
        - P2P 
   - Eth-Wire


## Overview

<img width="739" alt="overview" src="https://github.com/user-attachments/assets/c9be73b4-553f-49dc-b9b9-e4b466ed2ede">

Reth's P2P networking consists primarily of 4 ongoing tasks
:  `Discovery` , `Transactions` , `ETH Requests`, `Network Handle` .

The `Network Handle` manages the network state and interacts with the `Fetch Client` to send `ETH requests`, retrieve `transactions`, and manage `discovery` of peers. The `Swarm` handles **sessions**, **connections**, and **state** management, while the `NetworkManager` coordinates peer connections and P2P protocol operations. `Eth-Wire` facilitates communication between peers by encoding and decoding protocol messages.

## Network Setting

[File : docs/crates/network.md](https://github.com/paradigmxyz/reth/blob/main/docs/crates/network.md?plain=1)

[File: bin/reth/src/node/mod.rs](https://github.com/paradigmxyz/reth/blob/1563506aea09049a85e5cc72c2894f3f7a371581/bin/reth/src/node/mod.rs)

```rust
// Start network
let network = start_network(network_config(db.clone(), chain_id, genesis_hash)).await?;

// Fetch the client
let fetch_client = Arc::new(network.fetch_client().await?);

// Create a new pipeline
let mut pipeline = reth_stages::Pipeline::new()

    // Push the HeaderStage into the pipeline
    .push(HeaderStage {
        downloader: 
        headers::reverse_headers::ReverseHeadersDownloaderBuilder::default()
            .batch_size(config.stages.headers.downloader_batch_size)
            .retries(config.stages.headers.downloader_retries)
            .build(consensus.clone(), fetch_client.clone()),
        consensus: consensus.clone(),
        client: fetch_client.clone(),
        network_handle: network.clone(),
        commit_threshold: config.stages.headers.commit_threshold,
        metrics: HeaderMetrics::default(),
    })

    // Push the BodyStage into the pipeline
    .push(BodyStage {
        downloader: Arc::new(
            bodies::bodies::BodiesDownloader::new(
                fetch_client.clone(),
                consensus.clone(),
            )
            .with_batch_size(config.stages.bodies.downloader_batch_size)
            .with_retries(config.stages.bodies.downloader_retries)
            .with_concurrency(config.stages.bodies.downloader_concurrency),
        ),
        consensus: consensus.clone(),
        commit_threshold: config.stages.bodies.commit_threshold,
    })

    // Push the SenderRecoveryStage into the pipeline
    .push(SenderRecoveryStage {
        commit_threshold: config.stages.sender_recovery.commit_threshold,
    })

    // Push the ExecutionStage into the pipeline
    .push(ExecutionStage { config: ExecutorConfig::new_ethereum() });

// Check a tip (latest block)
if let Some(tip) = self.tip {
    debug!("Tip manually set: {}", tip);

     // Notify the consensus mechanism of the fork choice state
    consensus.notify_fork_choice_state(ForkchoiceState {
        head_block_hash: tip,
        safe_block_hash: tip,
        finalized_block_hash: tip,
    })?;
}

// Run pipeline
info!("Starting pipeline");
pipeline.run(db.clone()).await?;
```
Now Let's Start the Network

File: bin/reth/src/node/mod.rs
``` Rust
// Start the Network
async fn start_network<C>(config: NetworkConfig<C>) -> Result<NetworkHandle, NetworkError>
where
    C: BlockReader + HeaderProvider + 'static,
{
    // Clone the network client
    let client = config.client.clone();
    // Set up the network manager and initialize the network components
    let (handle, network, _txpool, eth) =
        NetworkManager::builder(config).await?.request_handler(client).split_with_handle();

    // Network : Background Execution (Asynchronous)
    tokio::task::spawn(network);
    // TODO: tokio::task::spawn(txpool);
    // Ethereum protocol handler : Background Execution (Asynchronous)
    tokio::task::spawn(eth);

    // Return the network handle to control the network
    Ok(handle)
}
```

## Tasks
### Network Builder
네트워크 설정을 구성하는 빌더 패턴을 구현한 것입니다. 주요 구성 요소는 `NetworkBuilder` 구조체와 그 구현입니다. 이 코드는 네트워크 매니저, 트랜잭션 매니저, 그리고 요청 핸들러를 설정하고 관리하는 기능을 제공합니다.  

[File: crates/net/network/src/builder.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/builder.rs)

#### 1. 상수 정의
```Rust
pub(crate) const ETH_REQUEST_CHANNEL_CAPACITY: usize = 256;
```
- `ETH_REQUEST_CHANNEL_CAPACITY`는 `EthRequestHandler`의 최대 채널 용량을 256으로 설정합니다. 이는 악의적인 10MB 크기의 요청이 256개일 경우 2.6GB의 메모리를 차지할 수 있음을 의미합니다.

#### 2. `NetworkBuilder` 구조체 

```Rust
pub struct NetworkBuilder<Tx, Eth> {
    pub(crate) network: NetworkManager,
    pub(crate) transactions: Tx,
    pub(crate) request_handler: Eth,
}
```
- `NetworkBuilder`는 네트워크 매니저, 트랜잭션 매니저, 요청 핸들러를 포함하는 구조체입니다. 제네릭 타입 Tx와 Eth를 사용하여 다양한 트랜잭션 매니저와 요청 핸들러를 지원합니다.

#### 3.  `NetworkBuilder` 구현
```Rust
impl<Tx, Eth> NetworkBuilder<Tx, Eth> {
    // 여러 메서드들...
}
```

- **split**: 구조체를 분해하여 개별 필드를 반환합니다.
- **network**: 네트워크 매니저에 대한 참조를 반환합니다.
- **network_mut**: 네트워크 매니저에 대한 가변 참조를 반환합니다.
- handle: 네트워크 핸들을 반환합니다.
- **split_with_handle**: 구조체를 분해하여 네트워크 핸들과 개별 필드를 반환합니다.
- **transactions**: 새로운 - TransactionsManager를 생성하고 네트워크에 연결합니다.
- **equest_handler**: 새로운 EthRequestHandler를 생성하고 네트워크에 연결합니다.

#### 사용 예제 
```Rust
let network_manager = NetworkManager::new();
let builder = NetworkBuilder {
    network: network_manager,
    transactions: (),
    request_handler: (),
};

// 트랜잭션 매니저 설정
let pool = MyTransactionPool::new();
let transactions_manager_config = TransactionsManagerConfig::default();
let builder = builder.transactions(pool, transactions_manager_config);

// 요청 핸들러 설정
let client = MyClient::new();
let builder = builder.request_handler(client);

// 네트워크 매니저와 핸들 얻기
let (network_handle, network_manager, transactions, request_handler) = builder.split_with_handle();
```

### Network Config
네트워크 초기화 설정 지원 모듈  
[File: crates/net/network/src/config.rs](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/config.rs)

#### 1. `NetworkConfig` 구조체
- 네트워크 초기화에 필요한 모든 설정
- **client** : 체인과 상호작용하는 client
- **secret_key** : 노드의 비밀 키
- **boot_nodes**: 부트 노드의 집합; 네트워크 탐색 시 사용할 기본 노드들.
- **discovery_v4_config, dns_discovery_config**: 네트워크 디스커버리 설정.
- **peers_config, sessions_config**: 피어 및 세션 관리 설정
- **chain_spec**: 네트워크가 사용할 체인 스펙.
- **fork_filter**: 세션 인증에 사용할 포크 필터.
- **block_import**: 블록 가져오기를 처리하는 객체.
- **network_mode**: 네트워크가 사용 중인 모드(POS 또는 POW).
- **executor**: 네트워크 작업을 비동기로 처리할 실행기(executor).

#### 2. `NetworkConfigBuilder` 구조체
- `NetworkConfig` 빌드 위한 빌더 패턴 구현
- 단계별 설정 메서드 제공
- 기본값 설정 통한 초기화 편의 메서드 포함  
---  

**주요 메서드**  
- **new(secret_key)**: 새로운 빌더 인스턴스 생성 / 기본값 설정  
- **chain_spec, network_mode, set_head, hello_message** : 체인 스펙, 네트워크 모드, 헤드 정보, HelloMessage 설정  
- **set_addrs, listener_addr, discovery_addr** : 네트워크 리스너 및 디스커버리 주소 설정  
- **build(client)**: 설정을 기반으로 `NetworkConfig` 객체 생성.

#### 3. `NetworkMode` 열거형
- 네트워크 모드 정의   
: `Work` /  `Stake`
- `is_stake` 메서드로 모드 확인 가능

#### 4. 테스트 모듈 
- `NetworkConfig`와 `NetworkConfigBuilder` 기능 테스트
- `test_network_dns_defaults`, `test_network_fork_filter_default`

---