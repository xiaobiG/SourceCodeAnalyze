OpenZeppelin提供了智能合约的三种访问控制模式：Ownable合约、 Roles库和3.0新增的AccessControl合约。

控制对智能合约特定方法的访问权限，对于智能合约的安全性非常重要。 熟悉OpenZeppelin的智能合约库的开发者都知道这个库已经提供了根据 访问等级进行访问限制的选项，其中最常见的就是`Ownable`合约管理的 `onlyOwner`模式，另一个是OpenZeppelin的`Roles`库，它允许合约 在部署前定义多种角色并为每个函数设置规则，以确保`msg.sender`具有 正确的角色。在OpenZeppelin 3.0中又引入了更强大的AccessControl合约， 其定位是一站式访问控制解决方案。

# Ownable

最简单的权限控制，确保msgsender是合约的owner，同时提供了更改合约owner的方法。

```solidity
// SPDX-License-Identifier: MIT
// OpenZeppelin Contracts (last updated v4.7.0) (access/Ownable.sol)

pragma solidity ^0.8.0;

import "../utils/Context.sol";

/**
 * @dev Contract module which provides a basic access control mechanism, where
 * there is an account (an owner) that can be granted exclusive access to
 * specific functions.
 *
 * By default, the owner account will be the one that deploys the contract. This
 * can later be changed with {transferOwnership}.
 *
 * This module is used through inheritance. It will make available the modifier
 * `onlyOwner`, which can be applied to your functions to restrict their use to
 * the owner.
 */
abstract contract Ownable is Context {
    address private _owner;

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    /**
     * @dev Initializes the contract setting the deployer as the initial owner.
     */
    constructor() {
        _transferOwnership(_msgSender());
    }

    /**
     * @dev Throws if called by any account other than the owner.
     */
    modifier onlyOwner() {
        _checkOwner();
        _;
    }

    /**
     * @dev Returns the address of the current owner.
     */
    function owner() public view virtual returns (address) {
        return _owner;
    }

    /**
     * @dev Throws if the sender is not the owner.
     */
    function _checkOwner() internal view virtual {
        require(owner() == _msgSender(), "Ownable: caller is not the owner");
    }

    /**
     * @dev Leaves the contract without owner. It will not be possible to call
     * `onlyOwner` functions anymore. Can only be called by the current owner.
     *
     * NOTE: Renouncing ownership will leave the contract without an owner,
     * thereby removing any functionality that is only available to the owner.
     */
    function renounceOwnership() public virtual onlyOwner {
        _transferOwnership(address(0));
    }

    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`).
     * Can only be called by the current owner.
     */
    function transferOwnership(address newOwner) public virtual onlyOwner {
        require(newOwner != address(0), "Ownable: new owner is the zero address");
        _transferOwnership(newOwner);
    }

    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`).
     * Internal function without access restriction.
     */
    function _transferOwnership(address newOwner) internal virtual {
        address oldOwner = _owner;
        _owner = newOwner;
        emit OwnershipTransferred(oldOwner, newOwner);
    }
}
```



# Roles库

虽然Ownable合约简单易用，但是OpenZeppelin库中的其他合约都是 使用Roles库来实现访问控制。这是因为Roles库比Ownable合约提供 了更多的灵活性。

我们使用using语句引入Roles合约库，来为数据类型增加功能。 Roles库为Role数据类型实现了三个方法。

```java
pragma solidity ^0.5.0;

/**
 * @title Roles
 * @dev Library for managing addresses assigned to a Role.
 */
library Roles {
    struct Role {
        mapping (address => bool) bearer;
    }

    /**
     * @dev Give an account access to this role.
     */
    function add(Role storage role, address account) internal {
        require(!has(role, account), "Roles: account already has role");
        role.bearer[account] = true;
    }

    /**
     * @dev Remove an account's access to this role.
     */
    function remove(Role storage role, address account) internal {
        require(has(role, account), "Roles: account does not have role");
        role.bearer[account] = false;
    }

    /**
     * @dev Check if an account has this role.
     * @return bool
     */
    function has(Role storage role, address account) internal view returns (bool) {
        require(account != address(0), "Roles: account is the zero address");
        return role.bearer[account];
    }
}
```

在代码的开头部分，我们剋有看到Role结构，合约使用该结构定义多个角色 以及其成员。方法`add()`、`remove()`、`has()`则提供了与Role结构交互 的接口。

例如，下面的代码展示了如何使用两个不同的角色 —— `_minters`和`_burners` —— 来为特定的方法设定访问控制规则：

```solidity
pragma solidity ^0.5.0;

import "@openzeppelin/contracts/access/Roles.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20Detailed.sol";

contract MyToken is ERC20, ERC20Detailed {
    using Roles for Roles.Role;

    Roles.Role private _minters;
    Roles.Role private _burners;

    constructor(address[] memory minters, address[] memory burners)
        ERC20Detailed("MyToken", "MTKN", 18)
        public
    {
        for (uint256 i = 0; i < minters.length; ++i) {
            _minters.add(minters[i]);
        }

        for (uint256 i = 0; i < burners.length; ++i) {
            _burners.add(burners[i]);
        }
    }

    function mint(address to, uint256 amount) public {
        // Only minters can mint
        require(_minters.has(msg.sender), "DOES_NOT_HAVE_MINTER_ROLE");

        _mint(to, amount);
    }

    function burn(address from, uint256 amount) public {
        // Only burners can burn
        require(_burners.has(msg.sender), "DOES_NOT_HAVE_BURNER_ROLE");

       _burn(from, amount);
    }
}
```

注意在`mint()`函数中，`require`语句确保交易发起方具有`minter`角色， 即`_minters.has(msg.sender)`。

## AccessControl

Roles库虽然灵活，但也存在一定的局限性。因为它是一个Solidity库， 所以它的数据存储是被引入的合约控制的，而理想的实现应当是让引入 Roles库的合约只需要关心每个方法能实现的访问控制功能。

OpenZeppelin 3.0新增的AccessControl合约被官方称为：

> 可以满足所有身份验证需求的一站式解决方案，它允许你：
>
> 1、轻松定义具有 不同权限的多种角色 
>
> 2、定义哪个账号可以进行角色的授权与回收
>
> 3、 枚举系统中所有的特权账号。

上述的3个特性中，后两点都是Roles库不支持的。看起来OpenZeppelin 正在逐渐实现基于角色的访问控制和基于属性的访问控制这些在传统的 计算安全中非常重要的标准。

```java
pragma solidity ^0.6.0;

import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20, AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");

    constructor() public ERC20("MyToken", "TKN") {
        // Grant the contract deployer the default admin role: it will be able
        // to grant and revoke any roles
        _setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }

    function mint(address to, uint256 amount) public {
        require(hasRole(MINTER_ROLE, msg.sender), "Caller is not a minter");
        _mint(to, amount);
    }

    function burn(address from, uint256 amount) public {
        require(hasRole(BURNER_ROLE, msg.sender), "Caller is not a burner");
        _burn(from, amount);
    }
}
```

`_setupRole()`用于在构造函数中设置初始的角色管理员，以便跳过AccessControl 中的grantRole()进行的检查（因为在创建合约时还没有管理员）。

另外，不再需要像调用库方法那样需要借助于特定的数据类型，例如_minters.has(msg.sender)， 现在角色操作的相关方法可以在子合约中直接调用，例如hasRole(MINTER_ROLE,msg.sender)， 这可以让子合约的代码看起来更加干净，可读性更高。