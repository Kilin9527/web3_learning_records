USDT（Tether USD，中文名“泰达币”）是一种与美元1:1挂钩的**稳定币**​（Stablecoin），由Tether公司于2014年发行。它在加密货币市场中扮演着“数字美元”的角色，旨在解决加密货币价格波动大的问题，提供价值稳定的交易媒介。

USDT 并非仅基于单一区块链，而是在多个公链上发行，以适配不同的生态系统。

- **ERC-20 USDT**：基于以太坊区块链，是最早的版本，应用广泛但受限于以太坊的 Gas 费和速度。

- **TRC-20 USDT**：基于波场（Tron）区块链，转账速度快、手续费低，在波场生态中使用较多。

- **OMNI USDT**：基于比特币区块链的 OMNI 协议，是最早的版本之一，但转账速度慢、手续费高，目前使用较少。

- 其他版本：如基于 Polygon、Avalanche 等公链的 USDT，以适应不同链的 DApp 生态。

#### SafeMath 库

在 USDT 源代码中，直接引入了 solidity 中 SafeMath 库，这个库主要提供整数溢出/下溢的标准方法。

```Solidity
library SafeMath {
    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) {
            return 0;
        }
        uint256 c = a * b;
        assert(c / a == b);
        return c;
    }

    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a / b;
        return c;
    }

    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        assert(b <= a);
        return a - b;
    }

    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        assert(c >= a);
        return c;
    }
}
```

在 Solidity 0.8.0 之后，默认内置会对所有算术运算进行溢出/下溢检查，如果发生溢出/下溢交易会revert。

#### Ownerable 合约

Ownable 合约，用于实现合约的所有权控制。这种模式允许将某些功能限制为仅合约所有者可调用，是智能合约开发中的常见设计模式。

```Solidity
contract Ownable {
    address public owner;

    function Ownable() public {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }

    function transferOwnership(address newOwner) public onlyOwner {
        if (newOwner != address(0)) {
            owner = newOwner;
        }
    }
}
```

- `owner`当前合约所有者，声明为 public 会自动生成一个 `getter`函数。

- `Ownable()`构造函数，仅在合约部署时调用一次，将部署者的地址设为 owner。

    - 在 Solidity 0.5.0 之后，构造函数使用 `constructor`关键字，而非合约同名函数。

- `onlyOwner`访问控制修饰器，只允许合约所有者调用。

- `transferOwnership`合约所有权转移。

#### ERC20 抽象合约

下面是两个抽象合约，定义了 ERC20 标准的最基础功能和授权功能。

```Solidity
contract ERC20Basic {
    uint public _totalSupply;
    function totalSupply() public constant returns (uint);
    function balanceOf(address who) public constant returns (uint);
    function transfer(address to, uint value) public;
    event Transfer(address indexed from, address indexed to, uint value);
}

contract ERC20 is ERC20Basic {
    function allowance(address owner, address spender) public constant returns (uint);
    function transferFrom(address from, address to, uint value) public;
    function approve(address spender, uint value) public;
    event Approval(address indexed owner, address indexed spender, uint value);
}
```

- ERC20Basic 是 ERC20 标准的最小化接口，仅包含最基础的功能，不涉及授权机制。

- ERC20 继承 ERC20Basic 标准并扩展授权相关的函数。

#### BasicToken 合约

BasicToken 定义了手续费机制。

```Solidity
contract BasicToken is Ownable, ERC20Basic {
    using SafeMath for uint;

    mapping(address => uint) public balances;

    uint public basisPointsRate = 0;
    uint public maximumFee = 0;

    modifier onlyPayloadSize(uint size) {
        require(!(msg.data.length < size + 4));
        _;
    }

    function transfer(address _to, uint _value) public onlyPayloadSize(2 * 32) {
        uint fee = (_value.mul(basisPointsRate)).div(10000);
        if (fee > maximumFee) {
            fee = maximumFee;
        }
        uint sendAmount = _value.sub(fee);
        balances[msg.sender] = balances[msg.sender].sub(_value);
        balances[_to] = balances[_to].add(sendAmount);
        if (fee > 0) {
            balances[owner] = balances[owner].add(fee);
            Transfer(msg.sender, owner, fee);
        }
        Transfer(msg.sender, _to, sendAmount);
    }

    function balanceOf(address _owner) public constant returns (uint balance) {
        return balances[_owner];
    }
}
```

- `SafeMath`：防止整数溢出 / 下溢的数学库，所有`uint`操作都通过它安全执行。

- `basisPointsRate`：转账手续费率（基点，1 基点 = 0.01%）。

- `maximumFee`：单笔转账的最高手续费上限。

- `onlyPayloadSize`：验证调用数据长度是否符合预期，防止因参数不足导致的攻击（如短地址攻击）。msg.data 前4个字节是函数选择器，size则是调用函数的参数的长度，比如这里是一个 address 和 uint，address 虽然只占用20个字节，但因calldata的数据对齐规则，会被填充到32字节，uint占用32个字节，所以总数为32 * 2。

    - ​**短地址攻击原理**​：
攻击者可能构造一个非标准长度的 `calldata`（例如仅 52 字节：`4 字节选择器 + 20 字节地址 + 28 字节数值`）。
EVM 在处理时会自动补零到 64 字节，导致数值高位被填充，实际转账金额被放大（如 `1 wei` 变为 `2^224 wei`）

- `transfer`函数负责转账并扣除手续费：

    1. 计算手续费：fee = 转账金额 * 费率 / 10000。

    2. 若手续费超过上限，则设为上限值。

    3. 实际转账金额 = 转账金额 - 手续费。

    4. 扣减发送者余额（全额）。

    5. 增加接收者余额（扣除手续费后的金额）。

    6. 若手续费大于 0，将手续费转给合约所有者，并触发手续费转账事件。

    7. 触发主转账事件。

- `balanceOf`获取目标地址的代币余额。

#### StandardToken 合约

```Solidity
contract StandardToken is BasicToken, ERC20 {

    mapping (address => mapping (address => uint)) public allowed;

    uint public constant MAX_UINT = 2**256 - 1;

    function transferFrom(address _from, address _to, uint _value) public onlyPayloadSize(3 * 32) {
        var _allowance = allowed[_from][msg.sender];

        uint fee = (_value.mul(basisPointsRate)).div(10000);
        if (fee > maximumFee) {
            fee = maximumFee;
        }
        if (_allowance < MAX_UINT) {
            allowed[_from][msg.sender] = _allowance.sub(_value);
        }
        uint sendAmount = _value.sub(fee);
        balances[_from] = balances[_from].sub(_value);
        balances[_to] = balances[_to].add(sendAmount);
        if (fee > 0) {
            balances[owner] = balances[owner].add(fee);
            Transfer(_from, owner, fee);
        }
        Transfer(_from, _to, sendAmount);
    }

    function approve(address _spender, uint _value) public onlyPayloadSize(2 * 32) {
        require(!((_value != 0) && (allowed[msg.sender][_spender] != 0)));

        allowed[msg.sender][_spender] = _value;
        Approval(msg.sender, _spender, _value);
    }

    function allowance(address _owner, address _spender) public constant returns (uint remaining) {
        return allowed[_owner][_spender];
    }
}
```

- `allowed`：双重映射，记录授权信息（`owner`允许`spender`动用的代币额度）。

- `MAX_UINT`：常量，表示 uint256 的最大值（用于无限授权）。

- `transferFrom`：允许`spender`从`_from`账户转账到`_to`账户（需提前授权）。

    1. 获取`_from`对`msg.sender`的授权额度`_allowance`。

    2. 计算手续费（逻辑同`BasicToken`的`transfer`）。

    3. 若授权额度不是无限（`< MAX_UINT`），则扣减授权额度。

    4. 扣减`_from`账户余额（全额）。

    5. 增加`_to`账户余额（扣除手续费后的金额）。

    6. 若手续费大于 0，转给合约所有者并触发事件。

    7. 触发主转账事件。

- `approve`：设置`_spender`可以从`msg.sender`账户动用的代币额度。

    - 要求当授权额度非零时，必须先将当前额度清零才能重新设置（避免前端确认延迟导致的攻击）。

- `allowance`：返回`_owner`允许`_spender`动用的剩余代币额度。

#### Pausable 合约

定义了一个名为`Pausable`的可暂停合约，它继承自`Ownable`合约并提供了暂停 / 恢复合约功能的机制。这种设计模式在以太坊智能合约中非常常见，用于在紧急情况下（如发现安全漏洞）暂停关键操作，或在维护期间限制合约活动。

```Solidity
contract Pausable is Ownable {
  event Pause();
  event Unpause();

  bool public paused = false;

  modifier whenNotPaused() {
    require(!paused);
    _;
  }

  modifier whenPaused() {
    require(paused);
    _;
  }

  function pause() onlyOwner whenNotPaused public {
    paused = true;
    Pause();
  }

  function unpause() onlyOwner whenPaused public {
    paused = false;
    Unpause();
  }
}
```

- `Pausable`继承自`Ownable`，因此只有合约所有者（`owner`）可以调用暂停 / 恢复函数。

- 通过`paused`状态变量控制合约的暂停 / 运行状态。

- 提供`whenNotPaused`和`whenPaused`两个修饰器，用于限制函数调用条件。

- 实现`pause()`和`unpause()`函数，允许所有者切换合约状态。

#### BlackList 合约

扩展代币合约的黑名单管理能力，实现对恶意地址的资产冻结与销毁。

```Solidity
contract BlackList is Ownable, BasicToken {
    function getBlackListStatus(address _maker) external constant returns (bool) {
        return isBlackListed[_maker];
    }

    function getOwner() external constant returns (address) {
        return owner;
    }

    mapping (address => bool) public isBlackListed;
    
    function addBlackList (address _evilUser) public onlyOwner {
        isBlackListed[_evilUser] = true;
        AddedBlackList(_evilUser);
    }

    function removeBlackList (address _clearedUser) public onlyOwner {
        isBlackListed[_clearedUser] = false;
        RemovedBlackList(_clearedUser);
    }

    function destroyBlackFunds (address _blackListedUser) public onlyOwner {
        require(isBlackListed[_blackListedUser]);
        uint dirtyFunds = balanceOf(_blackListedUser);
        balances[_blackListedUser] = 0;
        _totalSupply -= dirtyFunds;
        DestroyedBlackFunds(_blackListedUser, dirtyFunds);
    }

    event DestroyedBlackFunds(address _blackListedUser, uint _balance);

    event AddedBlackList(address _user);

    event RemovedBlackList(address _user);
}
```

- `getBlackListStatus`：查询功能。constant 等同于 view，只查询状态，不修改数据。

- `getOwner`：返回合约所有者。

- `addBlackList`和`removeBlackList`：添加/删除黑名单，同时触发事件。

- `destroyBlackFunds`：销毁黑名单地址的代币。

    - `require`校验：确保操作的是黑名单地址，避免误操作。

    - 余额清零：直接将`balances[_blackListedUser]`设为 0。

    - 扣减总供给：`_totalSupply -= dirtyFunds`。

    - 事件通知：通过`DestroyedBlackFunds`记录销毁的地址和数量。

#### UpgradedStandardToken 升级合约接口

用于实现代币合约升级后的兼容接口。这些接口允许旧版本合约通过调用新版本合约的方法来执行操作，从而实现平滑升级。

```Solidity
contract UpgradedStandardToken is StandardToken{
    function transferByLegacy(address from, address to, uint value) public;
    function transferFromByLegacy(address sender, address from, address spender, uint value) public;
    function approveByLegacy(address from, address spender, uint value) public;
}
```

- `transferByLegacy`：代理转账函数，允许旧合约通过新版本合约执行转账操作。

- `transferFromByLegacy`：代理授权转账函数，允许旧合约通过新版本合约执行授权转账操作。

- `approveByLegacy`：代理授权函数，允许旧合约通过新版本合约设置授权额度。

- **设计目的**：

    1. **合约升级兼容性**：
当代币合约需要升级时（如修复漏洞、添加新功能），用户可能已将代币存入旧合约（如交易所、钱包）。通过`UpgradedStandardToken`，旧合约可以调用新版本合约的方法，确保用户资产安全迁移。

    2. **无缝过渡机制**：
用户无需主动操作，旧合约会自动将操作转发到新合约，实现 “无感升级”。

    3. **权限控制**：
通过校验`msg.sender`是否为旧合约地址，确保只有旧合约能调用这些代理函数，防止恶意调用。

    > 使用 Legacy 接口是在旧合约中显示调用新合约，实现简单，但会造成存储耦合，迁移成本高。推荐使用最新的 OpenZeppelin 的 UUPS 代理。

#### TetherToken 合约

```Solidity
contract TetherToken is Pausable, StandardToken, BlackList {
    string public name;
    string public symbol;
    uint public decimals;
    address public upgradedAddress;
    bool public deprecated;

    function TetherToken(uint _initialSupply, string _name, string _symbol, uint _decimals) public {
        _totalSupply = _initialSupply;
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        balances[owner] = _initialSupply;
        deprecated = false;
    }

    function transfer(address _to, uint _value) public whenNotPaused {
        require(!isBlackListed[msg.sender]);
        if (deprecated) {
            return UpgradedStandardToken(upgradedAddress).transferByLegacy(msg.sender, _to, _value);
        } else {
            return super.transfer(_to, _value);
        }
    }

    function transferFrom(address _from, address _to, uint _value) public whenNotPaused {
        require(!isBlackListed[_from]);
        if (deprecated) {
            return UpgradedStandardToken(upgradedAddress).transferFromByLegacy(msg.sender, _from, _to, _value);
        } else {
            return super.transferFrom(_from, _to, _value);
        }
    }

    function balanceOf(address who) public constant returns (uint) {
        if (deprecated) {
            return UpgradedStandardToken(upgradedAddress).balanceOf(who);
        } else {
            return super.balanceOf(who);
        }
    }

    function approve(address _spender, uint _value) public onlyPayloadSize(2 * 32) {
        if (deprecated) {
            return UpgradedStandardToken(upgradedAddress).approveByLegacy(msg.sender, _spender, _value);
        } else {
            return super.approve(_spender, _value);
        }
    }

    function allowance(address _owner, address _spender) public constant returns (uint remaining) {
        if (deprecated) {
            return StandardToken(upgradedAddress).allowance(_owner, _spender);
        } else {
            return super.allowance(_owner, _spender);
        }
    }

    function deprecate(address _upgradedAddress) public onlyOwner {
        deprecated = true;
        upgradedAddress = _upgradedAddress;
        Deprecate(_upgradedAddress);
    }

    function totalSupply() public constant returns (uint) {
        if (deprecated) {
            return StandardToken(upgradedAddress).totalSupply();
        } else {
            return _totalSupply;
        }
    }

    function issue(uint amount) public onlyOwner {
        require(_totalSupply + amount > _totalSupply);
        require(balances[owner] + amount > balances[owner]);

        balances[owner] += amount;
        _totalSupply += amount;
        Issue(amount);
    }

    function redeem(uint amount) public onlyOwner {
        require(_totalSupply >= amount);
        require(balances[owner] >= amount);

        _totalSupply -= amount;
        balances[owner] -= amount;
        Redeem(amount);
    }

    function setParams(uint newBasisPoints, uint newMaxFee) public onlyOwner {
        require(newBasisPoints < 20);
        require(newMaxFee < 50);

        basisPointsRate = newBasisPoints;
        maximumFee = newMaxFee.mul(10**decimals);

        Params(basisPointsRate, maximumFee);
    }

    event Issue(uint amount);
    event Redeem(uint amount);
    event Deprecate(address newAddress);
    event Params(uint feeBasisPoints, uint maxFee);
}
```

- TetherToken 遵循了 ERC20 标准，同时继承自以下父合约：

    - **Pausable**：提供暂停功能，允许合约所有者在必要时暂停代币的转移操作。

    - **StandardToken**：实现了标准的 ERC20 代币功能，如转账、授权等。

    - **BlackList**：提供黑名单功能，允许将特定地址列入黑名单，禁止其进行交易。

- 状态变量：

    - `name`、`symbol`、`decimals`：代币的名称、符号和小数位数。

    - `upgradedAddress`：升级后的合约地址，用于合约升级。

    - `deprecated`：标记当前合约是否已被弃用。

    - `_totalSupply`：代币总供应量（继承自 StandardToken）。

    - `balances`：地址余额映射（继承自 StandardToken）。

    - `isBlackListed`：黑名单地址映射（继承自 BlackList）。

- 事件：

    - `Issue(uint amount)`：记录代币增发。

    - `Redeem(uint amount)`：记录代币销毁。

    - `Deprecate(address newAddress)`：记录合约升级。

    - `Params(uint feeBasisPoints, uint maxFee)`：记录费用参数变更。

- 构造函数：`TetherToken`通过构造函数初始化代币的基本信息，包括总供应量、名称、符号和小数位数，并将初始代币分配给合约创建者。

- 合约中的 ERC20 具体实现：

    - 主要功能：

        - `transfer`：转账。

        - `transferFrom`：第三方转账。

        - `balanceOf`：查询余额。

        - `approve`：授权给第三方。

        - `allowance`：已授权的代币剩余额度。

        - `totalSupply`：总代币数量。

    - 限制使用条件：

        - 合约未被暂停（`whenNotPaused`修饰符）。

        - 发送者不在黑名单中。

    - 升级逻辑：

        - 如果合约已被弃用（`deprecated`为 true），则调用新合约的`transferByLegacy`方法。

        - 否则，调用父合约的`transfer`方法。

- 合约升级函数：`deprecate`。

    - 只有合约所有者可以调用此函数。

    - 将`deprecated`标记设为 true，并记录新合约地址。

    - 触发`Deprecate`事件。

    - 升级后，所有代币操作将被重定向到新合约。

- 增发和销毁代币：issue 和 redeem。

    - 增发（issue）：仅合约所有者可调用，增加代币总供应量并将新代币分配给所有者。

    - 销毁（redeem）：仅合约所有者可调用，减少代币总供应量和所有者余额。

- 费用参数设置：`setParams`。

    - 设置转账手续费参数（代码中未显示完整的费用扣除逻辑）。

    - 限制手续费比例和最大费用上限。

