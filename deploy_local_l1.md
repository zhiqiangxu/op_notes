
# Motivation

Nowadays the ethereum testnet gas is often very high, so it's handy if we can deploy local L1 devnet.

Unfortunately, Anvil from Foundry only supports L1 EL devnet, which is not enough if we also want to play with BLOBs.

However, setting up a L1 CL devnet using Prysm/Lighthouse has complicated configurations (staking setup, deposit contract, etc). 

Luckily, [@Qi](https://x.com/qc_qizhou) found an off-the-shelf local L1 devnet maintained by [@Optimism](https://x.com/Optimism).

# Step

1. `cd monorepo_root`
2. `make devnet-up`
   1. This will create a devnet for L1 and L2;
3. `make devnet-down`
   1. This will stop the L1&L2 network;
4. `docker compose up -d l1 l1-bn l1-vc` under `ops-bedrock` folder.
   1. This will run the L1 network only;
5. In the same folder, you can find a pre-funded key in `op-batcher-key.txt` (and use `cast wallet address` to obtain its address);
6. L1 EL RPC port is `8545` and CL is `5052`.


# Reference

1. https://x.com/qc_qizhou/status/1840995552470937662