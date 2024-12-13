OP的手续费去了3个vault：

1. [SequencerFeeVault](https://github.com/ethereum-optimism/op-geth/blob/d5dbfcfaedbae9a58251e34288d512ea9f155bf2/core/state_transition.go#L591)，地址`0x4200000000000000000000000000000000000011`
2. [BaseFeeVault](https://github.com/ethereum-optimism/op-geth/blob/d5dbfcfaedbae9a58251e34288d512ea9f155bf2/core/state_transition.go#L606)，地址`0x4200000000000000000000000000000000000019`
3. [L1FeeVault](https://github.com/ethereum-optimism/op-geth/blob/d5dbfcfaedbae9a58251e34288d512ea9f155bf2/core/state_transition.go#L612)，地址`0x420000000000000000000000000000000000001A`


但第一个vault比较implicit，代码中是取的`BlockContext.Coinbase`，这个值来自于[这里](https://github.com/ethereum-optimism/op-geth/blob/d5dbfcfaedbae9a58251e34288d512ea9f155bf2/core/evm.go#L53)，最终取的是[header.Coinbase](https://github.com/ethereum-optimism/op-geth/blob/d5dbfcfaedbae9a58251e34288d512ea9f155bf2/consensus/beacon/consensus.go#L80)，所以问题转化为找`header.Coinbase`的源头。

不难发现`header.Coinbase`是由[`generateParams.coinbase`](https://github.com/ethereum-optimism/op-geth/blob/d5dbfcfaedbae9a58251e34288d512ea9f155bf2/miner/worker.go#L236)赋值的，从而问题转化为找`generateParams.coinbase`的源头。

但`generateParams.coinbase`其实有2个地方赋值：
1. [getPending](https://github.com/ethereum-optimism/op-geth/blob/d5dbfcfaedbae9a58251e34288d512ea9f155bf2/miner/miner.go#L211)
2. [buildPayload](https://github.com/ethereum-optimism/op-geth/blob/d5dbfcfaedbae9a58251e34288d512ea9f155bf2/miner/payload_building.go#L300)

OP实际走的是`buildPayload`。

继续trace不难发现，`BuildPayloadArgs.FeeRecipient`是来自[`PayloadAttributes.SuggestedFeeRecipient`](https://github.com/ethereum-optimism/op-geth/blob/d5dbfcfaedbae9a58251e34288d512ea9f155bf2/eth/catalyst/api.go#L467)。

而`PayloadAttributes.SuggestedFeeRecipient`的源头是[`SequencerFeeVaultAddr`](https://github.com/ethereum-optimism/optimism/blob/0c0f7cff9f3a5c1090d2be68c52e12fddf6b7b8a/op-node/rollup/derive/attributes.go#L163)。