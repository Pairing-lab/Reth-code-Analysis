# `crates/storage/db` Analysis

## Overview

RETH 프로젝트의 `crates/storage/db` directory는 MDBX(Memory-Mapped Database eXtreme) database 위에 견고한 Abstraction Layer를 제공합니다. data 저장, 검색, tx 관리, 성능 metric 등 모든 low-level db interation을 관리합니다. EVM(Ethereum Virtual Machine)의 상태, 블록체인 data 등을 유지 관리하며, RETH 클라이언트에서 data integrity와 성능을 보장하는 데 필수적입니다.

### Role in the RETH Project

- **EVM State Storage**: account balances, Smart contract states, tx history 등 EVM State를 관리합니다.
- **Blockchain Data Management**: block header, receipts, tx 등 핵심 블록체인 data를 저장합니다.
- **Data Integrity and Performance**: tx를 통해 data consistency을 보장하고, caching과 metric을 사용하여 성능을 최적화합니다.

# MDBX(Memory-Mapped Database eXtreme)

![MDBX Performance](https://github.com/YoonHo-Chang/reth/blob/d56d32bdd6025f292ec5caa004cc654f7061944b/mdbx.png)


### MDBX main feature

MDBX는 매우 빠르고, 컴팩트하며, 강력한 embedded tx key-value database로, permissive license를 가지고 있습니다. 가벼운 솔루션을 만들기 위한 독특한 특성과 기능을 제공하도록 설계되었습니다. 다음은 MDBX database의 주요 특징입니다:

## Core Characteristics

- **Embedded**: MDBX는 애플리케이션 프로세스 내에서 실행되는 embedded database로, 별도의 서버로 실행되지 않습니다.
- **Transactional**: 완전한 ACID(Atomicity, Consistency, Isolation, Durability) tx를 지원합니다.
- **Key-Value Model**: 정보를 저장하기 위해 Key-Value data 모델을 사용합니다.

## Performance and Efficiency

- **Extremely fast**: MDBX는 고성능 작업을 위해 최적화되어 있습니다.
- **Memory-Mapped**: 전체 database가 메모리 맵으로 노출되어 `malloc`이나 `memcpy` 없이 직접 data 가져오기가 가능합니다.
- **B+ Tree Structure**: 효율적인 data 조직을 위해 B+ 트리를 사용하며, O(log N)의 연산 비용을 제공합니다.
- **Compact**: 라이브러리는 매우 가볍고, x86 아key텍처에서 핵심 바이너리 코드는 약 64KB에 불과합니다.

## Unique Features

- **Multi-Process Support**: 여러 process가 ACID 보장을 유지하면서 동시에 database를 읽고 업데이트할 수 있습니다.
- **No Maintenance Required**: Write-Ahead Log 또는 Append-Only 방식을 사용하는 database와 달리, MDBX는 주기적인 checkpointing이나 compaction이 필요하지 않습니다.
- **Automatic Compactification**: commit 시점에 지속적인 zero-overhead database compactification를 수행합니다.
- **Consistent Database Format:**: 포맷은 endianness에만 의존하며, 32비트와 64비트 시스템 간에 일관성을 유지합니다<sup>1,2</sup>.

## Advanced Capabilities

- **Large Value Support**: 최대 1기가바이트 크기의 value을 저장할 수 있습니다.
- **Multiple Named Databases**: 하나의 환경 내에서 multiple named databases (key-value 공간)를 지원합니다.
- **Nested tx**: 중첩된 write tx을 허용합니다.
- **Cursors**: 더 복잡한 data 작업을 위한 cursor 기능을 제공합니다.
- **Duplicate Keys**: `MDBX_DUPSORT` 플래그를 통해 single key에 대한 multiple values을 지원합니다.

## Additional Features

- **Integrity Checking**: database integrity 검사를 위한 `mdbx_chk` 유틸리티를 포함합니다.
- **Extended Information**: database, sub-databases, tx 및 reader에 대한 상세 정보를 제공합니다.
- **Flexible Opening Modes**: 네트워크 공유를 포함하여 exclusive mode로 database를 열 수 있습니다.
- **Zero-Length Keys and Values**: 길이가 0인 key와 value을 허용합니다.
- **Sequence Generation**: built-in sequence 생성과 지속적인 64비트 마커를 제공합니다.

## Module Analysis

### `crates/storage/db/srcs/lib.rs`

#### Summary

Database Abstraction layer의 MDBX 구현을 제공합니다. 주로 **테스트 유틸리티**를 포함하며, 임시 database를 생성하고 삭제하는 기능을 제공합니다. 또한 database의 버전 관리 및 클라이언트 버전 기록을 위한 테스트도 포함되어 있습니다.

#### Example Usage

```rust
use crate::test_utils::create_test_rw_db;

let db = create_test_rw_db();
let tx = db.tx().unwrap();
// database tx을 사용하여 data 조작
```

#### Code Analysis

##### Inputs

- **database 경로**: database가 저장될 파일 시스템 경로.
- **`DatabaseArguments` 객체**: database 초기화를 위한 설정 파라미터.

##### Flow

1. **`TempDatabase` struct**
   - 임시 database를 생성하고 삭제하는 기능을 제공합니다.
2. **`create_test_rw_db` 함수**
   - read/write가 가능한 테스트 database를 생성합니다.
3. **`db_version` 테스트**
   - database 버전 파일의 유효성을 검사하여 호환성을 확인합니다.
4. **`db_client_version` 테스트**
   - 클라이언트 버전 기록을 확인하여 일관성을 보장합니다.

##### Outputs

- **임시 database 객체**: 메모리 또는 디스크에 생성된 database 인스턴스.
- **database tx 객체**: database에서 원자적 read/write 작업을 수행하기 위한 객체.

#### Dependencies

- **MDBX 라이브러리**
- **Rust 표준 라이브러리**
- **내부 module**: `crate::common`, `crate::utils` 

---

### `crates/storage/db/srcs/lockfile.rs`

#### Summary

`StorageLock` struct를 정의하여 파일 잠금을 통해 저장소 디렉토리에 대한 독점적인 read-write 접근을 보장합니다. 프로세스의 PID를 저장하고 정상 종료 시 잠금을 해제합니다. 충돌 후 재개 시 저장된 PID를 사용하여 다른 프로세스가 잠금을 보유하고 있는지 확인합니다.

#### Example Usage

```rust
use std::path::Path;
use your_crate::StorageLock;

fn main() {
    let path = Path::new("/path/to/storage");
    match StorageLock::try_acquire(path) {
        Ok(lock) => println!("Lock acquired successfully."),
        Err(err) => println!("Failed to acquire lock: {:?}", err),
    }
}
```

#### Code Analysis

##### Inputs

- `path`: 잠금을 시도할 디렉토리의 경로를 나타내는 `&Path` type.

##### Flow

1. `try_acquire` method는 지정된 경로에 대한 파일 잠금을 시도합니다.
2. 잠금 파일이 이미 존재하고 다른 활성 프로세스가 보유하고 있는 경우 오류를 return합니다.
3. 그렇지 않으면 현재 프로세스의 PID를 포함하는 새로운 잠금 파일을 생성합니다.
4. `Drop` 구현은 `StorageLock`이 해제될 때 잠금 파일을 삭제합니다.

##### Outputs

- 성공 시 `StorageLock` 인스턴스를 return합니다.
- 실패 시 `StorageLockError`를 return합니다.

#### Key Concepts

- **파일 잠금 메커니즘**: database에 대한 읽기-쓰기 접근을 단일 프로세스로 제한합니다.
- **PID 기반 잠금 관리**: 프로세스 ID를 사용하여 프로세스 간의 잠금 상태를 추적하고 관리합니다.

#### Dependencies

- **표준 라이브러리**: 파일 시스템 작업과 오류 처리를 위해 Rust의 표준 라이브러리를 사용합니다.

---

### `crates/storage/db/srcs/mdbx.rs`

#### Summary

Rust에서 MDBX database를 생성, 초기화 및 열기 위한 함수를 정의합니다. `create_db` 함수는 지정된 경로에 database를 생성하고, `init_db`는 database를 초기화하며, `open_db_read_only`와 `open_db`는 각각 read only 및 read/write 모드로 database를 엽니다.

#### Example Usage

```rust
use std::path::Path;
use crate::DatabaseArguments;

let path = Path::new("/path/to/database");
let args = DatabaseArguments::new();

// database 생성 및 초기화
let db_env = init_db(path, args).expect("Failed to initialize database");

// read only로 database 열기
let db_env_ro = open_db_read_only(path, args).expect("Failed to open database read-only");

// read/write 모드로 database 열기
let db_env_rw = open_db(path, args).expect("Failed to open database read/write");
```

#### Code Analysis

##### Inputs

- `path`: database가 생성되거나 열릴 경로를 나타내는 `Path` 객체.
- `args`: database 환경 설정을 위한 `DatabaseArguments` 객체.

##### Flow

1. **`create_db` 함수**
   - database가 비어 있는지 확인합니다.
   - 비어 있다면 디렉토리를 생성하고 버전 파일을 만듭니다.
2. **`init_db` 함수**
   - `create_db`를 호출하여 database를 생성합니다.
   - 테이블을 생성하고 클라이언트 버전을 기록합니다.
3. **`open_db_read_only` 함수**
   - 지정된 경로에서 database를 read only 모드로 엽니다.
4. **`open_db` 함수**
   - 지정된 경로에서 database를 read/write 모드로 열고 클라이언트 버전을 기록합니다.

##### Outputs

- 성공적으로 database 환경을 생성하거나 열면 `DatabaseEnv` 객체를 return합니다.
- 오류가 발생하면 `eyre::Result`를 통해 에러를 return합니다.

#### Key Concepts

- **database 초기화**
- **read only과 read/write 모드**

#### Dependencies

- **MDBX 라이브러리**: database 작업을 위해 사용됩니다.
- **`DatabaseArguments`**: 구성 설정을 위한 커스텀 struct.
- **에러 처리**: `eyre` crate를 사용하여 에러를 관리합니다.

---

### `crates/storage/db/srcs/metrics.rs`

#### Summary

database 환경에서 metric handle을 caching하여 각 작업 시마다 재생성되지 않도록 하는 `DatabaseEnvMetrics` struct를 정의합니다. 이 struct는 database 작업, tx 모드, tx 결과에 대한 metric을 기록하고 관리합니다.

#### Example Usage

```rust
let db_metrics = DatabaseEnvMetrics::new();
db_metrics.record_opened_transaction(TransactionMode::ReadOnly);
db_metrics.record_operation("users", Operation::Get, Some(5000), || {
    // database에서 사용자 정보를 가져오는 작업
});
```

#### Code Analysis

##### Inputs

- `table`: database 테이블의 이름 (`&'static str` type).
- `operation`: 수행할 database 작업의 종류를 나타내는 `Operation` enum.
- `value_size`: 작업의 value 크기를 나타내는 `Option<usize>` type.
- `mode`: tx 모드를 나타내는 `TransactionMode` enum.
- `outcome`: tx 결과를 나타내는 `TransactionOutcome` enum.
- `open_duration`: tx이 열린 시간 (`Duration` type).
- `close_duration`: tx이 닫히는 데 걸린 시간 (`Option<Duration>` type).
- `commit_latency`: commit 지연 시간을 나타내는 `Option<reth_libmdbx::CommitLatency>` type.

##### Flow

1. **초기화**
   - `DatabaseEnvMetrics` struct는 모든 가능한 조합의 metric handle을 미리 생성하여 초기화합니다.
2. **작업 기록**
   - `record_operation`은 주어진 테이블과 작업에 대한 metric을 기록합니다.
   - `record_opened_transaction`와 `record_closed_transaction`은 tx의 수명 주기를 추적하고 기록합니다.

##### Outputs

- metric이 기록되고 성능 분석을 위해 저장됩니다.

#### Key Concepts

- **metric caching**: metric 핸들의 재생성을 피하여 성능 향상.
- **작업 metric 기록**: database 작업의 성능 특성을 이해하는 데 도움을 줍니다.

#### Dependencies

- **metric 라이브러리**: 성능 data를 기록하고 관리합니다.

---

### `crates/storage/db/srcs/utils.rs`

#### Summary

운영 체제에서 사용할 수 있는 기본 페이지 크기를 반환하는 함수와 database가 비어 있는지 확인하는 함수를 포함하고 있습니다.

#### Example Usage

```rust
let page_size = default_page_size();
println!("Default page size: {}", page_size);

let is_empty = is_database_empty("/path/to/db");
println!("Is database empty: {}", is_empty);
```

#### Code Analysis

##### Inputs

- `default_page_size`
- `is_database_empty`: database 경로를 나타내는 generic 파라미터 `P`.

##### Flow

1. **`default_page_size` 함수**
   - 운영 체제의 페이지 크기를 가져옵니다.
   - 최소 및 최대 페이지 크기 사이로 value을 제한합니다.
2. **`is_database_empty` 함수**
   - 주어진 경로가 존재하는지 확인합니다.
   - 경로가 디렉토리이고 비어 있는지 판단합니다.

##### Outputs

- `default_page_size`: 운영 체제의 기본 페이지 크기를 return합니다.
- `is_database_empty`: database가 비어 있는지 여부를 나타내는 불리언 value을 return합니다.

#### Key Concepts

- **페이지 크기의 중요성**: 운영 체제의 페이지 크기가 database 성능과 리소스 활용에 영향을 미칩니다.
- **database 상태 검증**: 유효한 database 상태에서 작업이 수행되도록 보장합니다.

#### Dependencies

- **표준 라이브러리**: 시스템 쿼리 및 파일 시스템 작업을 위해 사용됩니다.

---

### `crates/storage/db/srcs/version.rs`

#### Summary

database **버전 파일을 확인하고 관리**하는 유틸리티 함수들을 제공합니다. database 버전 파일의 경로를 생성하고, 파일에서 버전을 읽어 현재 버전과 비교하며, 필요 시 파일을 생성하거나 업데이트합니다.

#### Example Usage

```rust
use std::path::Path;
use my_crate::check_db_version_file;

let db_path = Path::new("/path/to/database");
match check_db_version_file(db_path) {
    Ok(_) => println!("Database version is up-to-date."),
    Err(e) => println!("Error: {:?}", e),
}
```

#### Code Analysis

##### Inputs

- `db_path`: database 버전 파일의 경로를 나타내는 `Path` type의 입력.

##### Flow

1. **버전 확인**
   - `check_db_version_file` 함수는 `get_db_version`을 호출하여 파일에서 버전을 읽습니다.
   - 읽은 버전을 현재 버전과 비교합니다.
2. **에러 처리**
   - 버전이 일치하지 않으면 `VersionMismatch` 오류를 return합니다.

##### Outputs

- 성공 시 `Ok(())`를 return합니다.
- 실패 시 `DatabaseVersionError`의 다양한 변형을 return합니다.

#### Key Concepts

- **버전 호환성**: database schema가 클라이언트와 호환되는지 확인합니다.
- **에러 처리**: 버전 불일치 및 파일 I/O 오류를 적절히 관리합니다.

#### Dependencies

- **표준 라이브러리**: 파일 작업과 오류 처리를 위해 사용됩니다.

---

### `crates/storage/db/srcs/implementation/mdbx/cursor.rs`

#### Summary

`libmdbx` 라이브러리를 사용하는 database cursor의 래퍼를 정의합니다. 이 래퍼는 read only 및 읽기-쓰기 cursor를 제공하며, database에서 key-value 쌍을 검색, 삽입, 삭제하는 기능을 포함합니다. 또한 중복 key를 처리하는 기능도 포함되어 있습니다.

#### Example Usage

```rust
// read only cursor 생성
let ro_cursor: CursorRO<MyTable> = Cursor::new_with_metrics(inner_cursor, Some(metrics));

// 첫 번째 key-value 쌍 검색
let first_pair = ro_cursor.first();

// 특정 key로 검색
let specific_pair = ro_cursor.seek_exact(my_key);

// 읽기-쓰기 cursor 생성
let mut rw_cursor: CursorRW<MyTable> = Cursor::new_with_metrics(inner_cursor, Some(metrics));

// key-value 쌍 삽입
rw_cursor.insert(my_key, my_value).unwrap();

// 현재 위치의 key-value 쌍 삭제
rw_cursor.delete_current().unwrap();
```

#### Code Analysis

##### Inputs

- `inner`: `libmdbx` cursor 객체.
- `metrics`: database 환경 metric에 대한 선택적 참조.
- `key`: 테이블의 key.
- `value`: 테이블의 value.

##### Flow

1. **cursor 래핑**
   - `Cursor` struct는 `libmdbx` cursor를 래핑하여 database 작업을 수행합니다.
2. **decoding**
   - `decode` 함수는 database에서 가져온 `(key, value)` 쌍을 decoding합니다.
3. **trait 구현**
   - `DbCursorRO` 및 `DbCursorRW` trait를 구현하여 read only 및 읽기-쓰기 작업을 제공합니다.
4. **작업 metric**
   - `execute_with_operation_metric` method는 metric을 기록하거나 클로저를 실행합니다.

##### Outputs

- database에서 검색된 `(key, value)` 쌍 또는 오류.
- database에 대한 삽입, 삭제 작업의 성공 또는 오류.

#### Key Concepts

- **cursor 작업**: cursor를 사용하여 database 항목을 탐색하고 조작하는 방법.
- **중복 key 처리**: 동일한 key에 대해 여러 value을 관리합니다.

#### Dependencies

- **`reth_libmdbx`**: MDBX cursor 작업을 위해 사용됩니다.
- **metric 라이브러리**: 작업 metric을 기록하기 위해 사용됩니다.

---

### `crates/storage/db/srcs/implementation/mod.rs`

#### Summary

Rust로 작성된 MDBX database의 **Abstraction Layer를 구현**합니다. database 환경 설정, tx 관리, cursor 기반의 data 순회 및 metric 기록과 같은 기능을 제공합니다. 특히 테스트 환경을 위한 유틸리티도 포함되어 있으며, database 버전 관리와 클라이언트 버전 기록을 처리합니다.

#### Key Features

##### **`DatabaseEnvKind` enum**

- **역할**: database 환경이 read only(RO)인지, read/write(RW)인지 구분합니다.
- **주요 method**:
  - `is_rw`: 현재 환경이 read/write 가능한지 여부를 return합니다.

##### **`DatabaseArguments` struct**

- **역할**: database 초기화 시 필요한 설정 인자를 관리합니다.
- **주요 필드**:
  - `client_version`: database에 접근하는 클라이언트 버전을 관리합니다.
  - `log_level`: 로그 레벨 설정 (선택적).
  - `max_read_transaction_duration`: 읽기 tx의 최대 지속 시간을 설정 (선택적).
  - `exclusive`: 독점 모드로 database를 열지 여부를 설정합니다.
- **주요 method**:
  - `new`: 주어진 클라이언트 버전을 사용하여 새로운 `DatabaseArguments` 객체를 생성합니다.
  - `with_log_level`, `with_max_read_transaction_duration`, `with_exclusive`: 각 필드를 설정하는 method.

##### **`DatabaseEnv` struct**

- **역할**: MDBX 환경을 관리하며, read/write tx을 처리합니다.
- **주요 필드**:
  - `inner`: 실제 MDBX 환경(`Environment`)을 나타냅니다.
  - `metrics`: database metric을 기록하는 캐시입니다.
  - `_lock_file`: read/write 환경에서 사용되는 잠금 파일입니다.
- **주요 method**:
  - `open`: 주어진 경로와 설정을 기반으로 MDBX 환경을 열고 설정을 적용합니다.
  - `tx`: read only tx을 return합니다.
  - `tx_mut`: read/write tx을 return합니다.
  - `create_tables`: database에 필요한 테이블을 생성합니다.
  - `record_client_version`: database 호환성을 관리하기 위해 클라이언트 버전을 기록합니다.

##### **tx 인터페이스**

- `Database` trait를 구현하여 tx 기능을 제공합니다.

#### CodeFlow

1. **MDBX 환경 초기화**
   - `DatabaseEnv::open` method를 사용하여 database 환경을 초기화합니다.
2. **tx 처리**
   - `tx`로 read only tx을 시작하고 `tx_mut`로 read/write tx을 시작합니다.
3. **cursor 작업**
   - `Cursor` module을 사용하여 data 순회 및 조작을 수행합니다.
4. **metric 기록**
   - `report_metrics` method를 통해 다양한 metric을 기록합니다.
5. **테스트 유틸리티**
   - `test_utils` module을 사용하여 테스트 환경에서 임시 database를 생성하고 관리합니다.

#### Key Concepts

- **환경 종류**: 환경 type(RO vs. RW)에 따라 동작을 구분합니다.
- **metric 및 성능 모니터링**: 성능 인사이트를 위해 metric을 기록하고 활용합니다.
- **테스트 지원**: 테스트 및 검증을 위한 유틸리티를 제공합니다.

#### Dependencies

- **`reth_libmdbx`**: MDBX 환경 관리를 위해 사용됩니다.
- **`reth_db_api`**: database tx 및 cursor trait를 제공합니다.
- **metric 라이브러리**: 성능 data를 기록합니다.
- **`eyre`**: 향상된 에러 처리를 위해 사용됩니다.

---

### `crates/storage/db/srcs/implementation/mdbx/tx.rs`

#### Summary

`libmdbx-sys` 라이브러리를 사용하는 tx 래퍼를 정의합니다. `Tx` struct는 database tx을 관리하며, metric을 기록하고 tx의 수명 주기를 추적합니다. 또한 tx의 commit, 중단, database 핸들 가져오기, cursor 생성 등의 기능을 제공합니다.

#### Example Usage

```rust
use reth_libmdbx::{Transaction, RW};
use std::sync::Arc;
use crate::metrics::DatabaseEnvMetrics;

// tx 생성
let transaction = Transaction::<RW>::new();
let env_metrics = Arc::new(DatabaseEnvMetrics::new());
let tx = Tx::new_with_metrics(transaction, Some(env_metrics)).unwrap();

// database 핸들 가져오기
let dbi = tx.get_dbi::<SomeTable>().unwrap();

// cursor 생성
let cursor = tx.new_cursor::<SomeTable>().unwrap();

// tx commit
let commit_result = tx.commit();
```

#### Code Analysis

##### Inputs

- `inner`: `libmdbx-sys` tx 객체.
- `env_metrics`: 선택적 metric 핸들러.

##### Flow

1. **tx 래핑**
   - `Tx` struct는 `libmdbx-sys` tx을 래핑합니다.
2. **metric 처리**
   - tx 생성 시 metric 핸들러를 설정할 수 있습니다.
3. **tx 작업**
   - tx ID를 가져오거나 database handle을 생성하고, cursor를 관리합니다.
4. **수명 주기 추적**
   - metric 핸들러는 tx의 수명 주기를 추적하고 기록합니다.

##### Outputs

- tx ID, database 핸들, cursor 객체, tx commit 결과 등을 return합니다.

#### Key Concepts

- **tx 관리**: tx의 시작, commit, 중단을 처리합니다.
- **metric 통합**: 성능 분석을 위해 tx metric을 기록합니다.

#### Dependencies

- **`reth_libmdbx`**: tx 작업을 위해 사용됩니다.
- **metric 라이브러리**: tx metric을 기록합니다.

---

### `crates/storage/db/srcs/static_file/mod.rs`

#### Summary

주어진 디렉토리 경로에서 정적 파일을 읽고, 각 파일을 `StaticFileSegment`에 따라 정렬된 블록 및 tx 범위 목록으로 반환하는 기능을 제공합니다. cursor 구현과 선택적 data 읽기를 위한 마스크 정의를 포함합니다.

#### Key Features

- **정적 파일 반복**
  - `iter_static_files` 함수는 정적 파일을 읽고 세그먼트별로 조직합니다.
- **정적 파일 cursor**
  - `StaticFileCursor`는 정적 파일 세그먼트에서 data를 탐색하고 검색할 수 있습니다.
- **마스크 정의**
  - 특정 열 value을 선택하고 압축 해제하기 위한 마스크가 정의됩니다.

#### Code Analysis

##### Flow

1. **파일 읽기**
   - 제공된 경로가 존재하지 않으면 디렉토리를 생성합니다.
   - 디렉토리의 모든 파일을 읽고, 파일 이름을 `StaticFileSegment`로 파싱합니다.
2. **data 조직**
   - 각 파일의 `NippyJar`를 로드하고 블록 및 tx 범위를 추출합니다.
   - data를 `SortedStaticFiles` HashMap으로 조직합니다.
3. **cursor 작업**
   - `StaticFileCursor`는 `get`, `get_one`, `get_two`, `get_three` 등의 method를 제공하여 key나 번호에 따라 data를 검색합니다.
4. **마스크 사용**
   - 마스크를 사용하여 특정 열을 선택하고 data 압축 해제를 수행합니다.

#### Key Concepts

- **data 세그먼트화**: 효율적인 접근을 위해 data를 세그먼트로 조직합니다.
- **선택적 data 읽기**: 필요한 data 열만 읽기 위해 마스크를 사용합니다.
- **cursor 탐색**: 정적 파일 data를 효율적으로 순회합니다.

#### Dependencies

- **`reth_nippy_jar`**: 압축된 data 세그먼트를 처리하기 위해 사용됩니다.
- **표준 라이브러리**: 파일 시스템 작업을 위해 사용됩니다.

---

### `crates/storage/db/srcs/static_file/cursor.rs`

#### Summary

`StaticFileCursor`라는 struct와 관련된 method를 정의합니다. 이 struct는 정적 파일 세그먼트의 커서 역할을 하며, 특정 key 또는 번호에 따라 data를 검색하고, 여러 열의 value을 가져오는 기능을 제공합니다.

#### Example Usage

```rust
use std::sync::Arc;
use reth_nippy_jar::{DataReader, NippyJar};
use reth_primitives::static_file::SegmentHeader;

let jar: NippyJar<SegmentHeader> = // ... 초기화 코드
let reader: Arc<DataReader> = // ... 초기화 코드
let cursor = StaticFileCursor::new(&jar, reader).unwrap();

let key_or_num = KeyOrNumber::Number(42);
if let Some(number) = cursor.number() {
    println!("Current number: {}", number);
}

if let Some(values) = cursor.get(key_or_num, 0).unwrap() {
    println!("Row values: {:?}", values);
}
```

#### Code Analysis

##### Inputs

- **`jar`**: `NippyJar<SegmentHeader>` type의 참조로, 정적 파일 세그먼트를 나타냅니다.
- **`reader`**: `Arc<DataReader>` type으로, data 읽기를 위한 객체입니다.
- **`key_or_num`**: `KeyOrNumber` type으로, key 또는 번호를 나타냅니다.
- **`mask`**: `usize` type으로, 열 선택을 위한 마스크입니다.

##### Flow

1. **`StaticFileCursor` struct 초기화**
   - `NippyJar`와 `DataReader`를 사용하여 새로운 `StaticFileCursor`를 생성합니다.
2. **data 검색**
   - `number` method를 통해 현재 커서의 블록 또는 트랜잭션 번호를 가져옵니다.
   - `get` method를 사용하여 주어진 key 또는 번호에 해당하는 행의 value을 가져옵니다.
3. **선택적 열 가져오기**
   - `get_one`, `get_two`, `get_three` method를 통해 각각 하나, 두 개, 세 개의 열 value을 선택적으로 가져올 수 있습니다.

##### Outputs

- **`StaticFileCursor::new`**: `ProviderResult<Self>` type의 결과를 반환합니다.
- **`number`**: `Option<u64>` type의 번호를 반환합니다.
- **`get`, `get_one`, `get_two`, `get_three`**: `ProviderResult<Option<T>>` type의 결과를 반환합니다.

#### Key Concepts

- **정적 파일 커서**: `StaticFileCursor`는 정적 파일 세그먼트 내의 data를 탐색하고 검색하기 위한 커서로, database 커서와 유사한 방식으로 동작합니다.
- **key 또는 번호 기반 검색**: `KeyOrNumber`를 사용하여 특정 key나 번호에 따라 data를 효율적으로 검색합니다.
- **NippyJar 통합**: `NippyJar` 라이브러리와 통합되어 압축된 data 세그먼트를 효율적으로 읽고 처리합니다.
- **동시성 및 공유 data 접근**: `Arc<DataReader>`를 통해 data 리더를 여러 스레드에서 안전하게 공유하여 동시성을 지원합니다.
- **선택적 data 열 가져오기**: 마스크와 `get_one`, `get_two`, `get_three` method를 사용하여 필요한 data 열만 선택적으로 가져옴으로써 메모리 및 성능 효율성을 높입니다.

#### Dependencies

- **`reth_nippy_jar`**: 압축된 data 세그먼트를 처리하기 위한 라이브러리입니다.
- **`reth_primitives`**: `SegmentHeader`와 같은 기본 type을 제공합니다.
- **`std::sync::Arc`**: 스레드 안전한 공유 포인터로, data 리더의 동시 접근을 가능하게 합니다.

---

### `crates/storage/db/srcs/static_file/mask.rs`

#### Summary

특정 열 value을 선택하고 압축 해제하기 위한 마스크를 정의하는 struct와 매크로를 제공합니다. `Mask` struct는 열 선택을 위한 마스크를 나타내며, `add_segments!` 매크로는 다양한 정적 파일 세그먼트에 대한 마스크 struct를 생성합니다. `ColumnSelectorOne`, `ColumnSelectorTwo`, `ColumnSelectorThree` trait는 각각 하나, 두 개, 세 개의 열 value을 선택하기 위한 마스크를 정의합니다.

#### Example Usage

```rust
// Header 세그먼트에서 첫 번째 열을 선택하기 위한 마스크 정의
add_static_file_mask!(HeaderMask, B256, 0b001);

// B256 type으로 HeaderMask에 대해 ColumnSelectorOne 구현
impl ColumnSelectorOne for HeaderMask<B256> {
    type FIRST = B256;
    const MASK: usize = 0b001;
}
```

#### Code Analysis

##### Inputs

- **`FIRST`, `SECOND`, `THIRD`**: 선택할 열의 type을 나타내는 generic type 매개변수.
- **`segment`**: 정적 파일 세그먼트의 이름.
- **`mask`**: 선택할 열을 나타내는 비트 마스크.

##### Flow

1. **Mask struct 정의**
   - `Mask` struct는 선택할 열을 나타내는 마스크를 정의합니다.
2. **매크로를 통한 마스크 생성**
   - `add_segments!` 매크로는 주어진 세그먼트에 대한 마스크 struct를 생성합니다.
3. **ColumnSelector trait 구현**
   - `ColumnSelectorOne`, `ColumnSelectorTwo`, `ColumnSelectorThree` trait는 각각 하나, 두 개, 세 개의 열을 선택하기 위한 마스크를 정의합니다.
4. **add_static_file_mask! 매크로 사용**
   - 특정 세그먼트에 대한 마스크를 정의하고 해당 trait를 구현합니다.

##### Outputs

- 특정 열을 선택하고 압축 해제하기 위한 마스크 struct와 trait 구현이 생성됩니다.

#### Key Concepts

- **열 선택을 위한 마스크 사용**: data의 특정 열만 선택적으로 읽거나 압축 해제하기 위해 비트 마스크를 사용합니다. 이는 불필요한 data 처리를 줄여 성능을 향상시킵니다.

- **매크로를 통한 코드 생성**: `add_segments!`와 `add_static_file_mask!` 매크로를 사용하여 반복적인 코드 작성을 피하고, 각 세그먼트에 대한 마스크 struct와 trait 구현을 자동으로 생성합니다.

- **ColumnSelector trait**: `ColumnSelectorOne`, `ColumnSelectorTwo`, `ColumnSelectorThree` trait는 선택할 열의 수에 따라 구현되며, 각 열의 type과 마스크 value을 정의합니다. 이를 통해 type 안전성과 코드 재사용성을 높입니다.

- **generic programming**: 열의 type을 generic으로 정의하여 다양한 data type에 대해 유연하게 마스크를 적용할 수 있습니다.

- **비트 마스크의 활용**: 비트 마스크를 사용하여 어떤 열을 선택할지 결정하며, 각 비트는 특정 열의 선택 여부를 나타냅니다. 예를 들어, `0b001`은 첫 번째 열을 선택한다는 것을 의미합니다.

#### Dependencies

- **Rust 표준 라이브러리**: 매크로와 generic type을 사용하기 위한 기본 기능을 제공합니다.
- **기본 type 및 trait**: `B256` 등의 기본 type과 `ColumnSelector` trait는 프로젝트 내에서 정의된 것들입니다.

---

### `crates/storage/db/srcs/static_file/masks.rs`

#### Summary

`add_static_file_mask!` 매크로를 사용하여 다양한 data type에 대한 마스크를 정의합니다. 이러한 마스크는 `HeaderMask`, `ReceiptMask`, `TransactionMask`와 같은 struct에 적용되며, 각 data type에 대해 특정 bit pattern을 할당합니다. 이를 통해 정적 파일에서 필요한 data 열을 효율적으로 선택하고 처리할 수 있습니다.

#### Example Usage

```rust
// HeaderMask에 대한 마스크 추가
add_static_file_mask!(HeaderMask, Header, 0b001);

// ReceiptMask에 대한 마스크 추가
add_static_file_mask!(ReceiptMask, <Receipts as Table>::Value, 0b1);

// TransactionMask에 대한 마스크 추가
add_static_file_mask!(TransactionMask, <Transactions as Table>::Value, 0b1);
```

#### Code Analysis

##### Inputs

- **`HeaderMask`, `ReceiptMask`, `TransactionMask`**: 마스크를 적용할 struct의 이름입니다.
- **data type**: 마스크를 적용할 data type으로, 각 마스크가 처리할 data의 유형을 나타냅니다.
- **bit pattern**: 각 data type에 할당할 bit pattern으로, 선택할 열을 지정합니다.

##### Flow

1. **마스크 정의를 위한 매크로 호출**
   - `add_static_file_mask!` 매크로를 사용하여 마스크를 정의합니다.
2. **HeaderMask에 대한 마스크 설정**
   - `HeaderMask`에 여러 data type과 bit pattern을 지정하여 필요한 열을 선택합니다.
3. **ReceiptMask와 TransactionMask 설정**
   - `ReceiptMask`와 `TransactionMask`에 각각의 data type과 bit pattern을 지정하여 마스크를 정의합니다.

##### Outputs

- 정의된 마스크는 각 struct와 data type에 대해 특정 bit pattern을 할당합니다.
- 결과적으로, 정적 파일에서 필요한 data 열을 선택하고 처리하기 위한 마스크 struct와 trait 구현이 생성됩니다.

#### Key Concepts

- **매크로를 통한 마스크 생성 자동화**: `add_static_file_mask!` 매크로를 사용하여 반복적인 마스크 정의 작업을 자동화합니다. 이는 코드의 가독성과 유지보수성을 향상시킵니다.

- **data type별 마스크 적용**: 각 마스크는 특정 data type에 대해 정의되며, 해당 type의 data를 처리할 때 필요한 열을 선택적으로 가져올 수 있습니다.

- **bit pattern을 통한 열 선택**: bit pattern은 어떤 열을 선택할지 결정하는데 사용됩니다. 예를 들어, `0b001`은 첫 번째 열을 선택하고, `0b1`은 첫 번째 열을 선택한다는 것을 의미합니다.

- **유연한 data 처리**: 이러한 마스크를 통해 정적 파일에서 필요한 data 열만 선택적으로 읽을 수 있으므로, 메모리 사용량을 줄이고 성능을 향상시킬 수 있습니다.

- **type 안전성과 재사용성**: 마스크와 data type을 generic으로 정의하여 다양한 data 구조에 대해 type 안전한 방식으로 마스크를 적용할 수 있습니다.

#### Dependencies

- **프로젝트 내부 module**: `add_static_file_mask!` 매크로와 관련된 기능은 프로젝트 내 다른 module에서 정의되어 있습니다.
- **Rust 표준 라이브러리**: 매크로 및 generic programming을 지원하기 위한 기본 기능을 제공합니다.
- **기본 data type 및 trait**: `Header`, `Receipts`, `Transactions` 등의 type과 `Table` trait는 프로젝트 내에서 정의된 것들입니다.

---

### `crates/storage/db/srcs/tables/utils.rs`

#### Summary

작은 database 테이블 유틸리티와 헬퍼 함수들을 정의합니다. 주어진 `(key, value)` 쌍을 decoding하거나, value만 decoding하는 함수들을 제공합니다. 이를 통해 database에 저장된 binary data를 애플리케이션에서 사용할 수 있는 형식으로 변환합니다.

#### Example Usage

```rust
use std::borrow::Cow;
use crate::decoder;
use reth_db_api::table::TableRow;

// 가상의 Table type과 DatabaseError type이 정의되어 있다고 가정합니다.
let key_value_pair = (Cow::Borrowed(b"key"), Cow::Borrowed(b"value"));
let result: Result<TableRow<MyTable>, DatabaseError> = decoder::<MyTable>(key_value_pair);

match result {
    Ok(decoded_row) => println!("Decoded successfully: {:?}", decoded_row),
    Err(e) => println!("Error decoding: {:?}", e),
}
```

#### Code Analysis

##### Inputs

- **`kv`**: `(Cow<'a, [u8]>, Cow<'a, [u8]>)` 형태의 key와 value 쌍입니다. `Cow`는 Copy-On-Write를 의미하며, data의 ownership을 효율적으로 관리합니다.

##### Flow

1. **`decoder` 함수 호출**
   - `(key, value)` 쌍을 받아 각각 decoding 및 decompression을 수행합니다.
2. **key decoding**
   - key는 테이블의 key type에 맞게 decoding됩니다.
3. **value decompression 및 decoding**
   - value은 압축된 경우 decompression하고, 테이블의 value type에 맞게 decoding합니다.
4. **결과 반환**
   - 성공 시 decoding된 결과를 `TableRow<T>` 형태로 반환합니다.
   - 실패 시 `DatabaseError`를 반환합니다.

##### Outputs

- **`decoder`**: `Result<TableRow<T>, DatabaseError>` 형태의 decoding된 테이블 행을 반환합니다.
- **`decode_value`**: `Result<T::Value, DatabaseError>` 형태의 decompression된 value을 반환합니다.
- **`decode_one`**: `Result<T::Value, DatabaseError>` 형태의 decompression된 value을 반환합니다.

#### Key Concepts

- **data decoding 및 decompression**: database에 저장된 binary data는 직렬화되고, 저장 공간 절약을 위해 압축됩니다. 이러한 data를 애플리케이션에서 사용할 수 있는 형태로 복원하는 기능을 제공합니다.

- **유연한 data 처리**: `Cow<'a, [u8]>`를 사용하여 data의 ownership을 효율적으로 관리합니다. data가 변경되지 않을 경우 복사를 피하고, 필요한 경우에만 복사하여 성능을 최적화합니다.

- **에러 처리와 안정성**: decoding 과정에서 발생할 수 있는 오류를 `Result` type으로 처리하여 안정성을 높입니다. 오류 발생 시 명확한 `DatabaseError`를 반환하여 debugging과 예외 처리를 용이하게 합니다.

- **generic programming 활용**: 함수들은 generic type `T`를 사용하며, 이는 `Table` trait를 구현해야 합니다. 이를 통해 다양한 테이블 type에 대해 동일한 decoding 로직을 적용할 수 있습니다.

- **압축된 value 처리**: value이 압축된 경우 이를 자동으로 decompression하여 애플리케이션에서 사용할 수 있도록 합니다. 이는 저장 공간을 절약하면서도 data 접근을 효율적으로 만듭니다.

#### Dependencies

- **`reth_db_api`**: database 테이블과 행을 나타내는 `Table` 및 `TableRow` trait를 제공합니다.
- **Rust 표준 라이브러리**: `std::borrow::Cow` 및 기타 기본 type과 기능을 사용합니다.
- **프로젝트 내부 module**: `decoder`, `decode_value`, `decode_one` 함수들은 프로젝트 내 다른 module에서 정의된 직렬화 및 압축 로직을 사용합니다.

---

### `crates/storage/db/srcs/tables/raw.rs`

#### Summary

`RawTable` 및 `RawDupSort` struct를 정의하여 database 테이블의 key와 value을 지연된 decoding/인코딩 방식으로 처리할 수 있도록 합니다. `RawKey`와 `RawValue` struct는 각각 테이블의 key와 value을 인코딩 및 decoding하는 기능을 제공합니다. 이를 통해 database와의 상호 작용에서 성능 최적화와 유연성을 제공합니다.

#### Example Usage

```rust
// 예제 테이블 정의
struct MyTable;
impl Table for MyTable {
    const NAME: &'static str = "my_table";
    type Key = String;
    type Value = String;
}

// RawTable 사용
let raw_key = RawKey::new("my_key".to_string());
let raw_value = RawValue::new("my_value".to_string());

// 인코딩된 key와 value 얻기
let encoded_key = raw_key.raw_key();
let encoded_value = raw_value.raw_value();

// decoding된 key와 value 얻기
let decoded_key = raw_key.key().unwrap();
let decoded_value = raw_value.value().unwrap();
```

#### Code Analysis

##### Inputs

- **`T`**: `Table` 또는 `DupSort` trait를 구현하는 type.
- **`K`**: `Key` trait를 구현하는 key type.
- **`V`**: `Value` trait를 구현하는 value type.

##### Flow

1. **`RawTable`와 `RawDupSort` struct 정의**
   - `Table` trait를 구현하여 테이블의 이름과 key, value을 정의합니다.
2. **`RawKey`와 `RawValue` struct 사용**
   - key와 value을 인코딩 및 decoding하는 method를 제공합니다.
3. **key 인코딩 및 decoding**
   - `RawKey`는 key를 바이트 벡터로 인코딩하여 원시 key를 반환합니다.
   - `key()` method를 통해 decoding된 key를 얻을 수 있습니다.
4. **value 인코딩 및 decoding**
   - `RawValue`는 value을 압축하여 저장하고 원시 value을 반환합니다.
   - `value()` method를 통해 decoding된 value을 얻을 수 있습니다.
5. **압축 및 해제 처리**
   - `Compress` 및 `Decompress` trait를 구현하여 value의 압축 및 해제를 처리합니다.

##### Outputs

- **인코딩된 key와 value의 바이트 벡터**: database에 저장되거나 전송될 수 있는 형태.
- **decoding된 key와 value**: 애플리케이션 로직에서 사용 가능한 형태.

#### Key Concepts

- **Lazy Decoding/Encoding**: database와의 상호 작용에서 필요한 시점까지 key와 value을 decoding하거나 인코딩하지 않고 지연시켜, 성능을 최적화하고 불필요한 연산을 줄입니다.

- **type generic programming**: `RawTable`과 `RawDupSort`는 generic type `T`, `K`, `V`를 사용하여 다양한 테이블과 data type에 대해 유연하게 적용할 수 있습니다.

- **압축을 통한 저장 공간 최적화**: `Compress` trait를 구현하여 value의 압축 및 해제를 처리함으로써 저장 공간을 효율적으로 사용합니다.

- **database와의 효율적인 상호 작용**: 원시 바이트 형태의 key와 value을 사용하여 database에 직접 접근하고, 필요한 경우에만 decoding하여 애플리케이션 로직에서 사용합니다.

- **에러 처리와 안정성**: decoding 과정에서 발생할 수 있는 오류를 적절히 처리하여 data integrity을 유지하고, 안정적인 시스템 동작을 보장합니다.

#### Dependencies

- **`reth_db_api`**: database 테이블 및 trait 정의를 위해 사용됩니다.
- **`Compress` 및 `Decompress` trait**: data 압축 및 해제를 위해 구현됩니다.
- **Rust 표준 라이브러리**: 바이트 벡터(`Vec<u8>`) 및 기타 기본 type과 기능을 사용합니다.

---

### `crates/storage/db/srcs/tables/mod.rs`

#### Summary

Rust에서 database 테이블과 관련된 다양한 구조와 trait를 정의합니다. `TableViewer` trait는 database 테이블을 추상적으로 조작할 수 있게 하며, `tables!` 매크로는 여러 테이블을 정의하고 이들을 enum `Tables`로 관리합니다. 각 테이블은 key와 value의 type을 지정하며, 일부는 `DupSort` 테이블로 정의됩니다.

#### Example Usage

```rust
use reth_db::{TableViewer, Tables};
use reth_db_api::table::{DupSort, Table};

struct MyTableViewer;

impl TableViewer<()> for MyTableViewer {
    type Error = &'static str;

    fn view<T: Table>(&self) -> Result<(), Self::Error> {
        Ok(())
    }

    fn view_dupsort<T: DupSort>(&self) -> Result<(), Self::Error> {
        Ok(())
    }
}

let viewer = MyTableViewer {};

let _ = Tables::Headers.view(&viewer);
let _ = Tables::Transactions.view(&viewer);
```

#### Code Analysis

##### Flow

1. **`TableViewer` trait**
   - 테이블을 일반적으로 조작할 수 있는 method를 정의합니다.
2. **`tables!` 매크로**
   - 여러 테이블을 정의하고, enum `Tables`를 생성합니다.
3. **`Tables` enum**
   - 모든 테이블을 포함하며, 각 테이블의 이름과 type을 반환하는 method를 제공합니다.
4. **테이블 조작**
   - `Tables` enum은 `TableViewer`를 사용하여 테이블을 조작합니다.

#### Key Concepts

- **테이블 abstraction**: database 테이블을 일반적으로 조작할 수 있게 abstraction합니다.
- **key와 value의 인코딩/decoding**: 저장 및 검색을 위한 data type을 처리합니다.

#### Dependencies

- **`reth_db_api`**: 테이블 및 cursor trait를 위해 사용됩니다.
- **매크로**: 코드 생성을 위해 사용됩니다.

---

### `crates/storage/db/srcs/fuzz/inputs.rs`

#### Summary

`IntegerListInput` struct를 정의하고, 이를 `IntegerList`로 변환하는 기능을 제공합니다. 변환 과정에서 입력된 리스트가 비어 있는 경우 기본value으로 `[1u64]`를 사용하고, 리스트를 정렬하여 반환합니다. 이는 fuzzing(fuzzing) 테스트에서 사용되는 입력 data를 표준화된 형식으로 변환하기 위한 도구입니다.

#### Example Usage

```rust
use reth_primitives_traits::IntegerList;

let input = IntegerListInput(vec![3, 1, 2]);
let integer_list: IntegerList = input.into();
// integer_list는 [1, 2, 3]으로 정렬된 `IntegerList`가 됩니다.
```

#### Code Analysis

##### Inputs

- **`IntegerListInput`**: `Vec<u64>` type의 벡터를 포함하는 struct입니다.

##### Flow

1. **`IntegerListInput` struct 생성**
   - `IntegerListInput`은 `Vec<u64>`를 받아들입니다.
2. **`Into<IntegerList>` 구현**
   - `IntegerListInput`에서 `IntegerList`로의 변환을 위한 `Into` trait를 구현합니다.
3. **리스트가 비어 있는지 확인**
   - 변환 과정에서 리스트가 비어 있는지 검사합니다.
4. **기본value 설정**
   - 리스트가 비어 있다면 `[1u64]`로 대체합니다.
5. **리스트 정렬**
   - 리스트를 오름차순으로 정렬합니다.
6. **`IntegerList`로 반환**
   - 정렬된 리스트를 `IntegerList`로 반환합니다.

##### Outputs

- **`IntegerList`**: 정렬된 `u64` 벡터를 포함하는 리스트로 반환됩니다.

#### Key Concepts

- **fuzzing 입력 data 표준화**: fuzzing 테스트에서 다양한 입력 data를 처리하기 위해 입력을 표준화된 형식으로 변환합니다. 이는 테스트의 일관성과 신뢰성을 높입니다.

- **빈 입력 처리**: 입력 리스트가 비어 있을 경우 기본value을 사용하여 예외 상황을 처리합니다. 이는 예상치 못한 오류를 방지하고 테스트를 원활하게 진행할 수 있도록 합니다.

- **정렬된 data 사용**: 입력 data를 정렬하여 순서에 무관한 테스트를 수행할 수 있습니다. 이는 data 순서로 인한 테스트 결과의 변동을 줄여줍니다.

- **trait 구현을 통한 type 변환**: `Into` trait를 구현하여 struct 간의 변환을 간결하고 명확하게 수행합니다.

#### Dependencies

- **`reth_primitives_traits`**: `IntegerList` type을 제공하는 crate로, fuzzing 테스트에서 사용되는 기본 type과 trait를 포함합니다.

- **Rust 표준 라이브러리**: `Vec`, `u64` 등의 기본 type과 정렬 method(`sort_unstable()`)를 사용합니다.

---

### `crates/storage/db/srcs/fuzz/mod.rs`

#### Summary

매크로를 사용하여 객체를 인코딩하고 decoding하는 fuzzing 테스트를 자동으로 생성합니다. `impl_fuzzer_with_input!` 매크로는 주어진 객체 type에 대해 인코딩 및 decoding 함수를 생성하고, 이 과정에서 객체가 원래 객체와 일치하는지 확인합니다. `impl_fuzzer_key!`와 `impl_fuzzer_value_with_input!` 매크로는 각각 key와 value을 fuzzing하는 데 사용됩니다.

#### Example Usage

```rust
impl_fuzzer_key!(BlockNumberAddress);
impl_fuzzer_value_with_input!((IntegerList, IntegerListInput));
```

#### Code Analysis

##### Flow

1. **매크로 정의**
   - `impl_fuzzer_with_input!`은 인코딩 및 decoding 함수를 생성합니다.
2. **fuzz 함수**
   - `fuzz` 함수는 객체를 인코딩하고 decoding하여 일관성을 확인합니다.
3. **key와 value fuzzing**
   - key와 value을 위해 특수화된 매크로를 사용합니다.

#### Key Concepts

- **fuzzing 테스트**: 인코딩/decoding 오류를 찾기 위한 자동화된 테스트.
- **매크로 사용**: 테스트를 위한 반복적인 코드를 자동화합니다.

#### Dependencies

- **매크로**: 코드 생성을 위해 사용됩니다.
- **테스트 프레임워크**: fuzzing 테스트를 실행합니다.