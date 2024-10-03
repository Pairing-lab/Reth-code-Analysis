# consensus 과정에 필요한 trait가 정의되어 있다. 

아래와 같다 

```rust 
pub trait Consensus: Debug + Send + Sync {
    /// Validate if header is correct and follows consensus specification.
    ///
    /// This is called on standalone header to check if all hashes are correct.
    fn validate_header(&self, header: &SealedHeader) -> Result<(), ConsensusError>;

    /// Validate that the header information regarding parent are correct.
    /// This checks the block number, timestamp, basefee and gas limit increment.
    ///
    /// This is called before properties that are not in the header itself (like total difficulty)
    /// have been computed.
    ///
    /// **This should not be called for the genesis block**.
    ///
    /// Note: Validating header against its parent does not include other Consensus validations.
    fn validate_header_against_parent(
        &self,
        header: &SealedHeader,
        parent: &SealedHeader,
    ) -> Result<(), ConsensusError>;

    /// Validates the given headers
    ///
    /// This ensures that the first header is valid on its own and all subsequent headers are valid
    /// on its own and valid against its parent.
    ///
    /// Note: this expects that the headers are in natural order (ascending block number)
    fn validate_header_range(&self, headers: &[SealedHeader]) -> Result<(), HeaderConsensusError> {
        if let Some((initial_header, remaining_headers)) = headers.split_first() {
            self.validate_header(initial_header)
                .map_err(|e| HeaderConsensusError(e, initial_header.clone()))?;
            let mut parent = initial_header;
            for child in remaining_headers {
                self.validate_header(child).map_err(|e| HeaderConsensusError(e, child.clone()))?;
                self.validate_header_against_parent(child, parent)
                    .map_err(|e| HeaderConsensusError(e, child.clone()))?;
                parent = child;
            }
        }
        Ok(())
    }

    /// Validate if the header is correct and follows the consensus specification, including
    /// computed properties (like total difficulty).
    ///
    /// Some consensus engines may want to do additional checks here.
    ///
    /// Note: validating headers with TD does not include other Consensus validation.
    fn validate_header_with_total_difficulty(
        &self,
        header: &Header,
        total_difficulty: U256,
    ) -> Result<(), ConsensusError>;

    /// Validate a block disregarding world state, i.e. things that can be checked before sender
    /// recovery and execution.
    ///
    /// See the Yellow Paper sections 4.3.2 "Holistic Validity", 4.3.4 "Block Header Validity", and
    /// 11.1 "Ommer Validation".
    ///
    /// **This should not be called for the genesis block**.
    ///
    /// Note: validating blocks does not include other validations of the Consensus
    fn validate_block_pre_execution(&self, block: &SealedBlock) -> Result<(), ConsensusError>;

    /// Validate a block considering world state, i.e. things that can not be checked before
    /// execution.
    ///
    /// See the Yellow Paper sections 4.3.2 "Holistic Validity".
    ///
    /// Note: validating blocks does not include other validations of the Consensus
    fn validate_block_post_execution(
        &self,
        block: &BlockWithSenders,
        input: PostExecutionInput<'_>,
    ) -> Result<(), ConsensusError>;
}

```

# consensus 과정에서 일어날 수 있는 에러들이 정의되어 있다. 

```rust 
pub enum ConsensusError 
```


# Consensus를 테스트 하는 테스트 코드들이 정의되어 있다. 

