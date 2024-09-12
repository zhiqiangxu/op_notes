plasma为可选功能，如想开启，需要在初始搭建时将DeployConfig的[`UsePlasma`](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/op-chain-ops/genesis/config.go#L271)设置为true。

op-batcher会通过[SetInput](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/op-batcher/batcher/driver.go#L504)把原数据上传DA，并将DA commitment上链。

op-node在derive区块时，会调用[`DA.GetInput(da_commitment)`](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/op-plasma/damgr.go#L193)获取原数据，该方法内部会调用[`TrackCommitment`](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/op-plasma/damgr.go#L207)将`da_commitment`追踪起来（challenge一个没有被追踪的`da_commitment`没有意义）.

但为了防止data withhold，增加了DA challenge机制：在DA commitment上链后一定区块(`ChallengeWindow`)内如果被challenge，则需要在一定区块(`ResolveWindow`)内在链上reveal原数据，否则这批数据会视为无效。

举例：

1. 当batcher把DA commitment在高度`h`上链后，需要在高度`[h, h+ChallengeWindow]`内才能challenge。

2. 当DA commitment在高度`h'`被challenge后，需要在高度`[h', h'+ResolveWindow]`内才能resolve。

plasma对一批数据无效化的处理方式比较tricky，有两种情况：
1. 因为DA challenge导致无效。
   1. op-node在derive高度`h`时，如果[`DA.GetInput(da_commitment)`](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/op-plasma/damgr.go#L193)成功获取了原数据，则会马上基于这批数据derive L2区块，并不会等DA challenge。如果后面发现这批数据被challenge了并没有及时reveal原数据，则会[触发](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/op-plasma/dastate.go#L178)reorg，清空追踪的`da_commitment`但[保留](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/op-plasma/damgr.go#L179)`challenge`。发生reorg后系统会回退到[`FindL2Heads`](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/op-node/rollup/sync/start.go#L107)返回的一个[足够老](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/op-node/rollup/sync/start.go#L206)的L2区块作为起点重新derive，但这次[`DA.GetInput(da_commitment)`](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/op-plasma/damgr.go#L193)会[返回`ErrExpiredChallenge`](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/op-plasma/damgr.go#L204)，从而跳过这批数据。
2. 因为L1 reorg导致无效。
   1. [清空](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/op-plasma/damgr.go#L183)追踪的`da_commitment`和`challenge`，并回退到[`FindL2Heads`](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/op-node/rollup/sync/start.go#L107)返回的一个[足够老](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/op-node/rollup/sync/start.go#L206)的L2区块作为起点重新derive。



DA commitment分为`Keccak256CommitmentType`和`GenericCommitmentType`。对celestia/avail等第三方DA的支持属于`GenericCommitmentType`。

plasma目前[DataAvailabilityChallenge](https://github.com/ethereum-optimism/optimism/blob/46c49968de23bbc5f820b3710de248b21b7f05c4/packages/contracts-bedrock/src/L1/DataAvailabilityChallenge.sol)合约里[只支持](https://github.com/ethereum-optimism/optimism/blob/0d3273e267802599399ae99426a44a8dedfd8444/packages/contracts-bedrock/src/L1/DataAvailabilityChallenge.sol#L18)`Keccak256CommitmentType`，[不支持](https://github.com/ethereum-optimism/optimism/blob/0d3273e267802599399ae99426a44a8dedfd8444/op-plasma/commitment.go#L37)`GenericCommitmentType`。

这也意味着如果用了celestia/avail等第三方DA，将无法进行DA challenge。









