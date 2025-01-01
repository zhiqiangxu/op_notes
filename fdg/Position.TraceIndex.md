
`Position.TraceIndex`的[实现](https://github.com/ethereum-optimism/optimism/blob/a1df3e127b82c4d290fd568bc7e1680145dab55c/op-challenger/game/fault/types/position.go#L88)，用更直观的表示是这个：

```
rd := maxDepth - p.depth
return uint64(p.indexAtDepth<<rd | ((1 << rd) - 1))
```

为何成立？

因为层`p.depth`的[0, `p.indexAtDepth`]编号的节点，在层`maxDepth`一共会有`(p.indexAtDepth+1) << (maxDepth - p.depth)`个子节点，

对应最大子节点的编号是`(p.indexAtDepth+1) << (maxDepth - p.depth) - 1`，

也就是`p.indexAtDepth << (maxDepth - p.depth) | (1 << (maxDepth - p.depth) - 1)`，

即`p.indexAtDepth<<rd | ((1 << rd) - 1)`。