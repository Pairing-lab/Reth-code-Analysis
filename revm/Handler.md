``` rust 

pub struct Handler<'a, EvmWiringT: EvmWiring, H: Host + 'a> {
/// Handler hardfork
pub spec_id: EvmWiringT::Hardfork,
/// Instruction table type.
pub instruction_table: InstructionTables<'a, H>,
/// Registers that will be called on initialization.
pub registers: Vec<HandleRegisters<'a, EvmWiringT>>,
/// Validity handles.
pub validation: ValidationHandler<'a, EvmWiringT>,
/// Pre execution handle.
pub pre_execution: PreExecutionHandler<'a, EvmWiringT>,
/// Post Execution handle.
pub post_execution: PostExecutionHandler<'a, EvmWiringT>,
/// Execution loop that handles frames.
pub execution: ExecutionHandler<'a, EvmWiringT>,

}

```


Handler는 evm의 실행 로직을 담당한다.
``` rust 
pub struct PostExecutionHandler<'a, EvmWiringT: EvmWiring> {
/// Calculate final refund
pub refund: RefundHandle<'a, EvmWiringT>,
/// Reimburse the caller with ethereum it didn't spend.
pub reimburse_caller: ReimburseCallerHandle<'a, EvmWiringT>,
/// Reward the beneficiary with caller fee.
pub reward_beneficiary: RewardBeneficiaryHandle<'a, EvmWiringT>,
/// Main return handle, returns the output of the transact.
pub output: OutputHandle<'a, EvmWiringT>,
/// Called when execution ends.
/// End handle in comparison to output handle will be called every time after execution.
/// Output in case of error will not be called.
pub end: EndHandle<'a, EvmWiringT>,
/// Clear handle will be called always. In comparison to end that
/// is called only on execution end, clear handle is called even if validation fails.
pub clear: ClearHandle<'a, EvmWiringT>,

}
```



PreExcution - excution 전에 실행하는 것 
``` rust 
pub struct PreExecutionHandler<'a, EvmWiringT: EvmWiring> {
/// Load precompiles
pub load_precompiles: LoadPrecompilesHandle<'a, EvmWiringT>,
/// Main load handle
pub load_accounts: LoadAccountsHandle<'a, EvmWiringT>,
/// Deduct max value from the caller.
pub deduct_caller: DeductCallerHandle<'a, EvmWiringT>,
/// Apply EIP-7702 auth list
pub apply_eip7702_auth_list: ApplyEIP7702AuthListHandle<'a, EvmWiringT>,

}
```


Validation - 실행 하기 전에 가스랑 env 확인 
``` rust 
pub struct ValidationHandler<'a, EvmWiringT: EvmWiring> {

/// Validate and calculate initial transaction gas.

pub initial_tx_gas: ValidateInitialTxGasHandle<'a, EvmWiringT>,

/// Validate transactions against state data.

pub tx_against_state: ValidateTxEnvAgainstState<'a, EvmWiringT>,

/// Validate Env.

pub env: ValidateEnvHandle<'a, EvmWiringT>,

}
```


Excution 실행 
``` rust 

pub struct ExecutionHandler<'a, EvmWiringT: EvmWiring> {
/// Handles last frame return, modified gas for refund and
/// sets tx gas limit.
pub last_frame_return: LastFrameReturnHandle<'a, EvmWiringT>,
/// Executes a single frame.
pub execute_frame: ExecuteFrameHandle<'a, EvmWiringT>,
/// Frame call
pub call: FrameCallHandle<'a, EvmWiringT>,
/// Call return
pub call_return: FrameCallReturnHandle<'a, EvmWiringT>,
/// Insert call outcome
pub insert_call_outcome: InsertCallOutcomeHandle<'a, EvmWiringT>,
/// Frame crate
pub create: FrameCreateHandle<'a, EvmWiringT>,
/// Crate return
pub create_return: FrameCreateReturnHandle<'a, EvmWiringT>,
/// Insert create outcome.
pub insert_create_outcome: InsertCreateOutcomeHandle<'a, EvmWiringT>,
/// Frame EOFCreate
pub eofcreate: FrameEOFCreateHandle<'a, EvmWiringT>,
/// EOFCreate return
pub eofcreate_return: FrameEOFCreateReturnHandle<'a, EvmWiringT>,
/// Insert EOFCreate outcome.
pub insert_eofcreate_outcome: InsertEOFCreateOutcomeHandle<'a, EvmWiringT>,
}

```

