FDG有2个关键状态：[`ABSOLUTE_PRESTATE`](https://github.com/ethereum-optimism/optimism/blob/b36d065ebf12343f36455e47ed7f610804854bf0/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L124)和[`anchor state`](https://github.com/ethereum-optimism/optimism/blob/b36d065ebf12343f36455e47ed7f610804854bf0/packages/contracts-bedrock/src/dispute/AnchorStateRegistry.sol#L40)。


# `ABSOLUTE_PRESTATE`

`ABSOLUTE_PRESTATE`是`op-program`最初的状态，根据不同的vm或者game type，有不同的计算方式（例如[1](https://github.com/ethereum-optimism/optimism/blob/b36d065ebf12343f36455e47ed7f610804854bf0/op-program/Dockerfile.repro#L36)，[2](https://github.com/ethereum-optimism/optimism/blob/b36d065ebf12343f36455e47ed7f610804854bf0/op-program/Dockerfile.repro#L37)，[3](https://github.com/ethereum-optimism/optimism/blob/b36d065ebf12343f36455e47ed7f610804854bf0/op-program/Dockerfile.repro#L38)）。

初始的`ABSOLUTE_PRESTATE`，可以通过`intent.toml`中key `disputeAbsolutePrestate`来自定义：

```bash
[globalDeployOverrides]
  disputeAbsolutePrestate = "0x03df46da61723c81ea1d2ef51d71a4055f028e43725bbb011e775f7ca0c1b222"
```  
([source](https://github.com/ethereum-optimism/optimism/blob/b36d065ebf12343f36455e47ed7f610804854bf0/op-deployer/pkg/deployer/pipeline/opchain.go#L93))

由于[`ABSOLUTE_PRESTATE`](https://github.com/ethereum-optimism/optimism/blob/b36d065ebf12343f36455e47ed7f610804854bf0/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L124)在链上是`immutable`，在`code`里，而且合约[不](https://github.com/ethereum-optimism/optimism/blob/b36d065ebf12343f36455e47ed7f610804854bf0/packages/contracts-bedrock/src/L1/OPContractsManager.sol#L276)支持升级，所以`ABSOLUTE_PRESTATE`的更新只能通过重新部署一个FDG合约的方式，并通过调用[`DisputeGameFactory.setImplementation`](https://github.com/ethereum-optimism/optimism/blob/b36d065ebf12343f36455e47ed7f610804854bf0/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol#L260)进行替换。

值得一提的是，`ABSOLUTE_PRESTATE`并不包含L2 chainID信息，这个信息在需要时是可以通过FDG的`addLocalData`方法[添加](https://github.com/ethereum-optimism/optimism/blob/b36d065ebf12343f36455e47ed7f610804854bf0/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L594)的。

# `anchor state`

`anchor state`是已经finalize的L2状态：

`anchor state` = `ABSOLUTE_PRESTATE` + `L2 blocks`

初始的`anchor state`来自硬编码的[`DeployOPChainInput.startingAnchorRoots`](https://github.com/ethereum-optimism/optimism/blob/b36d065ebf12343f36455e47ed7f610804854bf0/packages/contracts-bedrock/scripts/deploy/DeployOPChain.s.sol#L162)，eventually来自[`DEFAULT_STARTING_ANCHOR_ROOTS`](https://github.com/ethereum-optimism/optimism/blob/b36d065ebf12343f36455e47ed7f610804854bf0/packages/contracts-bedrock/scripts/libraries/Constants.sol#L17)。

默认情况下，`anchor state`的更新是通过FDG的`resolve`[自动触发](https://github.com/ethereum-optimism/optimism/blob/b36d065ebf12343f36455e47ed7f610804854bf0/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L700)。
不过紧急情况下，也可以通过superchain guardian来[手动更新](https://github.com/ethereum-optimism/optimism/blob/b36d065ebf12343f36455e47ed7f610804854bf0/packages/contracts-bedrock/src/dispute/AnchorStateRegistry.sol#L106)。
