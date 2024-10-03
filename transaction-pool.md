# transaction-pool
-----


## pool

### best.rs
트랜잭션 풀과 관련된 코드입니다. 코드의 주요 목표는 최적의 트랜잭션을 선택하여 실행 가능한 상태에서 반환하는 다양한 iterator와 이를 기반으로 한 구조체를 정의하는 것입니다

#### datastructure

```rust
pub(crate) struct BestTransactionsWithFees<T: TransactionOrdering> {
    pub(crate) best: BestTransactions<T>,
    pub(crate) base_fee: u64,
    pub(crate) base_fee_per_blob_gas: u64,
}
```

```rust
pub(crate) struct BestTransactions<T: TransactionOrdering> {
    pub(crate) all: BTreeMap<TransactionId, PendingTransaction<T>>,
    pub(crate) independent: BTreeSet<PendingTransaction<T>>,
    pub(crate) invalid: HashSet<TxHash>,
    pub(crate) new_transaction_receiver: Option<Receiver<PendingTransaction<T>>>,
    pub(crate) skip_blobs: bool,
}
```
- all : Contains a copy of _all_ transactions of the pending pool at the point
- independent : Transactions that can be executed right away
- invalid : Track the case where a yielded transactions is invalid
- new_transaction_receiver : new pending transactions are inserted into this iterator's pool before yielding the next value
- skip_blobs: Flag to control whether to skip blob transactions (EIP4844)


#### function next
```rust
fn next(&mut self) -> Option<Self::Item> {
        // find the next transaction that satisfies the base fee
        loop {
            let best = self.best.next()?;
            // If both the base fee and blob fee (if applicable for EIP-4844) are satisfied, return
            // the transaction
            if best.transaction.max_fee_per_gas() >= self.base_fee as u128 &&
                best.transaction
                    .max_fee_per_blob_gas()
                    .map_or(true, |fee| fee >= self.base_fee_per_blob_gas as u128)
            {
                return Some(best);
            }
            crate::traits::BestTransactions::mark_invalid(self, &best);
        }
    }
```
#### function add_new_transactions
An iterator that returns transactions that can be executed on the current state. This iterator guarantees that all transaction it returns satisfy both the base fee and blob fee
```rust
fn add_new_transactions(&mut self) {
        while let Some(pending_tx) = self.try_recv() {
            let tx = pending_tx.transaction.clone();
            //  same logic as PendingPool::add_transaction/PendingPool::best_with_unlocked
            let tx_id = *tx.id();
            if self.ancestor(&tx_id).is_none() {
                self.independent.insert(pending_tx.clone());
            }
            self.all.insert(tx_id, pending_tx);
        }
```



### blob.rs
트랜잭션 풀에서 Blob 트랜잭션을 관리하는 BlobTransactions 구조체와 관련된 구현입니다. Blob 트랜잭션은 이더리움의 특정 제안(EIP-4844)과 관련된 트랜잭션 유형으로, 이 코드는 해당 트랜잭션을 효율적으로 관리하고 우선순위를 정하며, 필요시 삭제 및 정렬을 수행하는 구조를 정의합니다.
A set of validated blob transactions in the pool that are __not pending__.
The purpose of this pool is keep track of blob transactions that are queued and to evict the worst blob transactions once the sub-pool is full.
This expects that certain constraints are met:
- blob transactions are always gap less

#### datastructure

```rust
pub(crate) struct BlobTransactions<T: PoolTransaction> {
    /// Keeps track of transactions inserted in the pool.
    ///
    /// This way we can determine when transactions were submitted to the pool.
    submission_id: u64,
    /// _All_ Transactions that are currently inside the pool grouped by their identifier.
    by_id: BTreeMap<TransactionId, BlobTransaction<T>>,
    /// _All_ transactions sorted by blob priority.
    all: BTreeSet<BlobTransaction<T>>,
    /// Keeps track of the current fees, so transaction priority can be calculated on insertion.
    pending_fees: PendingFees,
    /// Keeps track of the size of this pool.
    ///
    /// See also [`PoolTransaction::size`].
    size_of: SizeTracker,
}

struct BlobTransaction<T: PoolTransaction> {
    /// Actual blob transaction.
    transaction: Arc<ValidPoolTransaction<T>>,
    /// The value that determines the order of this transaction.
    ord: BlobOrd,
}

struct BlobOrd {
    /// Identifier that tags when transaction was submitted in the pool.
    pub(crate) submission_id: u64,
    /// The priority for this transaction, calculated using the [`blob_tx_priority`] function,
    /// taking into account both the blob and priority fee.
    pub(crate) priority: i64,
}
```

#### function add_transaction
Adds a new transactions to the pending queue

Panics ->  If the transaction is not a blob tx / If the transaction is already included.
```rust
pub(crate) fn add_transaction(&mut self, tx: Arc<ValidPoolTransaction<T>>) {
        assert!(tx.is_eip4844(), "transaction is not a blob tx");
        let id = *tx.id();
        assert!(!self.contains(&id), "transaction already included {:?}", self.get(&id).unwrap());
        let submission_id = self.next_id();

        // keep track of size
        self.size_of += tx.size();

        // set transaction, which will also calculate priority based on current pending fees
        let transaction = BlobTransaction::new(tx, submission_id, &self.pending_fees);

        self.by_id.insert(id, transaction.clone());
        self.all.insert(transaction);
    }

    fn next_id(&mut self) -> u64 {
        let id = self.submission_id;
        self.submission_id = self.submission_id.wrapping_add(1);
        id
    }
```

#### function new 
Creates a new blob transaction, based on the pool transaction, submission id, and current pending fees.
```rust
impl<T: PoolTransaction> BlobTransaction<T> {
    
    pub(crate) fn new(
        transaction: Arc<ValidPoolTransaction<T>>,
        submission_id: u64,
        pending_fees: &PendingFees,
    ) -> Self {
        let priority = blob_tx_priority(
            pending_fees.blob_fee,
            transaction.max_fee_per_blob_gas().unwrap_or_default(),
            pending_fees.base_fee as u128,
            transaction.max_fee_per_gas(),
        );
        let ord = BlobOrd { priority, submission_id };
        Self { transaction, ord }
    }
}
```

#### function update_priority
Updates the priority for the transaction based on the current pending fees.
```rust
impl<T: PoolTransaction> BlobTransaction<T> {
    pub(crate) fn update_priority(&mut self, pending_fees: &PendingFees) {
            self.ord.priority = blob_tx_priority(
                pending_fees.blob_fee,
                self.transaction.max_fee_per_blob_gas().unwrap_or_default(),
                pending_fees.base_fee as u128,
                self.transaction.max_fee_per_gas(),
             );
        }
    }
}
```


#### function cmp
Compares two `BlobOrd` instances. The comparison is performed in reverse order based on the priority field. This is because transactions with larger negative values in the priority field will take more fee jumps, making them take longer to become executable. Therefore, transactions with lower ordering should return `Greater`, ensuring they are evicted first. If the priority values are equal, the submission ID is used to break ties.
```rust
impl Ord for BlobOrd {
    fn cmp(&self, other: &Self) -> Ordering {
        other
            .priority
            .cmp(&self.priority)
            .then_with(|| self.submission_id.cmp(&other.submission_id))
    }
}
```

### events.rs

트랜잭션 풀에서 발생하는 다양한 트랜잭션 이벤트를 나타내는 열거형(Enum) 타입을 정의하고 있습니다. 이 코드는 블록체인 네트워크 내에서 트랜잭션의 상태 변화와 관련된 정보를 관리하기 위해 사용됩니다.

두 가지 주요 열거형(Enum)이 정의되어 있습니다:

FullTransactionEvent<T: PoolTransaction>
트랜잭션 풀 내에서 발생하는 상세 이벤트를 정의합니다.
트랜잭션이 추가되거나 삭제되었을 때, 블록에 포함되었을 때, 또는 다른 트랜잭션으로 대체되었을 때 등의 상태를 나타냅니다.
이 열거형은 트랜잭션의 전체 내용을 포함할 수 있도록 설계되었습니다.

TransactionEvent
FullTransactionEvent보다 간단한 형태의 트랜잭션 이벤트입니다.
serde 기능을 활성화하면 Serialize, Deserialize를 사용할 수 있으며, 트랜잭션의 상태를 직렬화/역직렬화할 수 있습니다.

#### datastructure
```rust
pub enum FullTransactionEvent<T: PoolTransaction> {
    /// Transaction has been added to the pending pool.
    Pending(TxHash),
    /// Transaction has been added to the queued pool.
    Queued(TxHash),
    /// Transaction has been included in the block belonging to this hash.
    Mined {
        /// The hash of the mined transaction.
        tx_hash: TxHash,
        /// The hash of the mined block that contains the transaction.
        block_hash: B256,
    },
    /// Transaction has been replaced by the transaction belonging to the hash.
    ///
    /// E.g. same (sender + nonce) pair
    Replaced {
        /// The transaction that was replaced.
        transaction: Arc<ValidPoolTransaction<T>>,
        /// The transaction that replaced the event subject.
        replaced_by: TxHash,
    },
    /// Transaction was dropped due to configured limits.
    Discarded(TxHash),
    /// Transaction became invalid indefinitely.
    Invalid(TxHash),
    /// Transaction was propagated to peers.
    Propagated(Arc<Vec<PropagateKind>>),
}

pub enum TransactionEvent {
    /// Transaction has been added to the pending pool.
    Pending,
    /// Transaction has been added to the queued pool.
    Queued,
    /// Transaction has been included in the block belonging to this hash.
    Mined(B256),
    /// Transaction has been replaced by the transaction belonging to the hash.
    ///
    /// E.g. same (sender + nonce) pair
    Replaced(TxHash),
    /// Transaction was dropped due to configured limits.
    Discarded,
    /// Transaction became invalid indefinitely.
    Invalid,
    /// Transaction was propagated to peers.
    Propagated(Arc<Vec<PropagateKind>>),
}
```

### listener.rs
트랜잭션 풀 내에서 발생하는 이벤트를 관리하고, 이벤트를 수신하고자 하는 여러 리스너(listener)들에게 이벤트를 전달(broadcast)하는 구조를 정의합니다. 각 트랜잭션의 상태가 변화할 때마다, 이를 감지하고 특정 트랜잭션에 대해 구독(subscribe)한 리스너들에게 해당 이벤트를 전파함으로써 트랜잭션 풀 내 상태 변화에 따라 반응할 수 있게 합니다.

#### datastructure
```rust
pub struct TransactionEvents {
    hash: TxHash,
    events: UnboundedReceiver<TransactionEvent>,
}

/// A Stream that receives [`FullTransactionEvent`] for _all_ transaction.
pub struct AllTransactionsEvents<T: PoolTransaction> {
    pub(crate) events: Receiver<FullTransactionEvent<T>>,
}

/// A type that broadcasts [`TransactionEvent`] to installed listeners.
///
/// This is essentially a multi-producer, multi-consumer channel where each event is broadcast to
/// all active receivers.
#[derive(Debug)]
pub(crate) struct PoolEventBroadcast<T: PoolTransaction> {
    /// All listeners for all transaction events.
    all_events_broadcaster: AllPoolEventsBroadcaster<T>,
    /// All listeners for events for a certain transaction hash.
    broadcasters_by_hash: HashMap<TxHash, PoolEventBroadcaster>,
}
```

#### function subscribe / subscribe all
Create a new subscription for the given transaction hash /Create a new subscription for all transactions
```rust
/// Create a new subscription for the given transaction hash.
    pub(crate) fn subscribe(&mut self, tx_hash: TxHash) -> TransactionEvents {
        let (tx, rx) = tokio::sync::mpsc::unbounded_channel();

        match self.broadcasters_by_hash.entry(tx_hash) {
            Entry::Occupied(mut entry) => {
                entry.get_mut().senders.push(tx);
            }
            Entry::Vacant(entry) => {
                entry.insert(PoolEventBroadcaster { senders: vec![tx] });
            }
        };
        TransactionEvents { hash: tx_hash, events: rx }
    }

    /// 
    pub(crate) fn subscribe_all(&mut self) -> AllTransactionsEvents<T> {
        let (tx, rx) = tokio::sync::mpsc::channel(TX_POOL_EVENT_CHANNEL_SIZE);
        self.all_events_broadcaster.senders.push(tx);
        AllTransactionsEvents::new(rx)
    }
```

#### functions of PoolEventBroadcast
```rust
 /// Notify listeners about a transaction that was added to the pending queue.
    pub(crate) fn pending(&mut self, tx: &TxHash, replaced: Option<Arc<ValidPoolTransaction<T>>>) {
        self.broadcast_event(tx, TransactionEvent::Pending, FullTransactionEvent::Pending(*tx));

        if let Some(replaced) = replaced {
            // notify listeners that this transaction was replaced
            self.replaced(replaced, *tx);
        }
    }

    /// Notify listeners about a transaction that was replaced.
    pub(crate) fn replaced(&mut self, tx: Arc<ValidPoolTransaction<T>>, replaced_by: TxHash) {
        let transaction = Arc::clone(&tx);
        self.broadcast_event(
            tx.hash(),
            TransactionEvent::Replaced(replaced_by),
            FullTransactionEvent::Replaced { transaction, replaced_by },
        );
    }

    /// Notify listeners about a transaction that was added to the queued pool.
    pub(crate) fn queued(&mut self, tx: &TxHash) {
        self.broadcast_event(tx, TransactionEvent::Queued, FullTransactionEvent::Queued(*tx));
    }

    /// Notify listeners about a transaction that was propagated.
    pub(crate) fn propagated(&mut self, tx: &TxHash, peers: Vec<PropagateKind>) {
        let peers = Arc::new(peers);
        self.broadcast_event(
            tx,
            TransactionEvent::Propagated(Arc::clone(&peers)),
            FullTransactionEvent::Propagated(peers),
        );
    }

    /// Notify listeners about a transaction that was discarded.
    pub(crate) fn discarded(&mut self, tx: &TxHash) {
        self.broadcast_event(tx, TransactionEvent::Discarded, FullTransactionEvent::Discarded(*tx));
    }

    /// Notify listeners that the transaction was mined
    pub(crate) fn mined(&mut self, tx: &TxHash, block_hash: B256) {
        self.broadcast_event(
            tx,
            TransactionEvent::Mined(block_hash),
            FullTransactionEvent::Mined { tx_hash: *tx, block_hash },
        );
    }

     // Broadcast an event to all listeners. Dropped listeners are silently evicted.
    fn broadcast(&mut self, event: TransactionEvent) {
        self.senders.retain(|sender| sender.send(event.clone()).is_ok())
    }
```

### mod.rs

Transaction can have 3 state :
1. Transaction can _never_ be valid
2. Transaction is _currently_ valid
3. Transaction is _currently_ invalid, but could potentially become valid in the future

(2.) and (3.) of a transaction can only be determined on the basis of the current state, whereas (1.) holds indefinitely. This means once the state changes (2.) and (3.) the state of a transaction needs to be reevaluated again.

following characteristics fall under (3.):

a) Nonce of a transaction is higher than the expected nonce for the next transaction of its sender. A distinction is made here whether multiple transactions from the same sender have gapless nonce increments.

a)(1) If _no_ transaction is missing in a chain of multiple transactions from the same sender (all nonce in row), all of them can in principle be executed on the current state one after the other.
a)(2) If there's a nonce gap, then all transactions after the missing transaction are blocked until the missing transaction arrives.
b) Transaction does not meet the dynamic fee cap requirement introduced by EIP-1559: The fee cap of the transaction needs to be no less than the base fee of block.



Transaction pool is made of 3 sub-pools:

 - Pending Pool: Contains all transactions that are valid on the current state and satisfy (3.a)(1): _No_ nonce gaps. A _pending_ transaction is considered _ready_ when it has the lowest nonce of all transactions from the same sender. Once a _ready_ transaction with nonce `n` has been executed, the next highest transaction from the same sender `n + 1` becomes ready.
 
 - Queued Pool: Contains all transactions that are currently blocked by missing transactions: (3. a)(2): _With_ nonce gaps or due to lack of funds.

- Basefee Pool: To account for the dynamic base fee requirement (3. b) which could render an EIP-1559 and all subsequent transactions of the sender currently invalid. The classification of transactions is always dependent on the current state that is changed as soon as a new block is mined. Once a new block is mined, the account changeset must be applied
to the transaction pool.

#### Terminology
----
- _Pending_: pending transactions are transactions that fall under (2.). These transactions can currently be executed and are stored in the pending sub-pool


- _Queued_: queued transactions are transactions that fall under category (3.). Those transactions are _currently_ waiting for state changes that eventually move them into category (2.) and become pending.

#### datastructure
```rust
pub struct PoolInner<V, T, S>
where
    T: TransactionOrdering,
{
    /// Internal mapping of addresses to plain ints.
    identifiers: RwLock<SenderIdentifiers>,
    /// Transaction validation.
    validator: V,
    /// Storage for blob transactions
    blob_store: S,
    /// The internal pool that manages all transactions.
    pool: RwLock<TxPool<T>>,
    /// Pool settings.
    config: PoolConfig,
    /// Manages listeners for transaction state change events.
    event_listener: RwLock<PoolEventBroadcast<T::Transaction>>,
    /// Listeners for new _full_ pending transactions.
    pending_transaction_listener: Mutex<Vec<PendingTransactionHashListener>>,
    /// Listeners for new transactions added to the pool.
    transaction_listener: Mutex<Vec<TransactionListener<T::Transaction>>>,
    /// Listener for new blob transaction sidecars added to the pool.
    blob_transaction_sidecar_listener: Mutex<Vec<BlobTransactionSidecarListener>>,
    /// Metrics for the blob store
    blob_store_metrics: BlobStoreMetrics,
}

```
 ## blobstore

