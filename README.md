# ERC777 功能型代币（通证）最佳实践

https://learnblockchain.cn/article/893

https://learnblockchain.cn/2019/09/27/erc777

想必很多同学都已经使用过[ERC20 创建过代币](https://learnblockchain.cn/2018/01/12/create_token/)，或许已经被老板要求在ERC20代币上实现一些附加功能搞的焦头烂额，如果还有选择，一定要选择 ERC777 。

## ERC20 的问题

以下是一个遇到很多次的场景：有一天老板过来找你（开发者），最近存币生息很火，我们也做一个合约吧， 用户打币过来给他计算利息， 看起来是一个很简单的需求，你满口答应说好，结果自己一研究发现，使用 ERC20 标准没办法在合约里记录是谁发过来多少币，从而没法计算利息（因为接收者合约并不知道自己接收到ERC20代币）。

> ERC20 标准下，可以通过一个变通的办法，采用两个交易组合完成，方法是：第1步：先让用户把要转移的金额用 ERC20 的approve 授权的存币生息合约（这步通常称为解锁），第2步：再次让用户调用存币生息合约的计息函数，计息函数中通过 transferFrom 把代币从用户手里转移的合约内，并开始计息。

同样由于**ERC20 标准没有一个转账通知机制**，很多ERC20代币误转到合约之后，再也没有办法把币转移出来，已经有大量的ERC20 因为这个原因被锁死，如[锁死的QTUM](https://etherscan.io/address/0x9a642d6b3368ddc662CA244bAdf32cDA716005BC)，[锁死的EOS](https://etherscan.io/address/0x86fa049857e0209aa7d9e616f7eb3b3b78ecfdb0) 。

另外一个问题是**ERC20 转账时，无法携带额外的信息**，例如：我们有一些客户希望让用户使用 ERC20 代币购买商品，因为转账没法携带额外的信息， 用户的代币转移过来，不知道用户具体要购买哪件商品，从而展加了线下额外的沟通成本。

ERC777很好的解决了这些问题，同时ERC777 也兼容 ERC20 标准。因此强烈建议新开发的代币使用ERC777标准。

ERC777 在 ERC20的基础上定义了 `send(dest, value, data)` 来转移代币， send函数额外的参数用来携带其他的信息，send函数会检查持有者和接收者是否实现了相应的钩子函数，如果有实现（不管是普通用户地址还是合约地址都可以实现钩子函数），则调用相应的钩子函数。

## ERC1820 接口注册表合约

即便是一个普通用户地址，同样可以实现对 ERC777 转账的监听， 听起来有点神奇，其实这是通过 ERC1820 接口注册表合约来是实现的。

> ERC1820 如此的重要，以至于ERC777单独把它拆出来作为一个EIP。

ERC1820 是一个全局的合约，有一个唯一在以太坊链上都相同的合约地址，它总是 `0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24` ，这个合约是通过非常巧妙的方式进行部署的，有兴趣的同学可以阅读[EIP1820文档](https://learnblockchain.cn/docs/eips/eip-1820.html)。

ERC 1820 合约的官方实现代码在[ERC1820文档](https://learnblockchain.cn/docs/eips/eip-1820.html)可以查阅，这里说明合约实现的主要内容。

ERC1820合约提过了两个主要接口：

- setInterfaceImplementer(address _addr, bytes32 _interfaceHash, address _implementer)
  用来设置地址（_addr）的接口（_interfaceHash 接口名称的 keccak256 ）由哪个合约实现（_implementer）。
- getInterfaceImplementer(address _addr, bytes32 _interfaceHash) external view returns (address)
  这个函数用来查询地址（_addr）的接口由哪个合约实现。

setInterfaceImplementer函数会参数信息记录到下面这个interfaces映射里：

```
// 记录 地址(第一个键) 的接口（第二个键）的实现地址（第二个值）
mapping(address => mapping(bytes32 => address)) interfaces;
```

相对应的 getInterfaceImplementer() 通过 interfaces 这个mapping 来获得接口的实现。

ERC777 使用 send转账时会分别在持有者和接收者地址上使用ERC1820 的getInterfaceImplementer函数进行查询，查看是否有对应的实现合约，ERC777 标准规范里预定了接口及函数名称，如果有实现则进行相应的调用。

## ERC777 标准规范

### ERC777 接口

ERC777 为了在实现上可以兼容ERC20，除了查询函数和ERC20一致外，操作接口均采用的独立的命名（避免相同的命令无法分辨是哪个标准），ERC777的接口定义如下，要求所有的ERC777代币合约都必须实现这些接口：

```
interface ERC777Token {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function totalSupply() external view returns (uint256);
    function balanceOf(address holder) external view returns (uint256);

    // 定义代币最小的划分粒度
    function granularity() external view returns (uint256);

    // 操作员 相关的操作（操作员是可以代表持有者发送和销毁代币的账号地址）
    function defaultOperators() external view returns (address[] memory);
    function isOperatorFor(
        address operator,
        address holder
    ) external view returns (bool);
    function authorizeOperator(address operator) external;
    function revokeOperator(address operator) external;

    // 发送代币
    function send(address to, uint256 amount, bytes calldata data) external;
    function operatorSend(
        address from,
        address to,
        uint256 amount,
        bytes calldata data,
        bytes calldata operatorData
    ) external;

    // 销毁代币
    function burn(uint256 amount, bytes calldata data) external;
    function operatorBurn(
        address from,
        uint256 amount,
        bytes calldata data,
        bytes calldata operatorData
    ) external;

    // 发送代币事件
    event Sent(
        address indexed operator,
        address indexed from,
        address indexed to,
        uint256 amount,
        bytes data,
        bytes operatorData
    );

    // 铸币事件
    event Minted(
        address indexed operator,
        address indexed to,
        uint256 amount,
        bytes data,
        bytes operatorData
    );

    // 销毁代币事件
    event Burned(
        address indexed operator,
        address indexed from,
        uint256 amount,
        bytes data,
        bytes operatorData
    );

    // 授权操作员事件
    event AuthorizedOperator(
        address indexed operator,
        address indexed holder
    );

    // 撤销操作员事件
    event RevokedOperator(address indexed operator, address indexed holder);
}
```

接口定义在 [openzeppelin代码库](https://github.com/OpenZeppelin/openzeppelin-contracts) 里找到，路径为：` contracts/token/ERC777/IERC777.sol` 。

### 接口说明与实现约定

所有的ERC777 合约除了必须实现上述接口，还有一些其他的必须遵守的约定（直接导致了ERC777官方文档又长又臭...哭~）。

ERC777 合约必须要通过 ERC1820 注册 `ERC777Token` 接口，这样任何人都可以查询合约是否是ERC777标准的合约，注册方法是: 调用ERC1820 注册合约的 setInterfaceImplementer 方法，参数 _addr 及 _implementer 均是合约的地址，_interfaceHash 是 `ERC777Token` 的 keccak256 哈希值（0xac7fbab5...177054）

如果 ERC777 要实现ERC20标准，还必须通过ERC1820 注册`ERC20Token`接口。

#### ERC777 信息说明函数

name()，symbol()，totalSupply()，balanceOf(address) 和含义和在ERC20 中完全一样。

granularity() 用来定义代币最小的划分粒度（>=1）， 要求必须在创建时设定，之后不可以更改，不管是在铸币、发送还是销毁操作的代币数量，必需是粒度的整数倍。

> granularity 和 ERC20 的 decimals 不一样，decimals用来定义小数位数，decimals 是ERC20 可选函数，为了兼容 ERC20 代币, decimals 函数要求必须返回18。而 granularity 表示的是基于最小位数(内部存储)的划分粒度。例如：0.5个代币存储为 `500,000,000,000,000,000` (0.5 X 10^18)，如果粒度为2，则最小转账单位是2（相对于`500,000,000,000,000,000`）。

#### 操作员

ERC777 定义了一个新的操作员角色，操作员被作为移动代币的地址。 每个地址直观地移动自己的代币，将持有人和操作员的概念分开可以提供更大的灵活性。

> 与ERC20中的 approve 、 transferFrom 不同，其未明确定义批准地址的角色。

此外，ERC777还可以定义默认操作员（默认操作员列表只能在代币创建时定义的，并且不能更改），默认操作员是被所有持有人授权的操作员，这可以为项目方管理代币带来方便，当然认何持有人仍然有权撤销默认操作员。

**操作员相关的函数**：

- defaultOperators(): 获取代币合约默认的操作员列表.
- authorizeOperator(address operator): 设置一个地址作为msg.sender 的操作员，需要触发AuthorizedOperator事件。
- revokeOperator(address operator): 移除 msg.sender 上 operator 操作员的权限， 需要触发RevokedOperator事件。
- isOperatorFor(address operator, address holder)： 是否是某个持有者的操作员。

#### 发送代币

ERC777 发送代币 使用以下两个方法：

```
send(address to, uint256 amount, bytes calldata data) external

function operatorSend(
    address from,
    address to,
    uint256 amount,
    bytes calldata data,
    bytes calldata operatorData
) external
```

operatorSend 可以通过参数`operatorData`携带操作者的信息，发送代币除了执行对应账户的余额加减和触发事件之外，还有**额外的规定**：

1. 如果持有者有通过 ERC1820 注册 `ERC777TokensSender` 实现接口， 代币合约必须调用其 `tokensToSend` 钩子函数。
2. 如果接收者有通过 ERC1820 注册 `ERC777TokensRecipient` 实现接口， 代币合约必须调用其 `tokensReceived` 钩子函数。
3. 如果有 `tokensToSend` 钩子函数，必须在修改余额状态之前调用。
4. 如果有 `tokensReceived` 钩子函数，必须在修改余额状态之后调用。
5. 调用钩子函数及触发事件时， `data` 和 `operatorData`必须原样传递，因为 tokensToSend 和 tokensReceived 函数可能根据这个数据取消转账（触发 `revert`）。

**ERC777TokensSender 接口**定义如下：

```
interface ERC777TokensSender {
    function tokensToSend(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes calldata userData,
        bytes calldata operatorData
    ) external;
}
```

如果持有者希望在转账时收到代币转移通知，就需要在ERC1820合约上注册及实现 `ERC777TokensSender` 接口（稍后有案例介绍）。

有一个地方需要注意: 对于所有的 ERC777 合约， 一个持有者地址只能注册一个ERC777TokensSender接口实现。因此 ERC777TokensSender 实现会被多个ERC777合约调用，在ERC777TokensSender接口的实现合约里， msg.sender 是ERC777合约地址，而不是操作者。

**ERC777TokensRecipient 接口**定义如下：

```
interface ERC777TokensRecipient {
    function tokensReceived(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes calldata data,
        bytes calldata operatorData
    ) external;
}
```

如果接收者希望在转账时收到代币转移通知，就需要在ERC1820合约上注册及实现 `ERC777TokensRecipient` 接口。

如果接收者是一个合约地址， 则必须要注册及实现 `ERC777TokensRecipient` 接口（这样可以防止代币被锁死），如果没有实现，ERC777代币合约必须`revert` 回退交易状态。

#### 铸币与销毁

铸币（挖矿）是产生新币的过程，销毁代币则相反，在ERC20 中，没有明确定义这两个行为，通常会transfer方法和Transfer事件来表达。
ERC777 则定义了代币从铸币、转移到销毁的整个生命周期。

ERC777 没有定义铸币的方法名，只定义了 Minted事件，因为很多代币，是在创建的时候就确定好代币的数量。
如果有需要合约可以自己定义铸币函数，铸币函数在实现时要求：

1. 必须触发Minted事件
2. 发行量需要加上铸币量， 接收者是不为 0 ，且接收者余额加上铸币量。
3. 如果接收者有通过 ERC1820 注册 ERC777TokensRecipient 实现接口， 代币合约必须调用其 tokensReceived 钩子函数。

ERC777 定义了两个函数用于销毁代币 (`burn` 和 `operatorBurn`)，可以方便钱包和dapps有统一的接口交互。`burn` 和 `operatorBurn` 的实现要求：

1. 必须触发Burned事件。
2. 总供应量必须减少代币销毁量， 持有者的余额必须减少代币销毁的数量。
3. 如果持有者通过ERC1820注册ERC777TokensSender 实现，必须调用持有者的tokensToSend钩子函数。

> 注意，零个代币数量的交易（不管是转移、铸币与销毁）也是合法的，同样满足粒度（granularity) 的整数倍，因此需要正确处理。

## ERC777 代币实现

OpenZeppelin 实现了一个 ERC777 基础合约，要实现自己的ERC777代币只需要继承 OpenZeppelin ERC777。想了解 OpenZeppelin 的 ERC777 的实现可阅读[ERC777 源码解析](https://learnblockchain.cn/2019/09/26/erc777-code/)。

如果大家是Truffle开发（或者是Node工程），可以使用以下方式安装 OpenZeppelin 合约库：

```
npm install @openzeppelin/contracts
```

发行一个 2100 个的 LBC7 代币的代码就很简单了：

```js
pragma solidity ^0.5.0;

import "@openzeppelin/contracts/token/ERC777/ERC777.sol";

contract MyERC777 is ERC777 {
    constructor(
        address[] memory defaultOperators
    )
        ERC777("MyERC777", "LBC7", defaultOperators)
        public
    {
        uint initialSupply = 2100 * 10 ** 18;
        _mint(msg.sender, msg.sender, initialSupply, "", "");
    }
}
```

实现主要是两步：通过基类ERC777的构造函数确认代币名称、代号以及默认操作员（可为空），然后调用 _mint 初始化发行量，注意发行量的小数位是固定的18位（和ether保持一致），在合约内部是按小数位保存的，因此发行的币数需要乘上1018。

## 监听代币收款

我们假设有这样一个需求：寺庙要实现了一个功德箱合约接收捐赠，功德箱合约需要记录每位施主的善款金额。这时候就可以通过**实现 ERC777TokensRecipient接口**来完成。代码也很简单：

```js
pragma solidity ^0.5.0;

import "@openzeppelin/contracts/token/ERC777/IERC777Recipient.sol";
import "@openzeppelin/contracts/token/ERC777/IERC777.sol";
import "@openzeppelin/contracts/introspection/IERC1820Registry.sol";

contract Merit is IERC777Recipient {

  mapping(address => uint) public givers;
  address _owner;
  IERC777 _token;

  IERC1820Registry private _erc1820 = IERC1820Registry(0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24);

  // keccak256("ERC777TokensRecipient")
  bytes32 constant private TOKENS_RECIPIENT_INTERFACE_HASH =
      0xb281fc8c12954d22544db45de3159a39272895b169a852b314f9cc762e44c53b;

  constructor(IERC777 token) public {
    _erc1820.setInterfaceImplementer(address(this), TOKENS_RECIPIENT_INTERFACE_HASH, address(this));
    _owner = msg.sender;
    _token = token;
  }

// 收款时被回调
  function tokensReceived(
      address operator,
      address from,
      address to,
      uint amount,
      bytes calldata userData,
      bytes calldata operatorData
  ) external {
    givers[from] += amount;
  }

// 方丈取回功德箱token
  function withdraw () external {
    require(msg.sender == _owner, "no permision");
    uint balance = _token.balanceOf(address(this));
    _token.send(_owner, balance, "");
  }

}
```

功德箱合约在构造时，调用 ERC1820 注册表合约的 setInterfaceImplementer函数 注册ERC777TokensRecipient接口实现（接口的实现是自身），这样在收到代币时，会回调 tokensReceived函数，tokensReceived函数通过givers映射来保存每个施主的善款金额。

> 注意： 如果是在本地的开发者网络环境，可能会没有ERC1820 注册表合约，如果没有需要先部署ERC1820注册表合约，参考[eip-1820 中文文档](https://learnblockchain.cn/docs/eips/eip-1820.html)。

功德箱这个实例仅仅是抛砖引玉，告诉大家如何实现收款时的回调，之后有时间，我写一个完整的存币生息应用。

## 普通账户地址监听代币转出

功德箱合约的例子，收款地址和收款监听是同一个合约， 现在来看看一个普通的用户地址，如何委托一个合约来监听代币的转出。
监听代币的转出可以让持有者对发出去的代币有更多的控制，例如持有者可以设置一些黑名单，禁止操作员对黑名单内账号转账，

本部分的内容请订阅[我的小专栏](https://xiaozhuanlan.com/topic/5920148376)查看。

根据 ERC1820 标准，只有账号的管理者才可以为账号注册接口实现合约，
如果一个合约要为某个地址（或自身）实现某个接口， 则需要实现下面这个接口:

```
interface ERC1820ImplementerInterface {
    /// @notice 指示合约是否为地址 “addr” 实现接口 “interfaceHash”。
    /// @param interfaceHash 接口名称的 keccak256 哈希值
    /// @param addr 为哪一个地址实现接口
    /// @return 只有当合约为地址'addr'实现'interfaceHash'时返回 ERC1820_ACCEPT_MAGIC
    function canImplementInterfaceForAddress(bytes32 interfaceHash, address addr) external view returns(bytes32);
}
```

通过在 `canImplementInterfaceForAddress` 返回 `ERC1820_ACCEPT_MAGIC` 以声明实现了 interfaceHash 对应的接口。在调用ERC1820的 setInterfaceImplementer 函数设置接口实现时，会通过 `canImplementInterfaceForAddress` 检查合约时候实现了接口。

因此实现监听转出会和功德箱合约有些不一样：

```js
pragma solidity ^0.5.0;

import "@openzeppelin/contracts/token/ERC777/IERC777Sender.sol";
import "@openzeppelin/contracts/token/ERC777/IERC777.sol";
import "@openzeppelin/contracts/introspection/IERC1820Registry.sol";
import "@openzeppelin/contracts/introspection/IERC1820Implementer.sol";


contract SenderControl is IERC777Sender, IERC1820Implementer {

  IERC1820Registry private _erc1820 = IERC1820Registry(0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24);
  bytes32 constant private ERC1820_ACCEPT_MAGIC = keccak256(abi.encodePacked("ERC1820_ACCEPT_MAGIC"));

  //    keccak256("ERC777TokensSender")
  bytes32 constant private TOKENS_SENDER_INTERFACE_HASH =
        0x29ddb589b1fb5fc7cf394961c1adf5f8c6454761adf795e67fe149f658abe895;

  mapping(address => bool) blacklist;
  address _owner;

  constructor() public {
    _owner = msg.sender;
  }

  //  account call erc1820.setInterfaceImplementer
  function canImplementInterfaceForAddress(bytes32 interfaceHash, address account) external view returns (bytes32) {
    if (interfaceHash == TOKENS_SENDER_INTERFACE_HASH) {
      return ERC1820_ACCEPT_MAGIC;
    } else {
      return bytes32(0x00);
    }
  }

  function setBlack(address account, bool b) external {
    require(msg.sender == _owner, "no premission");
    blacklist[account] = b;
  }

  function tokensToSend(
      address operator,
      address from,
      address to,
      uint amount,
      bytes calldata userData,
      bytes calldata operatorData
  ) external {
    if (blacklist[to]) {
      revert("ohh... on blacklist");
    }
  }

}
```

合约实现 canImplementInterfaceForAddress 函数，以声明对 ERC777TokensSender 接口的实现，返回 ERC1820_ACCEPT_MAGIC。

函数setBlack用来设置黑名单，它使用一个blacklist映射来管理黑名单， 在 tokensToSend函数的实现里，先检查接收者是否在黑名单内，如果在则revert 回退交易阻止转账。

给发送者账号(假设为A）设置代理合约的方法为：先部署代理合约，获得代理合约地址， 然后用A账号去调用 ERC1820 的 setInterfaceImplementer函数，参数分别是 A的地址、接口的 keccak256 即0x29ddb589b1fb5fc7cf394961c1adf5f8c6454761adf795e67fe149f658abe895 以及 代理合约地址。

-->

通过实现 ERC777TokensSender 和 ERC777TokensRecipient 可以延伸出很多有意思的玩法，各位读者可以自行探索。

## 参考文献

1. [openzeppelin 项目文档](https://docs.openzeppelin.com/contracts/2.x/)
2. [openzeppelin 合约代码库](https://github.com/OpenZeppelin/openzeppelin-contracts)
3. [EIP 165 提案文档](https://learnblockchain.cn/docs/eips/eip-165.html)
4. [EIP 1820 提案文档](https://learnblockchain.cn/docs/eips/eip-1820.html)
5. [EIP 20 代币标准提案文档](https://learnblockchain.cn/docs/eips/eip-20.html)
6. [EIP 777 代币标准提案文档](https://learnblockchain.cn/docs/eips/eip-777.html)
7. [EIP 721 非同质代币提案文档](https://learnblockchain.cn/docs/eips/eip-721.html)

## ERC777到底该不该用？

## 可重入攻击不是ERC777的错

我在去年 9 月写过一篇ERC科普文章：[ERC777 功能型代币（通证）最佳实践](https://learnblockchain.cn/2019/09/27/erc777) ，文章里我推荐新开发的代币使用 ERC777 标准。

> Imtoken 使用 ERC777 发行 imbtc 其实是非常值得称赞的，典型的反面是 USDT （transfer不返回值）坑了多少项目。

周末两天Uniswap 和 Lendf.me 都发生了黑客攻击事件，都是Defi 应用与 ERC777 组合应用导致可重入漏洞，其中导致 Lendf.me 损失抵押资产千万美元。

发生这样的事情，相信是所有从业者不愿意看到的，本文也无意针对Lendf.me，你们也是受害者，只是看到有人甩锅给 ERC777 ，不忍从技术角度说几句公道话。 要把锅全甩给 ERC777 ，是特朗普坏（甩锅给你，只因你太优秀）。

ERC777 是一个好的Token标准， 可以极大的提高Defi 应用的用户体验，通过使用的 Hook 回调机制，在 ERC20 中需要二笔或多笔完成的交易（当然还有其他的特性），而使用ERC777单笔交易就可以完成。

对行业的发展我一直是乐观派， **如果因为本次攻击，拒绝使用ERC777，那一定在开历史倒车**。这次事件挫败了大家对 Defi的信心， 从长远看，我相信会让行业更健康。

## 可重入攻击是怎么发生的？

下面我用一段简洁的代码说明可重入攻击是如何发生的（警告，以下是代码请勿使用），下面是 Defi 应用最常见的逻辑，deposit 函数用来存款，存款时会记录下用户的存款金额，withdraw 函数用来取款，取款在余额的基础上加上一个利率。

```javascript
interface IToken {
  function transfer(address recipient, uint256 amount) external returns (bool);
  function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
}

contract Defi {
  
  IToken token;
  mapping(address => uint) balances;
  
 
  function deposit(uint256 amount) external {
    uint balance = balances[msg.sender] + amount;
  	if(token.transferFrom(msg.sender, this, amount)){
      balances[msg.sender] = balance;
    }
  }
  
  function withdraw() external {
  	if(token.transfer(msg.sender, balances[msg.sender] + 利息)) {
      // 取回后余额设置为 0
      balances[msg.sender] = 0;
    }
  
  }
  
  
}
```

在交互过程中，存在 3 个角色，用户、Defi合约、Token合约， 用户存款和取款的时序图是这样的：

用户Token 合约DeFi合约授权Approve(Defi, 100)存款deposit(100)transferFrom(用户,Defi,100)取款withdraw()transfer(用户,110)用户Token 合约DeFi合约

此时一切运行正常，（经过测试后）用户在一段时间之后可以赎回 110 个 token，开开心心发布上线了。

后来上线了一个 ERC777 代币， ERC777 定义了以下两个hook 接口：

```
interface ERC777TokensSender {
    function tokensToSend(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes calldata userData,
        bytes calldata operatorData
    ) external;
}
interface ERC777TokensRecipient {
    function tokensReceived(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes calldata data,
        bytes calldata operatorData
    ) external;
}
```

用来同时发送者和接收者进行相应的响应，当然发送者和接收者也可以选择不响应（不实现接口）。

ERC777 的转账实现一般类似下面这样：（transfer 和 transferFrom 实现差不多，下面用transfer举例)

```javascript
function transfer(address to, uint256 amount) public returns (bool) {

  if (有发送者接口实现) {
      发送者.tokensToSend(operator, from, to, amount, userData, operatorData);
  }

	_move(from, from, to, amount, "", "");
  
  if (有接收者接口实现) {
      接收者.tokensReceived(operator, from, to, amount, userData, operatorData);
  }
  return true;
}
```

简单来说，就是在更改 发送者 和 接收者余额的前后查看是否需要通知发送者和接收者，大部分情况下，普通账号对普通账号的转账（因为普通一般不会实现接口）和 ERC20 效果上一样的。

如果发送者和接收者实现了ERC777的转账接口， 上面的存款调用时序图就是这样的：

用户Token 合约DeFi合约授权Approve(Defi, 100)存款deposit(100)transferFrom(用户,Defi,100)tokensToSend()tokensReceived()用户Token 合约DeFi合约

在Defi合约调用Token 的transferFrom 时，Token合约会调用 tokensToSend 和 tokenReceived 以便发送者和接收者进行相应的相应。注意这里tokensToSend 由用户实现，tokenReceived 由 Defi 合约实现。

> 这个回调能力做很多有趣的事情，比如： 可以把授权和存款合并为一笔交易，用户直接调用 token 合约的转账，Defi 合约收到转账后，在tokenReceived中完成用户的存款操作。

ERC777 协议没有对用户如何实现tokensToSend 及 tokenReceived 做出规定，Defi合约开发者也不应该对参与方的实现进行任何的假定。 在 Lendf.me 的攻击案例中，黑客用户就是在tokensToSend的实现中，调用了 Defi 合约的 withdraw ，黑客用户合约的代码大概是这样的：

```javascript
contract Hacker {
  
  IToken token;
  IDefi  defi;
  
  
  function hack() external  {
  	token.approve(defi, 100);
  	defi.deposit(100)
  }
  
  function tokensToSend() external {
  	defi.withdraw()	
  }
  
}
```

黑客攻击的时序图如下：

HackerHacker合约Token 合约DeFi合约hack()Approve(Defi, 100)deposit(100)transferFrom(Hacker合约,Defi,100)tokensToSend()withdraw()赎回所有存款tokensReceived()HackerHacker合约Token 合约DeFi合约

注意 tokensToSend() 、 withdraw()和tokensReceived() 函数都是在 transferFrom()中执行的，根据deposit的代码：

```
  function deposit(uint256 amount) external {
    uint balance = balances[msg.sender] + amount;
  	if(token.transferFrom(msg.sender, this, amount)){
      balances[msg.sender] = balance;
    }
  }
```

只要前面 3 个函数没有出错，transferFrom执行成功之后，就重置用户余额（黑客合约）为 100（存款金额）。而实际上黑客已经把所有存款全部取出，从而实现了一次对 Defi 合约的攻击。

大家都没方法控制合约的实现，但是甩锅到 ERC777 对吗？ 那么对于 Defi 开发者，如何避免攻击呢？

## 避免 ERC777 重入攻击

其实可重入攻击一直都存在，OpenZeppelin 也给过解决方案，给 Defi 合约加上重入限制即可。

```javascript
contract Defi {
  bool private _notEntered;
  IToken token;
  mapping(address => uint) balances;
  
  modifier nonReentrant() {
    require(_notEntered, "ReentrancyGuard: reentrant call");
    _notEntered = false;
    _;
    _notEntered = true;
  }
// 这里也应该先更改状态 
  function deposit(uint256 amount) external nonReentrant {
  	if(token.transferFrom(msg.sender, this, amount)){
      balances[msg.sender] = balances[msg.sender] + amount;
    }
  }
  
  function withdraw() external nonReentrant {
  	if(token.transfer(msg.sender, balances[msg.sender] + 利息)) {
      // 取回后余额设置为 0
      balances[msg.sender] = 0;
    }
  }  
}
```

给deposit 和 withdraw 函数加入重入限制后，此时如果在 tokensToSend中调用withdraw就会败而回退交易。很明显在 Defi 合约中可以避免重入攻击。
