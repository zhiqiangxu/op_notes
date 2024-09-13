本文讨论L2数据从batcher提交到node derive的流转核心过程。

batcher的行为主要由DA类型和batch类型决定。

首先batcher可以选择3种DA类型：
1. `calldata`
   1. 每个batcher tx包括1个`frame`，但每个channel理论上可以包括任意数量的`frame`。
   2. `MaxFrameSize`是`CLIConfig.MaxL1TxSize - 1`, 其中`CLIConfig.MaxL1TxSize`默认是`120_000`。
2. `blobs`
   1. 每个batcher tx包括1~6个`frame`，并且每个`channel`包括同样数量的`frame`。
   2. `MaxFrameSize`是`eth.MaxBlobDataSize - 1`。
   3. batcher tx的每个BLOB[包含](https://github.com/ethereum-optimism/optimism/blob/5cdcfd46f0f309e9901dafd8f4c47357c8320c34/op-batcher/batcher/tx_data.go#L45)1个`frame`。
3. `auto`（智能切换到1或2）

并且可以选择2种batch类型:
1. `SingularBatch`
   1. `channel`由多个`SingularBatch`分别压缩后拼接构成。
   2. ready条件是凑满一个`frame`或者即将超时。
2. `SpanBatch`
   1. `channel`由1个`SpanBatch`整体压缩后构成。
   2. ready条件是凑满整个`channel`或者即将超时。


下面讨论接收端node derive的流转核心过程：
1. [`DataAvailabilitySource`](https://github.com/ethereum-optimism/optimism/blob/5cdcfd46f0f309e9901dafd8f4c47357c8320c34/op-node/rollup/derive/l1_retrieval.go#L14)扫描L1区块中的batcher tx并解析出`frame`字节。
2. `FrameQueue`将`frame`字节[解析](https://github.com/ethereum-optimism/optimism/blob/4656d49ac32e7cfecb3fb29e79e39f641defa198/op-node/rollup/derive/frame_queue.go#L42)成`frame`。
3. `ChannelBank`将`frame`[组装](https://github.com/ethereum-optimism/optimism/blob/4656d49ac32e7cfecb3fb29e79e39f641defa198/op-node/rollup/derive/channel_bank.go#L197)成`channel`。
4. `ChannelInReader`将`channel`[解析](https://github.com/ethereum-optimism/optimism/blob/4656d49ac32e7cfecb3fb29e79e39f641defa198/op-node/rollup/derive/channel_in_reader.go#L73)成多个`Batch`(`SingularBatch`或者`SpanBatch`)。
5. `BatchQueue`将`Batch`和L2 safe head对齐，并且如果`Batch`是`SpanBatch`的话，会[拆](https://github.com/ethereum-optimism/optimism/blob/6cf35daa3bcf760f1b3927514e438caf087bfc17/op-node/rollup/derive/batch_queue.go#L199)成`SingularBatch`返回给上层。
   1. 如果`Batch`[落后](https://github.com/ethereum-optimism/optimism/blob/e374443006d757d93330b7d5263bc90b20032faa/op-node/rollup/derive/batch_queue.go#L116)于L2 safe head，则认为无效。
6. `AttributesQueue`将`Batch`[补齐](https://github.com/ethereum-optimism/optimism/blob/4656d49ac32e7cfecb3fb29e79e39f641defa198/op-node/rollup/derive/attributes_queue.go#L72)成完整的`AttributesWithParent`。
   1. `Batch`只包括L2交易信息，不包括L1 deposit tx。
7. `DerivationPipeline`负责流水线的[Reset](https://github.com/ethereum-optimism/optimism/blob/4656d49ac32e7cfecb3fb29e79e39f641defa198/op-node/rollup/derive/pipeline.go#L162)以及[AdvanceL1Block](https://github.com/ethereum-optimism/optimism/blob/4656d49ac32e7cfecb3fb29e79e39f641defa198/op-node/rollup/derive/pipeline.go#L187)，并[返回](https://github.com/ethereum-optimism/optimism/blob/4656d49ac32e7cfecb3fb29e79e39f641defa198/op-node/rollup/derive/pipeline.go#L184)`AttributesWithParent`给上层。
8. `PipelineDeriver`[触发](https://github.com/ethereum-optimism/optimism/blob/4656d49ac32e7cfecb3fb29e79e39f641defa198/op-node/rollup/derive/deriver.go#L125)`DerivedAttributesEvent`，这时`needAttributesConfirmation`会[标为`true`](https://github.com/ethereum-optimism/optimism/blob/4656d49ac32e7cfecb3fb29e79e39f641defa198/op-node/rollup/derive/deriver.go#L124)，不再继续derive。
9. `AttributesHandler`[收到](https://github.com/ethereum-optimism/optimism/blob/e9337dbb4fd6deae6bdbda94f5178f4bad0c0f40/op-node/rollup/attributes/attributes.go#L63)`DerivedAttributesEvent`并触发`ConfirmReceivedAttributesEvent`和`PendingSafeRequestEvent`。
   1.  `EngDeriver`[收到](https://github.com/ethereum-optimism/optimism/blob/e9337dbb4fd6deae6bdbda94f5178f4bad0c0f40/op-node/rollup/engine/events.go#L373)`PendingSafeRequestEvent`并触发`PendingSafeUpdateEvent`。
   2.  `AttributesHandler`[收到](https://github.com/ethereum-optimism/optimism/blob/e9337dbb4fd6deae6bdbda94f5178f4bad0c0f40/op-node/rollup/attributes/attributes.go#L61)`PendingSafeUpdateEvent`，并[consolidateNextSafeAttributes](https://github.com/ethereum-optimism/optimism/blob/e9337dbb4fd6deae6bdbda94f5178f4bad0c0f40/op-node/rollup/attributes/attributes.go#L154)或触发[`BuildStartEvent`](https://github.com/ethereum-optimism/optimism/blob/e9337dbb4fd6deae6bdbda94f5178f4bad0c0f40/op-node/rollup/attributes/attributes.go#L158)。
10. `DerivationPipeline`[收到](https://github.com/ethereum-optimism/optimism/blob/e9337dbb4fd6deae6bdbda94f5178f4bad0c0f40/op-node/rollup/derive/deriver.go#L132)`ConfirmReceivedAttributesEvent`，将`needAttributesConfirmation`会标为`false`，从而可以继续derive。

