engine api基本的调用流程是：
1. 通过Forkchoice接口更新canonical head，并optionally开始基于canonical head构建新的区块
2. 通过GetPayload接口获取构建好的新区块
3. 通过NewPayload接口将构建的区块保存到EL
4. 通过Forkchoice接口把已保存的区块更新为canonical head


但是在看Forkchoice和NewPayload实现时，对其中两处细节有疑惑，分别是：
1. Forkchoice中对parent block的[检查](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/eth/catalyst/api.go#L303)
2. NewPayload中对parent block的[检查](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/eth/catalyst/api.go#L569)

特别是第一个检查，看起来像是：如果parent block的ttd也大于等于TerminalTotalDifficulty，就会返回失败。

但是Merge后每个区块的ttd应该都是停在一个固定值，这样岂不是没法调了？

最后发现是漏看了一个条件：
1. Forkchoice中还有个[判断](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/eth/catalyst/api.go#L289)：`block.Difficulty().BitLen() > 0`
2. NewPayload中还有个[判断](https://github.com/ethereum-optimism/op-geth/blob/336d284b606ec4792a605932201b97f04981db9d/eth/catalyst/api.go#L569)：`parent.Difficulty().BitLen() > 0`

这个判断使得作用范围仅是Merge前，不会对Merge后产生影响。

因为td大于等于TerminalTotalDifficulty的所有区块中，只有第一个区块的Difficulty大于0，之后的Difficulty都是0。

从而对于一个Difficulty大于0的区块，如果它的td大于等于TerminalTotalDifficulty，那么父区块的td必须小于TerminalTotalDifficulty。

