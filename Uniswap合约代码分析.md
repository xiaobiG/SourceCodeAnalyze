uniswap-v2 版本智能合约部分的代码存放在 [uniswap-v2-core](https://link.segmentfault.com/?enc=mqYnT4yS2oWzb1hAnNjL7A%3D%3D.fUMH0X8O2ys%2FCli6zdIn0IevBKRKqzUlgPC3AwrDvJm%2B1fljOsZhcxRofviF2aSK) 和 [uniswap-v2-periphery](https://link.segmentfault.com/?enc=v32HZWQG9Fp8zek55tzxow%3D%3D.A4RRursBjUgk1iYPYLJdlybZnJMDJa3Y67ydMdAOFUCcXt7Q%2F55cf7QxFUTUZ9YT) 两个仓库

> [[科普]由浅入深理解UniswapV3白皮书](https://learnblockchain.cn/article/3055)

# UniswapV2Pair


```solidity
pragma solidity =0.5.16;

import './interfaces/IUniswapV2Pair.sol';
import './UniswapV2ERC20.sol';

// 一个自定义的Math库，只有两个功能，一个是求两个uint的最小值，另一个是对一个uint进行开方运算。
import './libraries/Math.sol';

// 自定义的数据格式库。在UniswapV2中，价格为两种代币的数量比值，而在Solidity中，对非整数类型支持不好，通常两个无符号整数相除为地板除，会截断。
// 为了提高价格精度，UniswapV2使用uint112来保存交易对中资产的数量，而比值（价格）使用UQ112x112表示，一个代表整数部分，一个代表小数部分。
import './libraries/UQ112x112.sol';

// 标准ERC20接口，在获取交易对合约资产池的代币数量（余额）时使用。
import './interfaces/IERC20.sol';

// 导入factory合约相关接口，主要是用来获取开发团队手续费地址。
import './interfaces/IUniswapV2Factory.sol';

// 有些第三方合约希望接收到代币后进行其它操作，好比异步执行中的回调函数。
// 这里IUniswapV2Callee约定了第三方合约如果需要执行回调函数必须实现的接口格式。当然了，定义了此接口后还可以进行FlashSwap。
import './interfaces/IUniswapV2Callee.sol';

contract UniswapV2Pair is IUniswapV2Pair, UniswapV2ERC20 {
    using SafeMath  for uint;
    using UQ112x112 for uint224;

    // 定义了最小流动性。它是最小数值1的1000倍，用来在提供初始流动性时燃烧掉。
    uint public constant MINIMUM_LIQUIDITY = 10**3;

    // 用来计算标准ERC20合约中转移代币函数transfer的函数选择器。
    // 虽然标准的ERC20合约在转移代币后返回一个成功值，但有些不标准的并没有返回值。
    // 在这个合约里统一做了处理，并使用了较低级的call函数代替正常的合约调用。函数选择器用于call函数调用中。
    // 获取transfer方法的bytecode前四个字节
    bytes4 private constant SELECTOR = bytes4(keccak256(bytes('transfer(address,uint256)')));

    // 用来记录factory合约地址和交易对中两种代币的合约地址。
    address public factory;
    address public token0;
    address public token1;

    // 这三个状态变量记录了最新的恒定乘积中两种资产的数量和交易时的区块（创建）时间
    uint112 private reserve0;           // uses single storage slot, accessible via getReserves
    uint112 private reserve1;           // uses single storage slot, accessible via getReserves
    uint32  private blockTimestampLast; // uses single storage slot, accessible via getReserves

    // 记录交易对中两种价格的累计值。
    uint public price0CumulativeLast;
    uint public price1CumulativeLast;
    // 记录某一时刻恒定乘积中积的值，主要用于开发团队手续费计算。
    uint public kLast; // reserve0 * reserve1, as of immediately after the most recent liquidity event

    uint private unlocked = 1;
    modifier lock() {
        require(unlocked == 1, 'UniswapV2: LOCKED');
        unlocked = 0;
        _;
        unlocked = 1;
    }

    // 用于获取两个代币在池子中的数量和最后更新的时间
    function getReserves() public view returns (uint112 _reserve0, uint112 _reserve1, uint32 _blockTimestampLast) {
        _reserve0 = reserve0;
        _reserve1 = reserve1;
        _blockTimestampLast = blockTimestampLast;
    }

    // 使用call函数进行代币合约transfer的调用（使用了函数选择器）。
    // 注意，它检查了返回值（首先必须调用成功，然后无返回值或者返回值为true）。
    function _safeTransfer(address token, address to, uint value) private {
        (bool success, bytes memory data) = token.call(abi.encodeWithSelector(SELECTOR, to, value));
        require(success && (data.length == 0 || abi.decode(data, (bool))), 'UniswapV2: TRANSFER_FAILED');
    }

    event Mint(address indexed sender, uint amount0, uint amount1);
    event Burn(address indexed sender, uint amount0, uint amount1, address indexed to);
    event Swap(
        address indexed sender,
        uint amount0In,
        uint amount1In,
        uint amount0Out,
        uint amount1Out,
        address indexed to
    );
    event Sync(uint112 reserve0, uint112 reserve1);

    constructor() public {
        factory = msg.sender;
    }

    // 因为factory合约使用create2函数创建交易对合约，无法向构造器传递参数，所以这里写了一个初始化函数用来记录合约中两种代币的地址。
    // called once by the factory at time of deployment
    function initialize(address _token0, address _token1) external {
        require(msg.sender == factory, 'UniswapV2: FORBIDDEN'); // sufficient check
        token0 = _token0;
        token1 = _token1;
    }

    // 这个函数是用来更新价格oracle的，计算累计价格
    // update reserves and, on the first call per block, price accumulators
    function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
        // 防止溢出
        require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');
        uint32 blockTimestamp = uint32(block.timestamp % 2**32);
        uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
        // 计算时间加权的累计价格，256位中，前112位用来存整数，后112位用来存小数，多的32位用来存溢出的值
        if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
            // * never overflows, and + overflow is desired
            price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
            price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
        }
        reserve0 = uint112(balance0);
        reserve1 = uint112(balance1);
        blockTimestampLast = blockTimestamp;
        emit Sync(reserve0, reserve1);
    }

    // if fee is on, mint liquidity equivalent to 1/6th of the growth in sqrt(k)
    function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
        address feeTo = IUniswapV2Factory(factory).feeTo();
        feeOn = feeTo != address(0);
        uint _kLast = kLast; // gas savings
        if (feeOn) {
            if (_kLast != 0) {
                uint rootK = Math.sqrt(uint(_reserve0).mul(_reserve1));
                uint rootKLast = Math.sqrt(_kLast);
                if (rootK > rootKLast) {
                    uint numerator = totalSupply.mul(rootK.sub(rootKLast));
                    uint denominator = rootK.mul(5).add(rootKLast);
                    uint liquidity = numerator / denominator;
                    if (liquidity > 0) _mint(feeTo, liquidity);
                }
            }
        } else if (_kLast != 0) {
            kLast = 0;
        }
    }

    // this low-level function should be called from a contract which performs important safety checks
    function mint(address to) external lock returns (uint liquidity) {
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        // 合约里两种token的当前的balance
        uint balance0 = IERC20(token0).balanceOf(address(this));
        uint balance1 = IERC20(token1).balanceOf(address(this));
        // 获得当前balance和上一次缓存的余额的差值
        uint amount0 = balance0.sub(_reserve0);
        uint amount1 = balance1.sub(_reserve1);

        // 计算手续费
        bool feeOn = _mintFee(_reserve0, _reserve1);
        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        if (_totalSupply == 0) {
            // 第一次铸币，也就是第一次注入流动性，值为根号k减去MINIMUM_LIQUIDITY
            liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
            // 把MINIMUM_LIQUIDITY赋给地址0，永久锁住
           _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
        } else {
            // 计算增量的token占总池子的比例，作为新铸币的数量
            liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
        }
        require(liquidity > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED');
        // 铸币，修改to的token数量及totalsupply
        _mint(to, liquidity);
        // 更新时间加权平均价格
        _update(balance0, balance1, _reserve0, _reserve1);
        if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
        emit Mint(msg.sender, amount0, amount1);
    }

    // this low-level function should be called from a contract which performs important safety checks
    function burn(address to) external lock returns (uint amount0, uint amount1) {
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        address _token0 = token0;                                // gas savings
        address _token1 = token1;                                // gas savings
        // 分别获取本合约地址中token0、token1和本合约代币的数量
        uint balance0 = IERC20(_token0).balanceOf(address(this));
        uint balance1 = IERC20(_token1).balanceOf(address(this));
        // 此时用户的LP token已经被转移至合约地址，因此这里取合约地址中的LP Token余额就是等下要burn掉的量
        uint liquidity = balanceOf[address(this)];

        bool feeOn = _mintFee(_reserve0, _reserve1);
        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        // 根据liquidity占比获取两个代币的实际数量
        amount0 = liquidity.mul(balance0) / _totalSupply; // using balances ensures pro-rata distribution
        amount1 = liquidity.mul(balance1) / _totalSupply; // using balances ensures pro-rata distribution
        require(amount0 > 0 && amount1 > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_BURNED');
        // 销毁LP Token
        _burn(address(this), liquidity);
        // 将token0和token1转给地址to
        _safeTransfer(_token0, to, amount0);
        _safeTransfer(_token1, to, amount1);
        balance0 = IERC20(_token0).balanceOf(address(this));
        balance1 = IERC20(_token1).balanceOf(address(this));

        // 更新时间加权平均价格
        _update(balance0, balance1, _reserve0, _reserve1);
        if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
        emit Burn(msg.sender, amount0, amount1, to);
    }

    // this low-level function should be called from a contract which performs important safety checks
    function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
        require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');

        uint balance0;
        uint balance1;
        { // scope for _token{0,1}, avoids stack too deep errors
        address _token0 = token0;
        address _token1 = token1;
        require(to != _token0 && to != _token1, 'UniswapV2: INVALID_TO');
        if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
        if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
        if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
        balance0 = IERC20(_token0).balanceOf(address(this));
        balance1 = IERC20(_token1).balanceOf(address(this));
        }
        uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');
        { // scope for reserve{0,1}Adjusted, avoids stack too deep errors
        uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
        uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
        require(balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000**2), 'UniswapV2: K');
        }

        _update(balance0, balance1, _reserve0, _reserve1);
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }

    // 强制交易对合约中两种代币的实际余额和保存的恒定乘积中的资产数量一致（多余的发送给调用者）。
    // 注意：任何人都可以调用该函数来获取额外的资产（前提是如果存在多余的资产）。
    // force balances to match reserves
    function skim(address to) external lock {
        address _token0 = token0; // gas savings
        address _token1 = token1; // gas savings
        _safeTransfer(_token0, to, IERC20(_token0).balanceOf(address(this)).sub(reserve0));
        _safeTransfer(_token1, to, IERC20(_token1).balanceOf(address(this)).sub(reserve1));
    }

    // 和skim函数刚好相反，强制保存的恒定乘积的资产数量为交易对合约中两种代币的实际余额，用于处理一些特殊情况。
    // 通常情况下，交易对中代币余额和保存的恒定乘积中的资产数量是相等的。
    // force reserves to match balances
    function sync() external lock {
        _update(IERC20(token0).balanceOf(address(this)), IERC20(token1).balanceOf(address(this)), reserve0, reserve1);
    }
}

```

# UniswapV2Factory
```solidity
pragma solidity =0.5.16;

import './interfaces/IUniswapV2Factory.sol';
import './UniswapV2Pair.sol';

contract UniswapV2Factory is IUniswapV2Factory {
    // uniswap中每次交易代币会收取0.3%的手续费，目前全部分给了LQ，若此地址不为0时，将会分出手续费中的1/6给这个地址
    address public feeTo;
    address public feeToSetter;

    mapping(address => mapping(address => address)) public getPair;
    address[] public allPairs;

    event PairCreated(address indexed token0, address indexed token1, address pair, uint);

    constructor(address _feeToSetter) public {
        feeToSetter = _feeToSetter;
    }

    function allPairsLength() external view returns (uint) {
        return allPairs.length;
    }

    function createPair(address tokenA, address tokenB) external returns (address pair) {
        // 必须是两个不一样的ERC20合约地址
        require(tokenA != tokenB, 'UniswapV2: IDENTICAL_ADDRESSES');
        // 让tokenA和tokenB的地址从小到大排列
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        // token地址不能是0
        require(token0 != address(0), 'UniswapV2: ZERO_ADDRESS');
        // 必须是uniswap中未创建过的pair
        require(getPair[token0][token1] == address(0), 'UniswapV2: PAIR_EXISTS'); // single check is sufficient
        // 获取模板合约UniswapV2Pair的creationCode
        bytes memory bytecode = type(UniswapV2Pair).creationCode;
        // 以两个token的地址作为种子生产salt
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        // 直接调用汇编创建合约
        assembly {
            pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
        }
        // 初始化刚刚创建的合约
        IUniswapV2Pair(pair).initialize(token0, token1);
        // 记录刚刚创建的合约对应的pair
        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair; // populate mapping in the reverse direction
        allPairs.push(pair);
        emit PairCreated(token0, token1, pair, allPairs.length);
    }

    function setFeeTo(address _feeTo) external {
        require(msg.sender == feeToSetter, 'UniswapV2: FORBIDDEN');
        feeTo = _feeTo;
    }

    function setFeeToSetter(address _feeToSetter) external {
        require(msg.sender == feeToSetter, 'UniswapV2: FORBIDDEN');
        feeToSetter = _feeToSetter;
    }
}

```