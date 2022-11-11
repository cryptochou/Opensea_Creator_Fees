# Opensea Creator Fees

Opensea 前几天推出了链上版税强制执行工具。引发了很大的关注。

因此准备通过这篇文章来解释三个问题：

1. 目前版税是如何收取的。
2. Opensea 准备怎么改革。
3. 版税的强制执行是如何实现的。

## 版税是如何收取的

首先需要确认的是 ERC721 或者 ERC1155 类型的 NFT 合约中是没有收取版税这一功能的。在交易过程中无法强制收取版税。

目前收取版税是通过交易平台进行设置的。

以 Opensea 这个平台为例。具体流程大致如下：

1. 项目方发行了 NFT，也就是将合约部署到链上。
2. Opensea 读取链上数据，检测到有新的 NFT 合约的部署后。会自动为其创建一个 Collection。
3. 项目方使用 NFT 合约的创建者（也就是合约的 Owner）钱包地址登录到 Opensea
4. 项目方打开 NFT 对应 Collection 页面后会看到有个设置按钮，点击之后就可以修改 Collection 对应的信息。其中就包括版税的比率以及收取版税的地址。
5. 用户在购买 NFT 的时候，Opensea 会将总价对应比率的版税截留下来，并定期发送给项目方预留的地址。

通过这个流程可以看出来，版税的收取完全是链下的操作。只有交易者在 Opensea 上挂单之后，后续发生的交易才能在 Opensea 上收到版税。如果用户在其他平台挂单，就要由其他平台来收取版税。

这也就给了其他交易平台的一些操作空间。比如 X2Y2 就一低版税来最为噱头吸引交易者，SudoSwap 甚至不收版税，只收手续费。

## Opensea 准备怎么改

1. 推出链上版税强制执行工具，新发行的 NFT 必须集成这个工具才能在 Opensea 上收取版税。
2. 集成了工具的 NFT 无法在一些平台进行交易。


## 如何实现

代码仓库地址：[https://github.com/ProjectOpenSea/operator-filter-registry](https://github.com/ProjectOpenSea/operator-filter-registry)

其实这个工具与其说是链上版税强制执行工具，不如说是黑名单工具。

代码中有下面这句话

> Borrows heavily from the QQL BlacklistOperatorFilter contract:
> https://github.com/qql-art/contracts/blob/main/contracts/BlacklistOperatorFilter.sol

可以看出起原理与 QQL 黑名单工具原理一样。这个工具的主要作用就是限制部分地址调用 NFT 的转出操作。

这里主要有三个角色需要明确：

1. operator，操作者。指的是操作 NFT 转移的地址。需要加入黑名单的一般是交易所合约中涉及代币转移的合约地址这种 operator。
2. registry，注册表。 对应的合约是 OperatorFilterRegistry。核心代码，主要逻辑都在这一块儿。用于管理注册者。最终检查 Operator 是否在黑名单中的逻辑也是在这个合约中。
3. registrant，注册者。对应的合约是 OperatorFilterer 及其子类。新发布的 NFT 需要继承自该类型的合约。

使用流程如下：

1. 新发布的 NFT 项目的合约必须继承自 OperatorFilterer 及其子类。并重写 transfer 相关的方法。所有 transfer 方法都需要校验 该方法的操作者（operator）。
2. 合约在初始化的时候会作为注册者（registrant），在注册表（registry）注册需要过滤的 operator。
3. 设置过滤规则，过滤的规则有三种设置方式：
   1. 可以选择订阅某个已存在的规则，当订阅的这个规则变化的时候会跟着变化。
   2. 复制某个已经存在的规则，不跟着已存在的规则变化。
   3. 自己定义。
4. NFT 在进行 transfer 的时候，会先去校验这个 transfer 方法的调用者 operator 是否在黑名单中。如果在黑名单中就直接 revert。

### OperatorFilterRegistry

这个合约用于管理登记者（registrant）。保存了 登记者（registrant）的具体过滤信息。项目管理者可以通过它来更新或删除需要过滤的操作者（operator）。

#### 属性

##### 1. EOA_CODEHASH

用于过滤 EOA 账户。 EOA 不能加入到操作者（operator）黑名单中。

```solidity
    bytes32 constant EOA_CODEHASH = keccak256("");
```

##### 2. _filteredOperators

```solidity
    mapping(address => EnumerableSet.AddressSet) private _filteredOperators;
```

以登记者（registrant）也就是 NFT 合约地址为 key 存储该 NFT 所有需要过滤的操作者（operator）的地址信息。

这里有个数据结构 `EnumerableSet.AddressSet`。可以把它等价于一个存放 address 的数组。

##### 3. _filteredCodeHashes

```solidity
    mapping(address => EnumerableSet.Bytes32Set) private _filteredCodeHashes;
```

以登记者（registrant）也就是 NFT 合约地址为 key 存储该 NFT 所有需要过滤的操作者（operator）的代码 hash 信息。

 `EnumerableSet.Bytes32Set` 跟上面的地址数组类似，可以把它等价于一个存放 bytes32 的数组。

##### 4. _registrations

```solidity
    mapping(address => address) private _registrations;
```

以 NFT 合约地址为 key 存储该 NFT 过滤规则所对应的地址。

就是说一个 NFT 的过滤规则可以使用自己的，也能使用别的 NFT 已经存在的规则。

##### 5. _subscribers

```solidity
    mapping(address => EnumerableSet.AddressSet) private _subscribers;
```

key 是 某个 NFT 的地址。value 是一个地址集合，集合里面是所有订阅（subscribe）该 NFT 筛选规则的 NFT 地址。

#### 主要方法

##### 1. isOperatorAllowed

该方法通过校验某个 operator 是否在登记者（registrant）对应的黑名单中来判断操作是否能通过。

```solidity
    function isOperatorAllowed(address registrant, address operator) external view returns (bool) {
        // 
        address registration = _registrations[registrant];
        if (registration != address(0)) {
            EnumerableSet.AddressSet storage filteredOperatorsRef;
            EnumerableSet.Bytes32Set storage filteredCodeHashesRef;
            
            // 根据登记者的地址，查出来所需要过滤的 operator 地址
            filteredOperatorsRef = _filteredOperators[registration];
            // 根据登记者的地址，查出来所需要过滤的 operator 代码 hash
            filteredCodeHashesRef = _filteredCodeHashes[registration];

            if (filteredOperatorsRef.contains(operator)) {
                revert AddressFiltered(operator);
            }
            if (operator.code.length > 0) {
                bytes32 codeHash = operator.codehash;
                if (filteredCodeHashesRef.contains(codeHash)) {
                    revert CodeHashFiltered(operator, codeHash);
                }
            }
        }
        return true;
    }
```

##### 2. register

在注册处注册一个地址。

一般在初始化的时候调用，NFT 作为注册者，将其的地址放到 `_registrations` 中。value 和 key 都是这个地址。表明过滤规则在当前地址对应的规则中。

这时候，过滤规则的内容是空的。后续可以向过滤内容的集合中加入需要过滤的 operator。

```solidity
function register(address registrant) external onlyAddressOrOwner(registrant) {
        if (_registrations[registrant] != address(0)) {
            revert AlreadyRegistered();
        }
        _registrations[registrant] = registrant;
        emit RegistrationUpdated(registrant, true);
    }
```

##### 3. registerAndSubscribe

在注册处注册一个地址并订阅某个已存在的规则。如果规则改变了，这个注册者的规则也会跟着变化。

```solidity
function registerAndSubscribe(address registrant, address subscription) external onlyAddressOrOwner(registrant) {
        // 获取注册者对应的过滤规则地址
        address registration = _registrations[registrant];
        // 已经存在，则报错
        if (registration != address(0)) {
            revert AlreadyRegistered();
        }
        // 不能订阅自己
        if (registrant == subscription) {
            revert CannotSubscribeToSelf();
        }
        // 获取要订阅的规则对应的注册者地址
        address subscriptionRegistration = _registrations[subscription];
        // 如果地址不存在，则该规则不存在
        if (subscriptionRegistration == address(0)) {
            revert NotRegistered(subscription);
        }
        // 规则的注册者地址要与订阅的地址相同
        if (subscriptionRegistration != subscription) {
            revert CannotSubscribeToRegistrantWithSubscription(subscription);
        }
        // 将注册者的过滤规则指向要订阅的那个规则
        _registrations[registrant] = subscription;
        // 将订阅的信息存起来
        _subscribers[subscription].add(registrant);
        emit RegistrationUpdated(registrant, true);
        emit SubscriptionUpdated(registrant, subscription, true);
    }
```

##### 4. registerAndCopyEntries

在注册表上注册一个地址，并从另一个地址复制过滤后规则和codeHashes，无需订阅。

跟上面的方法差不多。不过不再订阅某个规则。过滤规则不跟着变化了。

```solidity
function registerAndCopyEntries(address registrant, address registrantToCopy)
        external
        onlyAddressOrOwner(registrant)
    {
        // 不能从自己的规则中复制
        if (registrantToCopy == registrant) {
            revert CannotCopyFromSelf();
        }
        // 获取注册者对应的过滤规则地址
        address registration = _registrations[registrant];
        if (registration != address(0)) {
            revert AlreadyRegistered();
        }
        // 获取将要复制的规则对应的注册者地址
        address registrantRegistration = _registrations[registrantToCopy];
        // 这个规则对应的注册者必须存在
        if (registrantRegistration == address(0)) {
            revert NotRegistered(registrantToCopy);
        }
        // 设置注册者对应的过滤规则对应地址
        _registrations[registrant] = registrant;
        emit RegistrationUpdated(registrant, true);
        // 将规则复制到当前注册者名下
        _copyEntries(registrant, registrantToCopy);
    }
```

### OperatorFilterer

该合约充当注册者的角色。要想符合 Opensea 规则继续收版税，项目方的 NFT 合约就必须继承自该合约。并且要过滤一些指定的 operator。

> Contract owners may implement their own filtering outside of this registry, or they may use this registry to curate their own lists of filtered operators. However, there are certain contracts that are filtered by the default subscription, and must be filtered in order to be eligible for creator fee enforcement on OpenSea.
> 合同所有者可以在此注册表之外实施他们自己的过滤，或者他们可以使用此注册表来管理他们自己的过滤操作员列表。但是，某些合同会被默认订阅过滤，并且必须过滤才能有资格在 OpenSea 上执行创作者费用。

<table>
<tr>
<th>Name</th>
<th>Address</th>
<th>Network</th>
</tr>

<tr>
<td>Blur.io ExecutionDelegate</td>
<td >
0x00000000000111AbE46ff893f3B2fdF1F759a8A8
</td>
<td >
Ethereum Mainnet
</td>
</tr>

<tr>
<td>LooksRare TransferManagerERC721</td>
<td>0xf42aa99F011A1fA7CDA90E5E98b277E306BcA83e</td>
<td>Ethereum Mainnet</td>
</tr>

<tr>
<td>LooksRare TransferManagerERC1155</td>
<td>0xFED24eC7E22f573c2e08AEF55aA6797Ca2b3A051</td>
<td>Ethereum Mainnet</td>
</tr>

<tr>
<td>X2Y2 ERC721Delegate</td>
<td>0xf849de01b080adc3a814fabe1e2087475cf2e354</td>
<td>Ethereum Mainnet</td>
</tr>

<tr>
<td>X2Y2 ERC1155Delegate</td>
<td>0x024ac22acdb367a3ae52a3d94ac6649fdc1f0779</td>
<td>Ethereum Mainnet</td>
</tr>

<tr>
<td>SudoSwap LSSVMPairRouter</td>
<td>0x2b2e8cda09bba9660dca5cb6233787738ad68329</td>
<td>Ethereum Mainnet</td>
</tr>

</table>

#### 属性

##### 1. operatorFilterRegistry

注册表的地址。

```solidity
IOperatorFilterRegistry constant operatorFilterRegistry =
        IOperatorFilterRegistry(0x000000000000AAeB6D7670E522A718067333cd4E);
```

#### 方法和修饰器

##### 1. 初始化方法

```solidity
    // 初始化的两个参数:
    // 1. subscriptionOrRegistrantToCopy: 将要订阅或者复制过滤规则的注册者地址
    // 2. subscribe: 是否要进行订阅
    constructor(address subscriptionOrRegistrantToCopy, bool subscribe) {
        // If an inheriting token contract is deployed to a network without the registry deployed, the modifier
        // will not revert, but the contract will need to be registered with the registry once it is deployed in
        // order for the modifier to filter addresses.
        // 注册表必须被部署了才会进行注册
        if (address(operatorFilterRegistry).code.length > 0) {
            // 如果是订阅
            if (subscribe) {
                // 将注册信息放到注册表中并订阅指定的过滤信息
                operatorFilterRegistry.registerAndSubscribe(address(this), subscriptionOrRegistrantToCopy);
            } else {
                // 不是订阅且规则对应注册者的地址存在
                if (subscriptionOrRegistrantToCopy != address(0)) {
                    // 将注册信息放到注册表中并复制指定的过滤信息
                    operatorFilterRegistry.registerAndCopyEntries(address(this), subscriptionOrRegistrantToCopy);
                } else {
                    // 都不是，则只注册到注册表中。过滤规则信息没有设置。
                    operatorFilterRegistry.register(address(this));
                }
            }
        }
    }
```

##### 2. `onlyAllowedOperator` 修饰器

所有需要进行限制的方法都需要加上该修饰器。

```solidity
modifier onlyAllowedOperator(address from) virtual {
        // Check registry code length to facilitate testing in environments without a deployed registry.
        // 注册表必须被部署了才会进行校验
        if (address(operatorFilterRegistry).code.length > 0) {
            // Allow spending tokens from addresses with balance
            // Note that this still allows listings and marketplaces with escrow to transfer tokens if transferred
            // from an EOA.
            // 允许 from 与 调用者相同时的操作。
            // 一般这种情况是 EOA 账户 自己转出 NFT 或者是将 NFT 进行托管给交易所，由交易所进行转移
            if (from == msg.sender) {
                _;
                return;
            }
            if (
                !(  // 校验 msg.sender 是否在黑名单中
                    operatorFilterRegistry.isOperatorAllowed(address(this), msg.sender)
                    && // 校验 from 是否在黑名单中
                    operatorFilterRegistry.isOperatorAllowed(address(this), from)
                )
            ) {
                revert OperatorNotAllowed(msg.sender);
            }
        }
        _;
    }
```

这里解释一下为什么要限制 `msg.sender` 和 `from`。

首先看一下 Opensea 要限制的合约是 Blur.io 的 ExecutionDelegate、LooksRare TransferManagerERC721、LooksRare TransferManagerERC1155、X2Y2 ERC721Delegate、X2Y2 ERC1155Delegate、SudoSwap LSSVMPairRouter。

这些合约的一个共同的特点就是负责最终的代币转移


DefaultOperatorFilterer 合约会自动订阅

## 参考

1. https://opensea.io/blog/announcements/on-creator-fees/
2. https://github.com/ProjectOpenSea/operator-filter-registry
3. 