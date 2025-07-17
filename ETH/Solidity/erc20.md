ERC-20（Ethereum Request for Comments 20）是以太坊区块链上的代币标准协议，它定义了一套通用的接口规范，使得不同的代币能够在以太坊生态系统中实现互操作性。

### **核心功能**

1. **代币转账**：用户可以将代币从一个地址发送到另一个地址。

2. **余额查询**：查看特定地址持有的代币数量。

3. **授权机制**：允许第三方（如去中心化交易所）代为操作代币，实现更复杂的金融功能。

### 协议要求实现的方法

```Solidity
function name() public view returns (string)
// 返回代币的符号
function symbol() public view returns (string)
// 返回代币使用的小数点位数，默认设定为 18，这和以太坊的 Wei 单位是一样的。
function decimals() public view returns (uint8)
// 返回代币的总供应量，也就是代币的发行总量。
function totalSupply() public view returns (uint256)

// 查询指定地址 _owner 持有的代币余额。
function balanceOf(address _owner) public view returns (uint256 balance)
// 将 _value 数量的代币从调用者的账户转移到目标地址 _to
function transfer(address _to, uint256 _value) public returns (bool success)
// 由授权的第三方来执行代币转移操作，把 _from 地址的 _value 数量代币转移到 _to 地址。
// 此操作需要事先调用 approve 方法完成授权。
function transferFrom(address _from, address _to, uint256 _value) public returns (bool success)
// 允许 _spender 地址从调用者的账户中转移最多 _value 数量的代币。
function approve(address _spender, uint256 _value) public returns (bool success)
// 查询 _spender 地址被 _owner 地址授权可以转移的剩余代币数量。
function allowance(address _owner, address _spender) public view returns (uint256 remaining)

// 当代币发生转移时触发该事件，可用于通知外部应用。
event Transfer(address indexed _from, address indexed _to, uint256 _value)
// 当调用 approve 方法成功时触发，表明授权情况发生了更新。
event Approval(address indexed _owner, address indexed _spender, uint256 _value)
```

### **ERC-20 在 OpenZeppelin 中的实现**

```Solidity
  /**
 * @dev Implementation of the {IERC20} interface.
 *
 * This implementation is agnostic to the way tokens are created. This means
 * that a supply mechanism has to be added in a derived contract using {_mint}.
 *
 * TIP: For a detailed writeup see our guide
 * https://forum.openzeppelin.com/t/how-to-implement-erc20-supply-mechanisms/226[How
 * to implement supply mechanisms].
 *
 * The default value of {decimals} is 18. To change this, you should override
 * this function so it returns a different value.
 *
 * We have followed general OpenZeppelin Contracts guidelines: functions revert
 * instead returning `false` on failure. This behavior is nonetheless
 * conventional and does not conflict with the expectations of ERC-20
 * applications.
 */
abstract contract ERC20 is Context, IERC20, IERC20Metadata, IERC20Errors { ... }
```

- 第二段注释表示：OpenZeppelin 的 ERC20 基础合约（如`ERC20.sol`）仅实现了代币的**转移、授权**等核心功能，但**不包含代币的创建逻辑**。

    - **不能直接部署基础合约：** 如果直接部署 OpenZeppelin 的`ERC20`合约，将无法创建任何代币（总供应量为 0）。

    - **必须在派生合约中添加铸造逻辑：** 需要创建一个继承自`ERC20`的新合约，并在其中使用`_mint`函数实现代币的发行。

        ```Solidity
        contract MyToken is ERC20 {
          constructor() ERC20("MyToken", "MTK") {
            // 在部署时铸造1000个代币到部署者地址
            _mint(msg.sender, 1000 * 10 ** decimals());
          }
          
          function mint(address to, uint256 amount) external onlyOwner {
            _mint(to, amount);
          }
        }
        ```

    - **灵活支持多种供应机制：** 通过这种设计，开发者可以根据需求自定义代币发行方式。

        - 一次性铸造（如示例）。

        - 权限控制的增发（如治理代币）。

        - 算法驱动的动态供应（如算法稳定币）。

    #### 为什么这样设计？

    1. **分离关注点**
ERC20 基础合约专注于代币的核心功能（转账、授权），将发行逻辑留给开发者，符合 “单一职责原则”。

    2. **避免安全风险**
如果基础合约默认包含铸造逻辑（如允许所有人铸造），可能导致代币滥发。通过让开发者自定义，可确保只有授权角色能铸造。

    3. **适配不同场景**
不同代币的发行机制差异很大，这种设计能兼容各种需求。

- 最后一段注释表示：**函数失败时`revert`而非返回`false`**。

    #### **OpenZeppelin 与传统 ERC20** 不同点

    - **传统 ERC20 实现：** 原始的 ERC20 标准（如 EIP-20）建议在操作失败时返回`false`（例如`transfer`和`approve`函数）。

    - **OpenZeppelin 的改进：** OpenZeppelin 选择让函数直接**触发`revert`**（即回滚交易），而非返回`false`。

    #### 为什么这样设计？

    1. **符合现代 Solidity 最佳实践**
Solidity 从 0.4.13 版本开始引入`require`和`revert`，鼓励使用异常处理而非返回布尔值，使代码更清晰。

    2. **减少调用者的判断负担**
返回`false`时，调用者必须显式检查返回值；而`revert`会自动终止执行，避免潜在的逻辑错误。

    3. **与其他合约更好兼容**
许多现代合约（如 Uniswap、Aave）依赖`revert`模式，使用 OpenZeppelin 的 ERC20 可无缝集成。

    #### 实际影响

    ```Solidity
    // 传统方式：需检查返回值
    bool success = token.transfer(to, amount);
    require(success, "Transfer failed");
    
    // 失败时自动revert
    token.transfer(to, amount);
    ```

- **OpenZeppelin 中的 ERC20 实现**

    OpenZeppelin 中的 ERC20 合约是一个抽象合约，它采用多重继承的方式，继承了下面四个父合约/接口：

    - **Context 合约**

        - 提供对调用上下文（如`msg.sender`和`msg.data`）的访问，是 OpenZeppelin 的基础辅助合约。

    - **IERC20 接口**

        - 实现了 ERC20 代币标准的核心功能接口（如`transfer`、`approve`等），定义了代币的基本交互规范。

            ```Solidity
            interface IERC20 {
                event Transfer(address indexed from, address indexed to, uint256 value);
                event Approval(address indexed owner, address indexed spender, uint256 value);
              
                function totalSupply() external view returns (uint256);
                function balanceOf(address account) external view returns (uint256);
                function transfer(address to, uint256 value) external returns (bool);
                function allowance(address owner, address spender) external view returns (uint256);
                function approve(address spender, uint256 value) external returns (bool);
                function transferFrom(address from, address to, uint256 value) external returns (bool);
            }
            ```

    - **IERC20Metadata 接口**

        - 扩展了 ERC20 标准，添加了元数据相关接口（如`name`、`symbol`、`decimals`），用于描述代币信息。

            ```Solidity
            interface IERC20Metadata is IERC20 {
                function name() external view returns (string memory);
                function symbol() external view returns (string memory);
                function decimals() external view returns (uint8);
            }
            ```

    - **IERC20Errors 接口**

        - 定义了 ERC20 操作可能抛出的错误（如`ERC20InsufficientBalance`），通过自定义错误（Custom Errors）增强了错误处理的可读性。

    ### ERC20 的具体实现

    #### 1. 私有变量部分

    ```Solidity
    // 存储每个账户的代币余额
    mapping(address account => uint256) private _balances;
    // 存储授权额度（允许第三方花费的代币数量）
    mapping(address account => mapping(address spender => uint256)) private _allowances;
    // 代币的总供应量，表示当前已发行的代币总量
    uint256 private _totalSupply;
    // 币名称（如 "MyToken"），用于显示和标识代币
    string private _name;
    // 代币符号（如 "MTK"），通常是代币的简短标识，类似股票代码
    string private _symbol;
    ```

    - `_allowances`对象中的`account`是代币持有者的地址。

    - `_allowances`对象中的`spender`是被授权的第三方地址。

    - `_allowances`对象中的`uint256`表示`spender`被允许从`account`转移的代币数量。

    #### 2. 转账功能

    ```Solidity
    function transfer(address to, uint256 value) public virtual returns (bool) {
        address owner = _msgSender();
        _transfer(owner, to, value);
        return true;
    }
    
    function _transfer(address from, address to, uint256 value) internal {
        if (from == address(0)) {
            revert ERC20InvalidSender(address(0));
        }
        if (to == address(0)) {
            revert ERC20InvalidReceiver(address(0));
        }
        _update(from, to, value);
    }
    
    function _update(address from, address to, uint256 value) internal virtual {
        if (from == address(0)) {
            // Overflow check required: The rest of the code assumes that totalSupply never overflows
            _totalSupply += value;
        } else {
            uint256 fromBalance = _balances[from];
            if (fromBalance < value) {
                revert ERC20InsufficientBalance(from, fromBalance, value);
            }
            unchecked {
                // Overflow not possible: value <= fromBalance <= totalSupply.
                _balances[from] = fromBalance - value;
            }
        }
        if (to == address(0)) {
            unchecked {
                // Overflow not possible: value <= totalSupply or value <= fromBalance <= totalSupply.
                _totalSupply -= value;
            }
        } else {
            unchecked {
                // Overflow not possible: balance + value is at most totalSupply, which we know fits into a uint256.
                _balances[to] += value;
            }
        }
        emit Transfer(from, to, value);
    }
    ```

    - `_msgSender()` 是一个虚函数，表示这个方法可以在派生合约中通过 override 重写它。

    - 为什么不直接使用 `msg.sender`？因为有些时候，真正的sender不是 msg.sender，而是其他地址的人。通过使用`_msgSender()`函数，可以方便的在派生合约中统一修改。详见元交易部分。

    - `_transfer`首先校验了两个转账地址不为空。然后调用`_update`方法。

    - `_update`方法是代币转账，铸造和销毁的核心逻辑，它负责更新代币余额和总供应量，并触发相应的事件。

        - **转账**：从一个地址向另一个地址转账。from ≠ 0; to ≠ 0。

        - **铸币**：创建新代币并发送到指定地址，from = 0; to ≠ 0。

        - **销毁**：从指定地址移除代币，from ≠ 0; to = 0。

    - **处理 `from` 为零地址的情况（铸造代币）**

        ```Solidity
        if (from == address(0)) {
          _totalSupply += value;
        }
        ```

        当 `from` 是零地址时，视为铸造新代币。此时直接增加总供应量 `_totalSupply`。

        > 需要检查 `_totalSupply + value` 是否溢出，因为后续代码假设 `_totalSupply` 不会溢出。

    - **处理 `from` 为非零地址的情况（减少余额）**

        ```Solidity
        else {
          uint256 fromBalance = _balances[from];
          if (fromBalance < value) {
            revert ERC20InsufficientBalance(from, fromBalance, value);
          }
          unchecked {
            // Overflow not possible: value <= fromBalance <= totalSupply.
            _balances[from] = fromBalance - value;
          }
        }
        ```

        - 当 `from` 是非零地址时，减少其余额。

        - 验证 `from` 的余额是否足够（防止透支）。

        - 使用 `unchecked` 块优化减法操作，关闭溢出检查（因前置条件 `value <= fromBalance` 已保证安全），节省Gas。

    - **处理 `to` 为零地址的情况（销毁代币）**

        ```Solidity
        if (to == address(0)) {
            unchecked {
                _totalSupply -= value;
            }
        }
        ```

        - 当 `to` 是零地址时，视为销毁代币。此时直接减少总供应量 `_totalSupply`。

        - 使用 `unchecked` 块优化减法操作，因为 `value` 要么是铸造时新增的量（`value <= totalSupply`），要么是从余额中扣除的量（`value <= fromBalance <= totalSupply`），不会溢出。

    - **处理 `to` 为非零地址的情况（增加余额）**

        ```Solidity
        else {
            unchecked {
                _balances[to] += value;
            }
        }
        ```

        - 当 `to` 是非零地址时，增加其余额。

        - 使用 `unchecked` 块优化加法操作。在 ERC20 合约中，`_totalSupply` 存储代币总发行量，其值等于所有地址余额之和，这意味着：任何地址的余额 `_balances[to]` 必然满足 `_balances[to] ≤ _totalSupply`，且新增的转账金额 `value` 必然来自其他地址的余额（或增发逻辑），而 `value` 的最大值不可能超过 `_totalSupply`。

    - **触发 `Transfer` 事件**

        - 无论操作是转账、铸造还是销毁，都触发 `Transfer` 事件，通知外部系统代币状态的变化。

    #### 3. 铸币

    ```Solidity
    function _mint(address account, uint256 value) internal {
        if (account == address(0)) {
            revert ERC20InvalidReceiver(address(0));
        }
        _update(address(0), account, value);
    }
    ```

    - `_mint`铸币方法，检查接收地址是否为零地址（`address(0)`），零地址通常代表 “黑洞” 或未初始化的地址，向其发送代币会导致永久丢失。铸造到零地址没有实际意义，因此被视为无效操作。

    - 调用 `_update` 执行铸造逻辑。

    - `from = address(0)`：表示代币从 “系统” 或 “无主” 地址创建。

    - `to = account`：接收新铸造代币的目标地址。

    - `value`：铸造的代币数量。

    #### 4. 销毁

    ```Solidity
    function _burn(address account, uint256 value) internal {
        if (account == address(0)) {
            revert ERC20InvalidSender(address(0));
        }
        _update(account, address(0), value);
    }
    ```

    - `_burn` 销毁代币，允许合约内部逻辑从指定账户中销毁代币，减少总供应量。这是实现代币销毁（如手续费燃烧、通缩机制）的核心方法。

    - 禁止从零地址销毁代币，零地址不持有代币，从其销毁代币没有意义，因此被视为无效操作。

    - 调用 `_update` 执行销毁逻辑。

    - `from = account`：要销毁代币的源账户。

    - `to = address(0)`：表示代币被发送到 “黑洞” 地址（永久移除）。

    - `value`：销毁的代币数量。

    #### 5. 授权

    ```Solidity
    function allowance(address owner, address spender) public view virtual returns (uint256) {
        return _allowances[owner][spender];
    }
    
    function approve(address spender, uint256 value) public virtual returns (bool) {
        address owner = _msgSender();
        _approve(owner, spender, value);
        return true;
    }
    ```

    - `allowance` 方法：查询授权额度。

    - 查询 `owner` 地址允许 `spender` 地址动用的代币数量。

    - `approve` 方法：设置授权额度。

    - 代币持有者（`msg.sender`）授权 `spender` 地址可以动用其账户中的 `value` 数量代币。

    - 调用后，`spender` 可通过 `transferFrom` 方法转移最多 `value` 数量的代币。

    - **授权机制的用途：**

        - **去中心化交易所（DEX）**：用户授权交易所合约动用代币，实现无需充值的交易。

        - **借贷协议**：用户授权协议合约在需要时清算抵押品。

        - **批量操作**：允许智能合约一次性处理多笔交易。

    另外，在 Solidity 的代码中，还有两个不同的 _approve 函数重载：

    ```Solidity
    function _approve(address owner, address spender, uint256 value) internal {
        _approve(owner, spender, value, true);
    }
    
    function _approve(address owner, address spender, uint256 value, bool emitEvent) internal virtual {
        if (owner == address(0)) {
            revert ERC20InvalidApprover(address(0));
        }
        if (spender == address(0)) {
            revert ERC20InvalidSpender(address(0));
        }
        _allowances[owner][spender] = value;
        if (emitEvent) {
            emit Approval(owner, spender, value);
        }
    }
    ```

    - 下面那个 _approve 可以选择是否在授权时触发 Approval 事件，使用场景：

        - 批量授权操作：假设合约需要原子性地调整多个授权值（如迁移合约），触发多个事件会浪费 gas 且产生冗余日志。

        - 合约内部状态迁移：当合约升级或重构时，可能需要在不通知外部系统的情况下重置授权关系。

        - 预授权（Pre-Approval）：某些协议可能需要预先设置授权值，但不希望外部监听这些变化。

    #### 6. 代币授权额度处理

    ```Solidity
    function _spendAllowance(address owner, address spender, uint256 value) internal virtual {
        uint256 currentAllowance = allowance(owner, spender);
        if (currentAllowance < type(uint256).max) {
            if (currentAllowance < value) {
                revert ERC20InsufficientAllowance(spender, currentAllowance, value);
            }
            unchecked {
                _approve(owner, spender, currentAllowance - value, false);
            }
        }
    }
    ```

    - `_spendAllowance` 用于在代币转账时消耗（减少）授权额度，确保转账操作符合代币所有者预先设定的授权规则。

    - `if (currentAllowance < type(uint256).max) { }` 代币授权额度被设置为这个值时，意味着代币所有者允许被授权者（spender）无限制地使用其代币，无需每次使用后减少授权额度。这种设计主要出于以下考虑：

        - Gas 优化：在以太坊上，修改存储变量（如减少授权额度）会产生 Gas 消耗。设置为最大值后，授权额度无需更新，节省了后续交易的 Gas 成本。

        - 简化逻辑：对于需要频繁转移代币的场景（如去中心化交易所），无限授权避免了每次交易前都需要检查和更新授权额度的复杂性。

        - 原子性保障：在复杂的多步骤交易中，无限授权确保不会因授权额度不足导致交易失败，增强了合约交互的可靠性。

        - **注意风险**：无限授权本质上是将代币控制权完全交给了被授权者，因此仅建议用于可信任的合约（如知名的 DEX 或钱包），否则可能导致代币被盗取。

    - `if (currentAllowance < value)` 有限授权校验，如果当前授权额度不足（小于 `value`），抛出 `ERC20InsufficientAllowance` 错误，附带实际额度和所需额度信息。

    - 使用 `unchecked` 块避免溢出检查，节省gas。

    - 更新授权额度：调用内部函数 `_approve` 更新授权额度为 `currentAllowance - value`。

