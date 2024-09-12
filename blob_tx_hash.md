Q：blob tx如何保证with和without blobsidecar计算出来的tx hash都一样？

如果看到[`BlobTx.encode`](https://github.com/ethereum/go-ethereum/blob/d8664490da47bfff1854869826268c486e6487d7/core/types/tx_blob.go#L200)的实现，容易产生一个疑问：显然是否带有blobsidecar会影响编码后的结果，那不是会影响到tx hash的一致性?

后来发现实际上并不会如此，而且geth有个[testcase](https://github.com/ethereum/go-ethereum/blob/d8664490da47bfff1854869826268c486e6487d7/core/types/tx_blob_test.go#L14)来确保这不会发生。下面看下[实现](https://github.com/ethereum/go-ethereum/blob/d8664490da47bfff1854869826268c486e6487d7/core/types/transaction.go#L492)。

```golang
// prefixedRlpHash writes the prefix into the hasher before rlp-encoding x.
// It's used for typed transactions.
func prefixedRlpHash(prefix byte, x interface{}) (h common.Hash) {
	sha := hasherPool.Get().(crypto.KeccakState)
	defer hasherPool.Put(sha)
	sha.Reset()
	sha.Write([]byte{prefix})
	rlp.Encode(sha, x)
	sha.Read(h[:])
	return h
}
```

其中的`rlp.Encode`并不会调到`BlobTx.encode`，而是直接对`BlobTx`结构进行编码：

```golang
// BlobTx represents an EIP-4844 transaction.
type BlobTx struct {
	ChainID    *uint256.Int
	Nonce      uint64
	GasTipCap  *uint256.Int // a.k.a. maxPriorityFeePerGas
	GasFeeCap  *uint256.Int // a.k.a. maxFeePerGas
	Gas        uint64
	To         common.Address
	Value      *uint256.Int
	Data       []byte
	AccessList AccessList
	BlobFeeCap *uint256.Int // a.k.a. maxFeePerBlobGas
	BlobHashes []common.Hash

	// A blob transaction can optionally contain blobs. This field must be set when BlobTx
	// is used to create a transaction for signing.
	Sidecar *BlobTxSidecar `rlp:"-"`

	// Signature values
	V *uint256.Int `json:"v" gencodec:"required"`
	R *uint256.Int `json:"r" gencodec:"required"`
	S *uint256.Int `json:"s" gencodec:"required"`
}
```

而其中的`Sidecar`字段由于``rlp:"-"``并不会被编码到`rlp`中。