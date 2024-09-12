[`FaultDisputeGame.sol`](https://github.com/ethereum-optimism/optimism/blob/1e62a0b7adccfc3b9d937f13c4671c8febc07ba5/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)中有这么个函数：

```solidity
    function l1Head() public pure returns (Hash l1Head_) {
        l1Head_ = Hash.wrap(_getArgFixedBytes(0x20));
    }
```

它是怎么工作的？

首先可以看到`FaultDisputeGame`是通过[这里](https://github.com/ethereum-optimism/optimism/blob/1e62a0b7adccfc3b9d937f13c4671c8febc07ba5/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol#L109)的`clone`函数创建的。

`clone`的具体实现在[这里](https://github.com/wighawag/clones-with-immutable-args/blob/f5ca191afea933d50a36d101009b5644dc28bc99/src/ClonesWithImmutableArgs.sol#L32)。

最终也是生成一个类似[`EIP-1167`](https://eips.ethereum.org/EIPS/eip-1167)的最小化proxy，不同之处是：在复用implementation的同时，在`clone`时可以廉价地设置一些`immutable data`。具体是通过对implementation进行`delegate call`时，把`immutable data`（[`clone`](https://github.com/wighawag/clones-with-immutable-args/blob/f5ca191afea933d50a36d101009b5644dc28bc99/src/ClonesWithImmutableArgs.sol#L32)的`data`参数）追加编码在`calldata`尾部，`calldata`的布局是这样：

```
[0 - cds): orig calldata, [cds - cds + e): extraData
```


即在原本的`calldata`后面追加了`extraData`，而`extraData`是在`immutable data`后面额外追加了2字节的[`extraLength`](https://github.com/wighawag/clones-with-immutable-args/blob/f5ca191afea933d50a36d101009b5644dc28bc99/src/ClonesWithImmutableArgs.sol#L121)，`extraLength`表示额外追加了多少`calldata`数据。

这样在implementation里便可以通过[`_getImmutableArgsOffset`](https://github.com/wighawag/clones-with-immutable-args/blob/f5ca191afea933d50a36d101009b5644dc28bc99/src/Clone.sol#L87)读取`calldata`的最后2字节知道`calldata`从什么`offset`开始是`immutable data`。

这样的好处是这些`immutable data`不需要slot storage开销，只有`codecopy`和`calldataload`开销，并且可以在`clone`时作为动态参数，否则的话只能在implementation部署时设定了。

所以开头的`l1Head`返回的是`immutable data`从第`0x20`字节开始的32字节，这里的`immutable data`是：

```solidity
abi.encodePacked(_rootClaim, parentHash, _extraData)
```

所以`l1Head`返回的是创建`FaultDisputeGame`时的`parentHash`。

而这里的`_extraData`目前是[`l2BlockNum`](https://github.com/ethereum-optimism/optimism/blob/1e62a0b7adccfc3b9d937f13c4671c8febc07ba5/op-challenger/game/fault/contracts/gamefactory.go#L138)。