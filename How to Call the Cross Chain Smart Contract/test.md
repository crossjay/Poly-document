# How to Call the Cross Chain Smart Contract



## Interface Requirements

To implement cross chain features for any chain, say Ethereum, there are two kinds of contracts that need to be deployed-

1. Block header synchronization contract: This contract maintains the record of block headers of the relay chain on this chain. These block headers serve as means to verify cross chain transactions.
2. Cross chain management contract: Every chain can have no more than one management contract. It creates the cross chain transactions that are transferred to the relay chain. All the service contracts that contain the business logic need to communicate with the management contract.

The interface methods that need to implemented by the respective contracts are as follows:

### Block Header Synchronization Contract

| Interface Method      | Description                                                  |
| --------------------- | ------------------------------------------------------------ |
| **SyncGenesisHeader** | Synchronizes the relay chain's genesis block header (or another block header where a change in block generation cycle occurred), method is invoked one time only when the contract is initialized, stores and processes the genesis block header, fetches the consensus node info of the relay chain, please refer to the [code](https://github.com/ontio/ontology/tree/master/smartcontract/service/native/cross_chain/header_sync) for more details |
| **SyncBlockHeader**   | Consistently synchronizes block cycle change and cross chain transaction block headers from the relay chain, relayer uses this interface method to synchronize block headers, stores and processes block headers, fetches the consensus node info if block generation cycle changes, please refer to the [code](https://github.com/ontio/ontology/tree/master/smartcontract/service/native/cross_chain/header_sync) for more details |

### Cross Chain Management Contract

| Interface Method             | Description                                                  |
| ---------------------------- | ------------------------------------------------------------ |
| **crossChain**               | Creates cross chain transactions, invoked by service contracts when a cross chain function is carried out in the logic, transaction includes unique chain ID, transaction is recorded in the merkle tree, please refer to the [code](https://github.com/polynetwork/eth-contracts/tree/master/contracts/core/cross_chain_manager/logic) for more details |
| **verifyHeaderAndExecuteTx** | Fetches and processes cross chain transactions, invoked by the relayer when fetching transactions and merkle proofs, finds the merkle root of a transaction based on the block height (in the block header), verifies the legitimacy of transaction using the transaction parameters, invokes the service contract on the target chain, please refer to the [code](https://github.com/polynetwork/eth-contracts/tree/master/contracts/core/cross_chain_manager/logic) for details |

## Cross Chain Interaction Between Chains

<div align=center><img width="800" height="570" src="resources/ark.png"/></div>

The figure above illustrates the cross chain interaction between chain A to chain B. The user sends a cross chain request from chain A by invoking a dApp's cross chain interface, and on the target chain B the dApp's smart contract executes the necessary logic to produce the final result. Chain A and B implement the two contracts and other necessary interfaces, and anyone can develop an infrastructure for dApps around the cross chain management contract. The contracts deployed on chain A and chain B make up a complete cross chain dApp.

The complete process flow from chain A to chain B is as follows:

- The user invokes the service contract on chain A, which then in turn invokes the cross chain management contract. The management contract transfers the parameters to the target chain and a cross chain transaction is created by management contract which is sent to the target chain based on block generation on chain A;
- Since there is no means of automatic data exchange between two chains, a **relayer** needs to be set up to transfer block header details from chain A to the relay chain's block header synchronization contract. It also fetches the management contract's response event from chain A which encapsulates the parameters passed by the user, and also fetches the merkle proof. Next, it groups this information together and sends it to the cross chain management contract.
- The management contract fetches the block headers from chain A, verifies whether or not the cross chain parameters and the proof are valid, and then transmits the necessary information to chain B in the form of an event;
- Chain B's relayer transfers the relay chain's block headers to chain B's block header synchronization contract. The relevant chain B cross chain transaction parameters and respective merkle proofs are fetched from the ledger records of the relay chain and transmitted to chain B's cross chain management contract;
- The management contract of chain B determines the legitimacy of the cross chain transaction information and then invokes the relevant target contract and completes the cross chain contract invocation;

There are two different merkle proofs that are transferred to the relay chain:

1. The merkle proof that is used to verify the legitimacy of cross chain transactions from chain A
2. The merkle proof that is used to ensure that a transaction has been created and has occurred on the relay chain

These merkle proofs help establish a trust mechanism for the cross chain ecosystem. Any chain can join the cross chain ecosystem by setting up the communication interface with the relay chain.

## Business Logic Contract Example 

This part provides an example of business logic smart contract, which provides a method to cross-chain transfer token between two chains where already equipped with Cross-Chain Manager Contract and other required contracts mentioned above. Here Chain A and B are still represent the source chain and target chain.

#### LockProxy.sol

##### Pre-set & bind assets:

```solidity
pragma solidity ^0.5.0;

import "./../../libs/ownership/Ownable.sol";

contract LockProxy is Ownable {
		address public managerProxyContract;
    mapping(uint64 => bytes) public proxyHashMap;
    mapping(address => mapping(uint64 => bytes)) public assetHashMap;
    
    function setManagerProxy(address ethCCMProxyAddr) onlyOwner public {
        managerProxyContract = ethCCMProxyAddr;
        emit SetManagerProxyEvent(managerProxyContract);
    }
    
    function bindProxyHash(uint64 toChainId, bytes memory targetProxyHash) onlyOwner public returns (bool) {
        proxyHashMap[toChainId] = targetProxyHash;
        emit BindProxyEvent(toChainId, targetProxyHash);
        return true;
    }
    
    function bindAssetHash(address fromAssetHash, uint64 toChainId, bytes memory toAssetHash) onlyOwner public returns (bool) {
        assetHashMap[fromAssetHash][toChainId] = toAssetHash;
        emit BindAssetEvent(fromAssetHash, toChainId, toAssetHash, getBalanceFor(fromAssetHash));
        return true;
    }
}
```

- Since cross-chain transaction processed by Cross-Chain Manager (CCM) Contract, user not only needs to set Cross-Chain Manager Proxy (CCMP) address on source chain, which works as the proxy of CCM contract, but also needs to bind CCMP contract on target chain to LockProxy contract. 
- Both on Chain A and B, the user needs to bind the asset contract to LockProxy smart contract and the target chain id (here for Chain A, Chain B is the target chain), so that the LockProxy contract can maintain mappings(making connections) from asset contract address on source chain and that on target chain with target chain id. After finishing setting all above, LockProxy contract will work properly as the business logic. Here we go!

##### Cross-Chain transaction:

```solidity
 /* 
  *  @param fromAssetHash     The asset address in current chain, uniformly named as `fromAssetHash`
  *  @param toChainId         The target chain id                         
  *  @param toAddress         The address in bytes format to receive same amount of tokens in target chain 
  *  @param amount            The amount of tokens to be crossed from ethereum to the chain with chainId
  */
  
    function lock(address fromAssetHash, uint64 toChainId, bytes memory toAddress, uint256 amount) public payable returns (bool) {
        require(amount != 0, "amount cannot be zero!");
        require(_transferToContract(fromAssetHash, amount), "transfer asset from fromAddress to lock_proxy contract  failed!");
        
        bytes memory toAssetHash = assetHashMap[fromAssetHash][toChainId];
        require(toAssetHash.length != 0, "empty illegal toAssetHash");

        TxArgs memory txArgs = TxArgs({
            toAssetHash: toAssetHash,
            toAddress: toAddress,
            amount: amount
        });
        bytes memory txData = _serializeTxArgs(txArgs);
        
        IEthCrossChainManagerProxy eccmp = IEthCrossChainManagerProxy(managerProxyContract);
        address eccmAddr = eccmp.getEthCrossChainManager();
        IEthCrossChainManager eccm = IEthCrossChainManager(eccmAddr);
        
        bytes memory toProxyHash = proxyHashMap[toChainId];
        require(toProxyHash.length != 0, "empty illegal toProxyHash");
        require(eccm.crossChain(toChainId, toProxyHash, "unlock", txData), "EthCrossChainManager crossChain executed error!");

        emit LockEvent(fromAssetHash, _msgSender(), toChainId, toAssetHash, toAddress, amount);
        
        return true;

    }
```

- This function is meant to be invoked by the user, a certain amount teokens will be locked in the proxy contract the invoker/msg.sender immediately. Then the same amount of tokens will be unloked from target chain proxy contract at the target chain with chainId later;
- The user makes an asset token cross-chain transaction request through the dApp which works in Chain A, LockProxy smart contract gets the transation information which contains the asset contract address on Chain A the target chain id, the target address on Chain B and amount of token to be transfered. By calling the function lock(), LockProxy contract will lock(transfer) the certain amount to asset contract;
- Then the transaction data will be packed, which then in turn invokes the cross chain management contract. The management contract transfers the parameters of transaction data to the target chain and a cross chain transaction is created by management contract which is sent to the target chain based on block generation on Chain A;
- The serialized transaction data, along with the chain id and CCMP contract address of target chain and the method needed to be called on target chain, will be sent through crossChain() in Cross-Chain Manager contract.

```solidity
/*              
 *  @param argsBs            The argument bytes recevied by the ethereum lock proxy contract, need to be deserialized.          
 *                           based on the way of serialization in the source chain proxy contract.
 *  @param fromContractAddr  The source chain contract address
 *  @param fromChainId       The source chain id
 */
    function unlock(bytes memory argsBs, bytes memory fromContractAddr, uint64 fromChainId) onlyManagerContract public returns (bool) 
    {
        TxArgs memory args = _deserializeTxArgs(argsBs);
        require(fromContractAddr.length != 0, "from proxy contract address cannot be empty");
        require(Utils.equalStorage(proxyHashMap[fromChainId], fromContractAddr), "From Proxy contract address error!");
        
        require(args.toAssetHash.length != 0, "toAssetHash cannot be empty");
        address toAssetHash = Utils.bytesToAddress(args.toAssetHash);

        require(args.toAddress.length != 0, "toAddress cannot be empty");
        address toAddress = Utils.bytesToAddress(args.toAddress);

        require(_transferFromContract(toAssetHash, toAddress, args.amount), "transfer asset from lock_proxy contract to toAddress failed!");
        
        emit UnlockEvent(toAssetHash, toAddress, args.amount);
        return true;
    }
```

- After verification through Poly (detailed verification process shown in part Cross Chain Interaction Between Chains), the packed transaction data could be executed on Chain B.
- verifyHeaderAndExecuteTx() in Cross-Chain Manager contract determines the legitimacy of the cross chain transaction information and resolve the parameters of transaction data from the Poly chain transaction merkle proof and crossStateRoot contained in the block header.
- Then call the function unlock() to deserialize the transaction data and unlock (transfer) the certain amount of token to the target address on Chain B and completes the cross chain contract invocation. 

##### Serialize & deserialize transaction data

```solidity
 		function _serializeTxArgs(TxArgs memory args) internal pure returns (bytes memory) {
        bytes memory buff;
        buff = abi.encodePacked(
            ZeroCopySink.WriteVarBytes(args.toAssetHash),
            ZeroCopySink.WriteVarBytes(args.toAddress),
            ZeroCopySink.WriteUint255(args.amount)
            );
        return buff;
    }

    function _deserializeTxArgs(bytes memory valueBs) internal pure returns (TxArgs memory) {
        TxArgs memory args;
        uint256 off = 0;
        (args.toAssetHash, off) = ZeroCopySource.NextVarBytes(valueBs, off);
        (args.toAddress, off) = ZeroCopySource.NextVarBytes(valueBs, off);
        (args.amount, off) = ZeroCopySource.NextUint255(valueBs, off);
        return args;
    }
```

- In the process of contract development, developers will always encounter serialization and deserialization problems, that is, how to save a struct type of data in the database and how to deserialize the byte array read from the database to obtain data of struct type. In the lib, ZeroCopySource.sol and ZeroCopySink.sol offered the interfaces to serialize and deserialize data. 
- When serializing various data types, for fixed-length data (for example: bytes, uint16, uint32, uint64, etc.), directly convert the data into a byte array; for data with variable length, serializing the length is required firstly, and then serialize the data (for example, unsigned integers of unknown size, including uint16, uint32, or uint64, etc.).
- Deserialization is the opposite of serialization. For all serialization methods, there are corresponding deserialization methods. When reading data of a specified type, if you know its length, you can read it directly; for data with an unknown length, read the length first, and then read the content.
