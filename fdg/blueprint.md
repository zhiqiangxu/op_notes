blueprint是一个代码复用相关的EIP，具体参考[这里](https://eips.ethereum.org/EIPS/eip-5202)。

如果不用blueprint，FDG是直接部署的（[source](https://github.com/ethereum-optimism/optimism/blob/3f4d94a444bfc63456aafaf045882d1ca5b95aba/packages/contracts-bedrock/scripts/deploy/DeployDisputeGame.s.sol#L291-L307)）。

而如果用了blueprint，FDG会拆成2个合约（[source](https://github.com/ethereum-optimism/optimism/blob/3f4d94a444bfc63456aafaf045882d1ca5b95aba/packages/contracts-bedrock/scripts/deploy/DeployImplementations.s.sol#L508-L509)）。

但blueprint实际只会增加3字节的preamble，为何就需要拆分？

这是因为[EIP3860](https://eips.ethereum.org/EIPS/eip-3860)将initcode的最大size定义成了48KB，而runtime的最大size仍然是24KB。

如果运行`just size-check`，可以看到：

```bash
╭-----------------------------------+------------------+-------------------+--------------------+---------------------╮
| Contract                          | Runtime Size (B) | Initcode Size (B) | Runtime Margin (B) | Initcode Margin (B) |
|-----------------------------------+------------------+-------------------+--------------------+---------------------|
| FaultDisputeGame                  | 23,898           | 25,906            | 678                | 23,246              |
|-----------------------------------+------------------+-------------------+--------------------+---------------------|
| PermissionedDisputeGame           | 24,533           | 26,672            | 43                 | 22,480              |
|-----------------------------------+------------------+-------------------+--------------------+---------------------|
```
因此不用blueprint时，FDG的size是满足要求的。

但如果用了blueprint，则需要把initcode作为runtime存在链上，因此需要保证加上3字节的preamble后仍然小于24KB（[source](https://github.com/ethereum-optimism/optimism/blob/3f4d94a444bfc63456aafaf045882d1ca5b95aba/packages/contracts-bedrock/src/libraries/Blueprint.sol#L217)）。

[这个](https://github.com/ethereum-optimism/optimism/pull/13720)PR加了个相关comment。