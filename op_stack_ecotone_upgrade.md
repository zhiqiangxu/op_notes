<!-- omit in toc -->
# OP Stack Ecotone Upgrade

**Table of Contents**
- [Background](#background)
- [Mechanism](#mechanism)
	- [Before Ecotone, a.k.a Bedrock](#before-ecotone-aka-bedrock)
	- [After Ecotone](#after-ecotone)
	- [Compare](#compare)
- [Example](#example)
- [References](#references)


# Background

On the high level, OP Stack mainly does three things:
1. [Sequencing](https://github.com/ethereum-optimism/optimism/blob/111f3f3a3a2881899662e53e0f1b2f845b188a38/op-node/rollup/driver/state.go#L269), which generates and executes L2 blocks, but does not publish to L1 yet.
2. Publishing L2 [data](https://github.com/ethereum-optimism/optimism/blob/3179d49f1cdbf7cea384544f250b4762a43222f1/op-batcher/cmd/main.go) and [state commitment](https://github.com/ethereum-optimism/optimism/blob/3179d49f1cdbf7cea384544f250b4762a43222f1/op-proposer/cmd/main.go) to L1.
3. [Deriving](https://github.com/ethereum-optimism/optimism/blob/111f3f3a3a2881899662e53e0f1b2f845b188a38/op-node/rollup/driver/state.go#L314) L2 states from published L2 data on L1.

Thus OP Stack transaction fees are made up of:
1. `L2 Execution Gas Fee` : The cost to run the transaction on OP Stack (L2)
2. `L1 Data Fee`: The cost of “rolling up” transactions into batches and submitting them to Ethereum (L1)

Or more formally:

$$
\begin{align*} 
OP Stack Transaction Fee &= [ L2 Execution Gas Fee ] + [ L1 Data Fee ]\\
L2 Execution Gas Fee & = L2 Gas Price * L2 Gas Used\\
\end{align*}
$$

Like Ethereum, OP Stack uses the EIP-1559 mechanism to set the base fee for transactions. The total price per unit gas(a.k.a `gas price`) that a transaction pays is the sum of the base fee and the optional additional priority fee.

Because OP Stack is EVM equivalent, the `gas` used by a transaction on OP Stack is exactly the same as the gas used by the same transaction on Ethereum. If a transaction costs 100,000 gas on Ethereum, it will cost 100,000 gas on OP Stack. The only difference is that the gas price on OP Stack is much lower than the gas price on Ethereum so you'll end up paying much less in ETH.

For this post, we're more interested in the `L1 Data Fee` part, which hopefully will be greatly reduced after the Ecotone Upgrade.

Let's quickly dive into the details.

# Mechanism

Before `Ecotone`, OP Stack publishes L2 transactions [using calldata](https://github.com/ethereum-optimism/optimism/blob/3179d49f1cdbf7cea384544f250b4762a43222f1/op-batcher/batcher/driver.go#L389). The derivation process will try to decode L2 data [from calldata](https://github.com/ethereum-optimism/optimism/blob/3179d49f1cdbf7cea384544f250b4762a43222f1/op-node/rollup/derive/data_source.go#L73).

After `Ecotone`, OP Stack has the option to publish L2 transactions [using blobs](https://github.com/ethereum-optimism/optimism/blob/3179d49f1cdbf7cea384544f250b4762a43222f1/op-batcher/batcher/driver.go#L380). The derivation process will try to decode L2 data [from blobs](https://github.com/ethereum-optimism/optimism/blob/3179d49f1cdbf7cea384544f250b4762a43222f1/op-node/rollup/derive/data_source.go#L71), but will also fall back to decode [from calldata](https://github.com/ethereum-optimism/optimism/blob/3179d49f1cdbf7cea384544f250b4762a43222f1/op-node/rollup/derive/blob_data_source.go#L128) for non-blob transactions.

Let's take a closer look at the cost of `L1 Data Fee` before and after the Ecotone Upgrade.

## Before Ecotone, a.k.a Bedrock

$$
\begin{align*} 
L1 Data Fee &= Base Fee Scalar * L1 Base Fee * [ Calldata + Fixed Overhead ]\\
Calldata &= CountZeroBytes(TxData) * 4 + CountNonZeroBytes(TxData) * 16\\
Base Fee Scalar &= 0.684\\
Fixed Overhead &= 188\\
\end{align*}
$$

(`Base Fee Scalar` accounts for the efficiency gains from data compression and gas gap caused by asynchoronous publishing, and is [configured](https://github.com/ethereum-optimism/optimism/blob/3179d49f1cdbf7cea384544f250b4762a43222f1/op-node/rollup/derive/l1_block_info.go#L292) via [`SystemConfig`](https://github.com/ethereum-optimism/optimism/blob/3179d49f1cdbf7cea384544f250b4762a43222f1/op-service/eth/types.go#L352-L369) on L1.)

Refer to [here](https://github.com/ethereum-optimism/op-geth/blob/0402d543c3d0cff3a3d344c0f4f83809edb44f10/core/types/rollup_cost.go#L160-L186) for the implementation details and [here](https://optimistic.etherscan.io/address/0x4200000000000000000000000000000000000015#readProxyContract) for the constant values.

## After Ecotone

$$
\begin{align*} 
L1 Data Fee &= Calldata * [ Base Fee Scalar * L1 Base Fee + Blob Base Fee Scalar * L1 Blob Base Fee / 16 ]\\
Calldata &= CountZeroBytes(TxData) * 4 + CountNonZeroBytes(TxData) * 16\\
Base Fee Scalar &= 0.001368 \\
Blob Base Fee Scalar &= 0.810949\\
\end{align*}
$$

Refer to [here](https://github.com/ethereum-optimism/op-geth/blob/0402d543c3d0cff3a3d344c0f4f83809edb44f10/core/types/rollup_cost.go#L188-L219) for the implementation details and [here](https://optimistic.etherscan.io/address/0x4200000000000000000000000000000000000015#readProxyContract) for the constant values.

## Compare

Let's substitute the formula for `L1 Data Fee` before and after Ecotone.

The diff is:

$$
\begin{align*} 
L1 Data Fee Diff &= [ Calldata ] * [ 0.001368 * L1 Base Fee + 0.810949 * L1 Blob Base Fee / 16 ] - 0.684 * L1 Base Fee * [ Calldata + 188 ]\\
				&= [ Calldata ] * [ 0.810949 * L1 Blob Base Fee / 16 - 0.682632 * L1 Base Fee ] - 128.592 * L1 Base Fee
\end{align*}				
$$

The ratio is:

$$
\begin{align*} 
\frac{Ecotone L1 Data Fee}{Bedrock L1 Data Fee} &= \frac{[ Calldata ] * [ 0.001368 * L1 Base Fee + 0.810949 * L1 Blob Base Fee / 16 ]}{0.684 * L1 Base Fee * [ Calldata + 188 ]}\\
					&= \frac{[ 0.001368 * L1 Base Fee + 0.810949 * L1 Blob Base Fee / 16 ]}{0.684 * L1 Base Fee * [ 1 + \frac{188}{Calldata}]}
\end{align*}	
$$

If `L1 Blob Base Fee` is negligible compared with `L1 Base Fee`(which will be the case initially when blobs have few users), then:

$$
\frac{Ecotone L1 Data Fee}{Bedrock L1 Data Fee} \approx \frac{[ 0.001368 * L1 Base Fee ]}{[0.684 * L1 Base Fee]} = 0.002
$$

# Example

Let's take [this transaction](https://optimistic.etherscan.io/tx/0xdcfd787b63db5955f75b9bddab0d9d15b60f367a2ff6e7120830d3643f2e38a1) as an example for bedrock:


$$
\begin{align*} 
OP Stack Transaction Fee &= [ L2 Execution Gas Fee ] + [ L1 Data Fee ]&&\\
						 &= [ L2 Gas Price * L2 Gas Used ] + [ Base Fee Scalar * L1 Base Fee * [ Calldata + Fixed Overhead ] ]&&\\
						 &= [ 0.000021358 Gwei * 21000 ] + [ 0.684 * 65.943700927 Gwei * 1972 ]&&\\
						 &= 0.000000000448518 ETH + 0.000088948029107982 ETH&&\\
						 &= 0.000088948477625982 ETH
\end{align*}
$$

Taking [this transaction](https://optimistic.etherscan.io/tx/0x5375b273b9758190eddc6b82a0f888408515a9b378d3abd47517882d51989a4d) as an example for ecotone:

$$
\begin{align*} 
OP Stack Transaction Fee &= [ L2 Execution Gas Fee ] + [ L1 Data Fee ]&&\\
						 &= [ L2 Gas Price * L2 Gas Used ] + [ Calldata * [ Base Fee Scalar * L1 Base Fee + Blob Base Fee Scalar * L1 Blob Base Fee / 16 ] ]&&\\
						 &= [ 0.007948736 Gwei * 29758 ] + [ 2552 * [ 0.001368 * 41.921275967 Gwei + 0.810949 * 1 / 16 wei ] ]&&\\
						 &= 0.000000236538485888 ETH + 0.000000146352875823 ETH&&\\
						 &= 0.000000382891361711 ETH
\end{align*}
$$

# References
1. https://docs.optimism.io/stack/transactions/fees
2. https://specs.optimism.io/protocol/exec-engine.html#ecotone-l1-cost-fee-changes-eip-4844-da
3. https://gov.optimism.io/t/upgrade-proposal-5-ecotone-network-upgrade/7669
4. https://medium.com/@afterdark_labs/how-the-op-stack-works-what-happens-when-you-mint-a-base-day-one-nft-c5abbca37a25
5. https://dune.com/haddis3/optimism-fee-calculator