# Greenfield可借鉴功能点

1.  数据所有权

Greenfield 赋予用户全面的数据所有权管理功能。用户可以将其数据的访问、修改、创建、删除和执行权限**授予个人或组**。在 Greenfield 上，用户可以完全控制自己的数据。

https://greenfield.bnbchain.org/docs/guide/greenfield-blockchain/overview.html#how-greenfield-blockchain-works

2.  数据到期后降级服务

一旦支付账户的BNB用完，与这些支付账户关联的对象将遭受降级下载服务，即下载速度和连接数将受到限制。一旦资金转入支付账户，服务质量便可立即恢复。如果长时间未恢复服务，则SP自行决定清除数据，类似于SP声称停止对某些对象的服务。在这种情况下，数据可能会完全从 Greenfield 中消失。

> 如果用户未能及时续订，则存在存储数据被永久删除的风险。

https://greenfield.bnbchain.org/docs/guide/concept/billing-payment.html#payment-account

3.  网络治理

验证者在网络治理方面也有发言权。他们对与 Greenfield 生态系统未来发展相关的问题进行投票，并根据需要调整各种网络参数。这确保了网络随着时间的推移保持健康和可持续发展，同时满足用户不断变化的需求。

https://greenfield.bnbchain.org/docs/guide/greenfield-blockchain/overview.html#core-of-greenfield-blockchain

4.  搜索和发现

市场应该有一个易于使用的搜索和发现系统来帮助买家找到他们需要的数据，这需要一个强大的索引后端系统。

# Greenfield 功能设计

用户A创建一个group；

将group与A上传的对象object进行绑定，并且指定操作权限，比如read等；

用户A将用户B添加到该group中；

则用户B拥有对object的read权限；

## Group设计

在BSC上管理group的合约接口设计如下：

每个组group对应一个NFT Token，group id 即为相应的 NFT ID；组中成员将对应于ERC1155 token，ERC1155 token ID 和所属组的 group id 一样。

```sol
interface IGroupHub {
    /** 
     * @dev  Query the contract address of group NFT
     * @return The contract address of group token
     * 每个组对应一个NFT Token, group ID 即为相应的NFT ID
     */
    function ERC721Token() external view returns (address);
   /** 
     * @dev  Query the contract address of member NFT
     * @return The contract address of member token
     * The member inside a group  will be mapped as a ERC1155 token on BSC. 
     * The ID of the ERC1155 token is same with the group ID.
     */
    function ERC1155Token() external view returns (address);

   /**
     * @dev create a group and send cross-chain request from BSC to GNFD
     *
     * @param creator The group's owner
     * @param name The group's name
     */
    function createGroup(address creator, string memory name) external payable returns (bool);

   /**
     * @dev create a group and send cross-chain request from BSC to GNFD.
     * Callback function will be called when the request is processed.
     *
     * @param creator The group's owner
     * @param name The group's name
     * @param callbackGasLimit The gas limit for callback function
     * @param extraData Extra data for callback function. The `appAddress` in `extraData` will be ignored.
     * It will be reset as the `msg.sender` all the time.
     */
     function createGroup(
        address creator,
        string memory name,
        uint256 callbackGasLimit,
        CmnStorage.ExtraData memory extraData
    ) external payable returns (bool);

    /**
     * @dev delete a group and send cross-chain request from BSC to GNFD
     *
     * @param id The group's id
     */
    function deleteGroup(uint256 id) external payable returns (bool);

    /**
     * @dev delete a group and send cross-chain request from BSC to GNFD
     * Callback function will be called when the request is processed.
     *
     * @param id The group's id
     * @param callbackGasLimit The gas limit for callback function
     * @param extraData Extra data for callback function. The `appAddress` in `extraData` will be ignored.
     * It will be reset as the `msg.sender` all the time.
     */
    function deleteGroup(uint256 id, uint256 callbackGasLimit, CmnStorage.ExtraData memory extraData) external payable returns (bool);

    /**
     * @dev update a group's member and send cross-chain request from BSC to GNFD
     *
     * @param synPkg Package containing information of the group to be updated
     */
    function updateGroup(GroupStorage.UpdateGroupSynPackage memory synPkg) external payable returns (bool);
/**
     * @dev update a group's member and send cross-chain request from BSC to GNFD
     * Callback function will be called when the request is processed.
     *
     * @param synPkg Package containing information of the group to be updated
     * @param callbackGasLimit The gas limit for callback function
     * @param extraData Extra data for callback function. The `appAddress` in `extraData` will be ignored.
     * It will be reset as the `msg.sender` all the time.
     */
    function updateGroup(
        GroupStorage.UpdateGroupSynPackage memory synPkg,
        uint256 callbackGasLimit,
        CmnStorage.ExtraData memory extraData
    ) external payable returns (bool);
}
```

## Bucket设计

每个Bucket将对应一个NFT，bucket id 即为 NFT token id.

```sol
interface IBucketHub {
   /** 
     * @dev  Query the contract address of bucket NFT
     * @return The contract address of bucket token
     * Each bucket will be mapped as a NFT on BSC. 
     * Bucket ID and NFT token ID are the same.
     */
    function ERC721Token() external view returns (address);

    /**
     * @dev create a bucket and send cross-chain request from BSC to GNFD
     *
     * @param synPkg Package containing information of the bucket to be created
     */
    function createBucket(BucketStorage.CreateBucketSynPackage memory synPkg) external payable returns (bool);

     /**
     * @dev create a bucket and send cross-chain request from BSC to GNFD.
     * Callback function will be called when the request is processed.
     *
     * @param synPkg Package containing information of the bucket to be created
     * @param callbackGasLimit The gas limit for callback function
     * @param extraData Extra data for callback function. The `appAddress` in `extraData` will be ignored.
     * It will be reset as the `msg.sender` all the time.
     */
    function createBucket(
        BucketStorage.CreateBucketSynPackage memory synPkg,
        uint256 callbackGasLimit,
        CmnStorage.ExtraData memory extraData
    ) external payable returns (bool);

   /**
     * @dev delete a bucket and send cross-chain request from BSC to GNFD
     *
     * @param id The bucket's id
     */
    function deleteBucket(uint256 id) external payable returns (bool);
    /**
     * @dev delete a bucket and send cross-chain request from BSC to GNFD.
     * Callback function will be called when the request is processed.
     *
     * @param id The bucket's id
     * @param callbackGasLimit The gas limit for callback function
     * @param extraData Extra data for callback function. The `appAddress` in `extraData` will be ignored.
     * It will be reset as the `msg.sender` all the time.
     */
    function deleteBucket(uint256 id, uint256 callbackGasLimit, CmnStorage.ExtraData memory extraData) external payable returns (bool);
}
```

## Object设计

object也是对应到一个NFT合约中，object id 和 NFT Token ID保持一致。

```sol
interface IObjectHub {
    /** 
     * @dev  Query the contract address of object NFT
     * @return The contract address of object token
     * Each object will be mapped as a NFT on BSC. 
     * Object ID and NFT token ID are the same.
     */
    function ERC721Token() external view returns (address);

   /**
    * @dev delete a object and send cross-chain request from BSC to GNFD
    *
    * @param id, the Id of the object
    */
    function deleteObject(uint256 id) external payable returns (bool);
   /**
    * @dev delete a object and send cross-chain request from BSC to GNFD
    * Callback function will be called when the request is processed
    *
    * @param id, the Id of the object
    * @param callbackGasLimit The gas limit for callback function
    * @param extraData Extra data for callback function. The `appAddress` in `extraData` will be ignored.
    * It will be reset as the `msg.sender` all the time
    */
    function deleteObject(uint256 id, uint256 callbackGasLimit, CmnStorage.ExtraData memory extraData) external payable returns (bool);
}
```

## 权限控制

GreenField 中的资产包括 group、bucket、object ，资产本身的 owner 具有所有权（增加、更改、删除），另外，还规定了额外的权限管理。

### 通用权限控制

Greenfield 中的所有资产（group、bucket、object）都对应为一个 ERC721 token，它们的一般权限管理也对应于 ERC721 的权限管理。比如对于group的权限管理，除了 owner 外，可以指定某一个 groupID 授权给一个账户（approve(address _approved, uint _tokenID)），也可以将自己的所有 group 授权给一个账户（setApprovalForAll(address _operator, bool _approved)）。

### 基于角色的权限控制

当需要对操作粒度进行权限控制，比如允许某一账户创建bucket，但不允许其删除bucket时，则引入了基于角色的权限控制。Greenfield 包含了两种权限控制的角色：

```sol
bytes32 public constant ROLE_CREATE = keccak256("ROLE_CREATE");
bytes32 public constant ROLE_DELETE = keccak256("ROLE_DELETE");
```
