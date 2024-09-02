# FVM用例-DataDAO

存储交易提议（deals）指的是由Filecoin网络里存储提供者获取并存储的数据。一项存储交易提议由存储客户发起，目的是存储数据。

参考这个模板： https://github.com/rk-rishikesh/DataDAO

### DataDAO.sol

```sol
/// @dev Gets a cid and returns Deal details associated with the cid
    mapping(bytes => Deal) public deals;
    /// @dev Gets a cid and provider returns true or false depending upon if its the providers of the cid
    mapping(bytes => mapping(uint64 => bool)) public cidProviders;
```

```sol
struct Deal {
        address proposedBy;
        bytes cidraw;
        uint size;
        uint256 storageFees;
        uint256 dealStartBlockStamp;
        uint256 dealDurationInDays;
        DealState dealState;
}

enum DealState {   
        Proposed,
        Passed,  
        Rejected,       
        Active,          
        Expired
} 
```

主要定义了数据存储交易提案信息，

用户可以先 **创建交易提案**， 调用 `createDealProposal` ，记录数据cid、size、dealDuration；

之后**接受或拒绝交易提案**，调用`approveOrRejectDealProposal`，修改`deal`的`dealState`状态为`Passed`或`Rejected`；

`deal`在Filecoin Network上创建之后，会产生一个`Deal ID`，之后调用`activateDeal(DealID)`**激活`deal`**，修改`deal`的`dealState`状态为`Active`，并设置`dealStartBlockStamp`为当前时刻；

如果`deal`超过`duration`，则状态变为`Expired`；

用户可以为`deals[cid]`授权为存储提供商provider，记录在`cidProviders`中；

### DataDAOExample.sol

```sol
mapping(bytes => mapping(address => uint256)) public fundings;
mapping(bytes => uint256) public dealStorageFees;
mapping(bytes => uint64) public dealClient;
```

定义一个NFT合约地址，拥有该NFT的用户可以加入DAO拥有`MEMBER_ROLE`；

拥有`MEMBER_ROLE`的用户可以创建交易提案，并且设置交易存储费，记录在`dealStorageFees`中；

DAO中的admin（拥有`ADMIN_ROLE`的账户）可以选择接收或拒绝交易提案，接收则意味着授权该数据可以存储在DAO中；

存储提供商存储该笔数据并在Filecoin网络上创建`deal`后，会获得一个`Deal ID`，之后就可以激活deal，并将存储提供商ID记录在`dealClient`中；

当deal过期之后，存储提供商可以取回该cid的rewards，记录在`dealStorageFees`中；
