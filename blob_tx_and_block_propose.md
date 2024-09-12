
# blob tx流转顺序
1. 首先出现在EL的blob pool，然后通过EL网络广播
   1. 广播时只发tx hash，peer需要的话再根据tx hash拉取带有完整blob的tx，参考[此处](https://github.com/ethereum/go-ethereum/blob/bca6c407098fefc757c263ae2da6aeff719e17ca/eth/handler.go#L489)
      1. 可以从[answerGetPooledTransactions](https://github.com/ethereum/go-ethereum/blob/04bf1c802ffe9dfc34c34b3e666ee15e96b4a203/eth/protocols/eth/handlers.go#L397)处理逻辑看到，根据tx hash返回的tx是从tx pool里拿，对于blob tx hash最终会从blob pool获取tx，而blob pool中的是[带有完整blob](https://github.com/ethereum/go-ethereum/blob/04bf1c802ffe9dfc34c34b3e666ee15e96b4a203/core/txpool/blobpool/blobpool.go#L1217)的tx
2. 然后随着CL的出块流程被打包到payload。

# 出块流程

入口是validator client的[publish_block](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/validator_client/src/block_service.rs#L516)方法，该方法先从beacon node的[produce_block](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/http_api/src/produce_block.rs#L41)方法获取区块，然后签名并提交给beacon node的[publish_block](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/http_api/src/publish_blocks.rs#L48)方法。


[produce_block](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/http_api/src/produce_block.rs#L41)调用流程：

1. 核心调用是[produce_block_with_verification](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/http_api/src/produce_block.rs#L57)
   1. 通过[load_state_for_block_production](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L4110)加载父状态
   2. 通过[produce_block_on_state](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L4120)生成区块
      1. 调用[produce_partial_beacon_block](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L4761)
         1. 核心是调用[get_execution_payload](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L4921)触发EL build block
            1. 如果没通过[prepare_beacon_proposer](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/canonical_head.rs#L1236)提前触发EL打包，则会通过[engine.notify_forkchoice_updated](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/execution_layer/src/lib.rs#L1257)即时触发
            2. 然后通过[engine.api.get_payload](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/execution_layer/src/lib.rs#L1293)获取EL打包的payload
      2. [等待](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L4778)EL返回payload（其中如果包含blob tx，则也会将blob[一并返回](https://github.com/ethereum/go-ethereum/blob/8f7fbdfedcbaca2a2bffb00badc75c03d58052ec/miner/worker.go#L113-L119)。）
      3. 调用[complete_partial_beacon_block](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L4791)生成完整的beacon block


[publish_block](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/http_api/src/publish_blocks.rs#L48)调用流程：

1. 调用[into_gossip_verified_block](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/http_api/src/publish_blocks.rs#L114)对区块和blob做一些基础的验证
2. 调用[process_gossip_blob](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/http_api/src/publish_blocks.rs#L197)对每个blob进行处理并缓存
3. 调用[process_block](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/http_api/src/publish_blocks.rs#L213)
   1. [into_execution_pending_block](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L3014) -> [into_execution_pending_block_slashable](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/block_verification.rs#L1219) 
      1. 通过[verify_kzg_for_rpc_block](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/block_verification.rs#L1230)判断区块是否available
      2. [SignatureVerifiedBlock::into_execution_pending_block_slashable](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/block_verification.rs#L1238) -> [from_signature_verified_components](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/block_verification.rs#L1162)最终会异步调到[notify_new_payload](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/block_verification.rs#L1343)尝试把区块保存到EL
   2. 通过[publish_fn](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L3019)对区块在CL网络进行广播（可以看到没等notify_new_payload执行结束就开始广播了），最终会调到[这里](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/network/src/service.rs#L662)通过libp2p广播。
   3. 调用[into_executed_block](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L3020)方法等待上述notify_new_payload返回，得到ExecutedBlock
   4. 如果区块available，则通过[import_available_block](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L3023)将区块保存到CL；否则调用[check_block_availability_and_import](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L3026)直到区块available才会将区块保存到CL
4. 调用[recompute_head_at_current_slot](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/http_api/src/publish_blocks.rs#L241)计算最新header并将结果同步到EL
   1. 通过[spawn_execution_layer_updates](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/canonical_head.rs#L784) -> [update_execution_engine_forkchoice](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/canonical_head.rs#L1214) -> [notify_forkchoice_updated](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L5824)更新EL的header
   2. spawn_execution_layer_updates最后会通过[prepare_beacon_proposer](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/canonical_head.rs#L1236)为下个slot[提前准备](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L5633)出块预备信息，并且如果当前时间接近出块时间的话，通过[update_execution_engine_forkchoice](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L5699)触发EL的打包。
      1. update_execution_engine_forkchoice会调用[notify_forkchoice_updated](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/beacon_chain/src/beacon_chain.rs#L5824)，它会[取出](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/execution_layer/src/lib.rs#L1449)预备信息并通过[engine.notify_forkchoice_updated](https://github.com/sigp/lighthouse/blob/2a3c709f8ceec4d47c1934446511660c429fe90a/beacon_node/execution_layer/src/lib.rs#L1485)触发EL打包（但不保存）