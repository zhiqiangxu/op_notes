

op_geth_test.go问题清单

1. 第一次跑op_geth_test.go遇到一个报错：`unknown field "faultGameWithdrawalDelay"`
   
    原因是packages/contracts-bedrock/deploy-config/devnetL1.json和go字段不一致。

    然后在[这里](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/packages/contracts-bedrock/.gitignore#L33)看到注释：

    ```
    Devnet config which changes with each 'make devnet-up'
    ```

    于是在根目录执行`make devnet-up`后解决。

2. [TestGethOnlyPendingBlockIsLatest](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L110)中的`checkPending`，会断言pending block和latest block相同。

    发现是修改了pending block的[实现](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/miner/worker.go#L360-L365)。(`RollupComputePendingBlock`默认是`false`)

3. [TestPreregolith](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L225)
   1. [GasUsed](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L237-L282)这个case用一地址发了笔gas为25000的deposit tx，然后断言该地址的余额不变。

        原因是DepositTx的gas price是0，[source](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/core/types/deposit_tx.go#L75-L77)。[这里](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/core/state_transition.go#L508)也有提及。

   2. [DepositNonce](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L285-L342)这个case是在测Regolith分叉之前创建合约，receipt的合约地址和实际的地址不一样。
   
        原因是deposit tx的nonce是0，[source](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/core/types/deposit_tx.go#L79)。相当于Regolith之前有bug，导致receipt的合约地址和实际的地址不一样，Regolith后做了[fix](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/core/state_processor.go#L114-L116)，并将实际生效的nonce[保存](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/core/state_processor.go#L147)到了Receipt.DepositNonce。

   3. [UnusedGasConsumed](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L344-L384)这个case是在测Regolith分叉之前，对于用户的deposit tx，gas limit是多少就会消耗多少，不会refund；对于系统(IsSystemTransaction为true)的deposit tx，gas used为0。[source](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/core/state_transition.go#L495-L501)。

4. [TestRegolith](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L409)
   1. [GasUsedIsAccurate](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L423-L473)这个case是在测Regolith分叉之后，deposit tx的gas used要像正常交易一样以实际evm消耗的为准，不再特殊处理。但由于gas price仍然是[0](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/core/types/deposit_tx.go#L75-L77)，所以sender balance并不会变化。
   2. [DepositNonceCorrect](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L475-L535)这个case是在测Regolith分叉之后，有效的nonce是caller每次+1后的值（同普通交易），而不是deposit tx写死的[0](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/core/types/deposit_tx.go#L79)。
   3. [ReturnUnusedGasToPool](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L537-L578)这个case是在测由于deposit tx的gas消耗以实际为准，所以为后面的tx腾出了空间，从而可以打包成功。
   4. [RejectSystemTx](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L580-L601)这个case是在测Regolith分叉之后，IsSystemTransaction被[禁用](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-node/rollup/derive/l1_block_info.go#L316)了，如果尝试打包IsSystemTransaction为true的deposit tx，会[打包失败](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/core/state_transition.go#L296https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/core/state_transition.go#L296)。
   5. [IncludeGasRefunds](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L603-L722)这个case比较复杂：首先deployData是构造了一个constructor，它的返回结果是sstoreContract，也就是合约做的事情是把calldata的第一个字存到slot 0。创建好合约后调用了3次，分别在slot 0存了一次非零和两次零值。第一次存零值会有refund所以开销会比第二次存零值低。
5. [TestPreCanyon](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L726)
   1. [ReturnsNilWithdrawals](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L739-L758)这个case是在测在Canyon分叉之前，ExecutionPayload.Withdrawals应该是nil，[source](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/eth/catalyst/api.go#L210)。
   2. [RejectPushZeroTx](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L760-L788)这个case是在测在Canyon分叉（包括主网的Shanghai和Capella分叉）之前，push0指令不支持（主网的Shanghai分叉才开始支持），从而receipt状态会是ReceiptStatusFailed。
6. [TestCanyon](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L792)
   1. [ReturnsEmptyWithdrawals](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L807)这个case是在测Canyon分叉后空的Withdrawals不再是nil而是空数组。
   2. [AcceptsPushZeroTxn](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L833-L862)这个case是在测在Canyon分叉之后，push0指令生效，从而receipt状态会是ReceiptStatusSuccessful。
7. [TestPreEcotone](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L866)
   1. 这个test是在测Ecotone分叉之前，l2的ParentBeaconBlockRoot为nil，并且不支持Tstore指令（主网Cancun分叉才开始支持）。
8. [TestEcotone](https://github.com/ethereum-optimism/optimism/blob/ac5b061dfce6a9817b928a8703be9252daaeeca7/op-e2e/op_geth_test.go#L934)
   1. 这个test是在测Ecotone分叉之后，l2的ParentBeaconBlockRoot不为nil，并且支持Tstore指令。
