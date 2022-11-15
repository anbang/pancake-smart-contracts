# Pancake Farm V2 源码详解

Pancake Smart Contracts V2 版本 Farm 源码的详细解读

在 bscscan 上认证的名字是 **PancakeSwap: Main Staking Contract**。

- BscScan 地址: [0xa5f8C5Dbd5F286960b9d90548680aE5ebFf07652](https://bscscan.com/address/0xa5f8C5Dbd5F286960b9d90548680aE5ebFf07652#code)
- Github 地址:
  - 没有开放 Github 的代码，但是浏览器内可以看到

## 前置介绍

还没有详细整理，暂时梳理了框架

## V2 对比 V1 的改变

### V2 中的改变

- 计算池子中区块之间的奖励时，精度由`1e12` 改为 `1e18`，计算奖励更精确

## 状态变量

MasterChef 中包含两个主要的数据结构: `UserInfo` 和 `PoolInfo`

### UserInfo : userInfo

```
// 每个用户都有这个数据信息
struct UserInfo {
    uint256 amount;             // 用户质押的LPToken数量
    uint256 rewardDebt;         // 用户已经收到的奖励数量
    uint256 boostMultiplier;    // 加权乘数(V2新增)
}

// 每个持有 LP 代币的用户的信息
// mapping 中的 uint256 是池子的 pid
// mapping 中的 address 是用户的 address
mapping (uint256 => mapping(address => UserInfo)) public userInfo;
```

`rewardDebt` 这里做一些复杂的数学运算。保证任何时间点，等待分发给用户 CAKE 的数量是：`待发奖励 = (user.amount * pool.accCakePerShare) - user.rewardDebt`

每当用户对池子进行 LP 代币的存取时，代码内做如下操作：

1. 池的 `accCakePerShare` 和 `lastRewardBlock` 进行更新。
   1. 该功能在 `updatePool()` 内实现 **更新池子系数**
2. 计算待发奖励，并直接转给用户地址。
   1. 该功能在 `settlePendingCake()` 内实现 **用户待发奖励结算**
3. 更新用户的 `amount`
4. 更新用户的 `rewardDebt`

> 变化: 此处比起 V1 版本，增加了 `boostMultiplier` 的加权乘数。

> 注: accCakePerShare 在 PoolInfo 内详细介绍。

### PoolInfo : poolInfo

```
// 每个LP池的信息
struct PoolInfo {
    uint256 accCakePerShare;    // 质押1 wei LP的全局CAKE收益；计算时会乘以 1e18
    uint256 lastRewardBlock;    // 最后一次分配 CAKEs 的区块号
    uint256 allocPoint;         // 该LP池的分配点数。CAKE按块分配

    uint256 totalBoostedShare;  // 总提升份额
    bool isRegular;             // 是否为普通池
}

PoolInfo[] public poolInfo;   // poolInfo 储存所有LP池的信息
```

- allocPoint :分配点数，质押池的分配比例，可以通过调整它来设置 LP 挖矿的权重
- lastRewardBlock :最后一次分配 CAKE 的块，代码计算时代表"上一次分配奖励的区块数"
- accCakePerShare :从池子创建时质押 1 个 wei LPToken ,到目前为止；在当前池子的 CAKE 收益

**池子奖励计算方法如下**（2 个区块之间的奖励）：

- accCakePerShare 默认值是 0，更新方法如下
- 2 个区块之间的池子奖励是:
  - `cakeReward = 单个块的 CAKE 奖励 * (pool.allocPoint / totalAllocPoint) * 两个块之间的块数`
  - 和 V1 相比，这里 totalAllocPoint 根据池子不同分为 `totalRegularAllocPoint` 和 `totalSpecialAllocPoint`,选择合适的计算
- 池子的总量是 lpSupply 个，则 1 wei LP 的在两个区块间的质押奖励是
  - `(cakeReward / lpSupply)`
- 用户的资产是一直放在池子内的，除了当前两个区块之间的奖励，之前的块也是有奖励的，所以这里的 pool.accCakePerShare 需要累加。
  - `pool.accCakePerShare = pool.accCakePerShare + (cakeReward * 1e18 / lpSupply)`
  - 这里引入 `1e18` 是为了精度，所以计算用户奖励以及给用户发奖励的时候，可以通过除以 `1e18` 得到真实值；源码注释里说使用的是`1e12`计算，其实是使用常量`ACC_CAKE_PRECISION` 的 `1e18` 精度来计算。

**用户奖励计算方法如下**（2 个区块之间的奖励）：

用户依赖`user.amount`，`user.rewardDebt` 和 `pool.accCakePerShare`计算实际收益。原理如下:

- 用户在质押 LPToken 时，`user.amount` 结合 `pool.accCakePerShare` 记下来作为起始点位，表示已经发给用户的奖励`user.rewardDebt`.
- 用户在解押 LPToken 时，用`user.amount` 结合最新 `pool.accCakePerShare` 计算累积奖励。
- 这一段时间内的实际收益可以通过累积奖励减去之前的`user.rewardDebt`得到

上面这个计算收益的原理，就是 `settlePendingCake()` 这个用户待发奖励结算的方法核心。

变化: 相比 V1 版本，移除了`lpToken`,新增了`totalBoostedShare` 和 `isRegular`。

- `totalBoostedShare` 每个池中的用户份额总量。
- `isRegular`：设置当前 LP 池是常规池，还是特殊池
  - **regular pools**: V2 farms
  - **special pools**:"特殊池"使用不同的"allocPoint"集合和它们自己的"totalSpecialAllocPoint"，旨在处理 CAKE 奖励向所有 PancakeSwap 产品的分配。

### 其他状态变量

```
// Farm V1 合约的地址: 供 V2 连接 V1
IMasterChef public immutable MASTER_CHEF;

// CAKE 合约地址
IBEP20 public immutable CAKE;

// 用于接收所有 burn CAKE 的唯一地址
address public burnAdmin;

// 处理 share boosts 的合约
address public boostContract;

// V2池所有 LP token 的地址集合
IBEP20[] public lpToken;

// 允许存放在特殊池(special pools)中的地址白名单。
mapping(address => bool) public whiteList;

// 模拟V1的 pool id
uint256 public immutable MASTER_PID;

// 所有 regular pool 的总分配点数。必须与所有常规池的AllocPoint匹配
uint256 public totalRegularAllocPoint;

// 所有 special pool 的总分配点数。必须与所有特殊池的AllocPoint匹配
uint256 public totalSpecialAllocPoint;

// V1 中每块 40 个CAKE币
uint256 public constant MASTERCHEF_CAKE_PER_BLOCK = 40 * 1e18;
uint256 public constant ACC_CAKE_PRECISION = 1e18;

// 基本的加权系数（不存在针对用户的加权系数）
uint256 public constant BOOST_PRECISION = 100 * 1e10;

// 最大加权系数的强制限制，该值必须大于 BOOST_PRECISION
uint256 public constant MAX_BOOST_PRECISION = 200 * 1e10;

// 总 CAKE 产出率的精度; 必须等于下面三个 cakeRate 的总和
// total cake rate = toBurn + toRegular + toSpecial
uint256 public constant CAKE_RATE_TOTAL_PRECISION = 1e12;

// 正在执行 CAKE 销毁动作的最后区块号。
// CAKE distribution % for burn
uint256 public cakeRateToBurn = 643750000000;

// CAKE distribute % for regular farm pool
uint256 public cakeRateToRegularFarm = 62847222222;

// CAKE distribute % for special pools
uint256 public cakeRateToSpecialFarm = 293402777778;

// 最后 Burned 的块号
uint256 public lastBurnedBlock;
```

## 构造函数

代码如下: MasterChefV2 部署时需要初始化的参数

```
constructor(
    IMasterChef _MASTER_CHEF,
    IBEP20 _CAKE,
    uint256 _MASTER_PID,
    address _burnAdmin
) public {
    MASTER_CHEF = _MASTER_CHEF;
    CAKE = _CAKE;
    MASTER_PID = _MASTER_PID;
    burnAdmin = _burnAdmin;
}
```

参数解析如下

- MASTER_CHEF: Pancake Master Chef V1 版本的合约地址
- CAKE: CAKE 币的合约地址
- MASTER_PID: V1 版本上模拟池的 pool id
- burnAdmin: 接收销毁 CAKE 的地址

在 V1 版本内，有 Staking 池 和 普通 pool 的区分，并且在构造函数内初始化 Staking 池。而在 V2 版本，分为普通池和特殊池。

## updatePool 更新池子系数

```
function updatePool(uint256 _pid) public returns (PoolInfo memory pool) {
    pool = poolInfo[_pid];
    if (block.number > pool.lastRewardBlock) {
        uint256 lpSupply = pool.totalBoostedShare;
        uint256 totalAllocPoint = (pool.isRegular ? totalRegularAllocPoint : totalSpecialAllocPoint);

        if (lpSupply > 0 && totalAllocPoint > 0) {
            uint256 multiplier = block.number.sub(pool.lastRewardBlock);
            uint256 cakeReward = multiplier.mul(cakePerBlock(pool.isRegular)).mul(pool.allocPoint).div(
                totalAllocPoint
            );
            pool.accCakePerShare = pool.accCakePerShare.add((cakeReward.mul(ACC_CAKE_PRECISION).div(lpSupply)));
        }
        pool.lastRewardBlock = block.number;
        poolInfo[_pid] = pool;
        emit UpdatePool(_pid, pool.lastRewardBlock, lpSupply, pool.accCakePerShare);
    }
}
```

## settlePendingCake 用户待发奖励结算

将待发给用户的 CAKE 奖励给结算掉

```
function settlePendingCake(
    address _user,
    uint256 _pid,
    uint256 _boostMultiplier
) internal {
    UserInfo memory user = userInfo[_pid][_user];

    uint256 boostedAmount = user.amount.mul(_boostMultiplier).div(BOOST_PRECISION);
    uint256 accCake = boostedAmount.mul(poolInfo[_pid].accCakePerShare).div(ACC_CAKE_PRECISION);
    uint256 pending = accCake.sub(user.rewardDebt);
    // SafeTransfer CAKE
    _safeTransfer(_user, pending);
}
```

## 核心

- init:`init(IBEP20 dummyToken)`
  - 🔒: onlyOwner
- add:`add(_allocPoint, _lpToken, _isRegular, _withUpdate)`
  - 🔒: onlyOwner
- set:`set(_pid, _allocPoint, _withUpdate)`
  - 🔒: onlyOwner
- deposit:`deposit(_pid, _amount)`
- withdraw:`withdraw(_pid, _amount)`

### init

```
// Deposits a dummy token to `MASTER_CHEF` MCV1. This is required because MCV1 holds the minting permission of CAKE.
/// It will transfer all the `dummyToken` in the tx sender address.
/// The allocation point for the dummy pool on MCV1 should be equal to the total amount of allocPoint.
/// @param dummyToken The address of the BEP-20 token to be deposited into MCV1.
function init(IBEP20 dummyToken) external onlyOwner {
    uint256 balance = dummyToken.balanceOf(msg.sender);
    require(balance != 0, "MasterChefV2: Balance must exceed 0");
    dummyToken.safeTransferFrom(msg.sender, address(this), balance);
    dummyToken.approve(address(MASTER_CHEF), balance);
    MASTER_CHEF.deposit(MASTER_PID, balance);
    // MCV2 start to earn CAKE reward from current block in MCV1 pool
    lastBurnedBlock = block.number;
    emit Init();
}
```

### add

```
// Add a new pool. Can only be called by the owner.
/// DO NOT add the same LP token more than once. Rewards will be messed up if you do.
/// @param _allocPoint Number of allocation points for the new pool.
/// @param _lpToken Address of the LP BEP-20 token.
/// @param _isRegular Whether the pool is regular or special. LP farms are always "regular". "Special" pools are
/// @param _withUpdate Whether call "massUpdatePools" operation.
/// only for CAKE distributions within PancakeSwap products.
function add(
    uint256 _allocPoint,
    IBEP20 _lpToken,
    bool _isRegular,
    bool _withUpdate
) external onlyOwner {
    require(_lpToken.balanceOf(address(this)) >= 0, "None BEP20 tokens");
    // stake CAKE token will cause staked token and reward token mixed up,
    // may cause staked tokens withdraw as reward token,never do it.
    require(_lpToken != CAKE, "CAKE token can't be added to farm pools");

    if (_withUpdate) {
        massUpdatePools();
    }

    if (_isRegular) {
        totalRegularAllocPoint = totalRegularAllocPoint.add(_allocPoint);
    } else {
        totalSpecialAllocPoint = totalSpecialAllocPoint.add(_allocPoint);
    }
    lpToken.push(_lpToken);

    poolInfo.push(
        PoolInfo({
    allocPoint: _allocPoint,
    lastRewardBlock: block.number,
    accCakePerShare: 0,
    isRegular: _isRegular,
    totalBoostedShare: 0
    })
    );
    emit AddPool(lpToken.length.sub(1), _allocPoint, _lpToken, _isRegular);
}
```

### set

```
// Update the given pool's CAKE allocation point. Can only be called by the owner.
/// @param _pid The id of the pool. See `poolInfo`.
/// @param _allocPoint New number of allocation points for the pool.
/// @param _withUpdate Whether call "massUpdatePools" operation.
function set(
    uint256 _pid,
    uint256 _allocPoint,
    bool _withUpdate
) external onlyOwner {
    // No matter _withUpdate is true or false, we need to execute updatePool once before set the pool parameters.
    updatePool(_pid);

    if (_withUpdate) {
        massUpdatePools();
    }

    if (poolInfo[_pid].isRegular) {
        totalRegularAllocPoint = totalRegularAllocPoint.sub(poolInfo[_pid].allocPoint).add(_allocPoint);
    } else {
        totalSpecialAllocPoint = totalSpecialAllocPoint.sub(poolInfo[_pid].allocPoint).add(_allocPoint);
    }
    poolInfo[_pid].allocPoint = _allocPoint;
    emit SetPool(_pid, _allocPoint);
}
```

### deposit

```
// Deposit LP tokens to pool.
/// @param _pid The id of the pool. See `poolInfo`.
/// @param _amount Amount of LP tokens to deposit.
function deposit(uint256 _pid, uint256 _amount) external nonReentrant {
    PoolInfo memory pool = updatePool(_pid);
    UserInfo storage user = userInfo[_pid][msg.sender];

    require(
        pool.isRegular || whiteList[msg.sender],
        "MasterChefV2: The address is not available to deposit in this pool"
    );

    uint256 multiplier = getBoostMultiplier(msg.sender, _pid);

    if (user.amount > 0) {
        settlePendingCake(msg.sender, _pid, multiplier);
    }

    if (_amount > 0) {
        uint256 before = lpToken[_pid].balanceOf(address(this));
        lpToken[_pid].safeTransferFrom(msg.sender, address(this), _amount);
        _amount = lpToken[_pid].balanceOf(address(this)).sub(before);
        user.amount = user.amount.add(_amount);

        // Update total boosted share.
        pool.totalBoostedShare = pool.totalBoostedShare.add(_amount.mul(multiplier).div(BOOST_PRECISION));
    }

    user.rewardDebt = user.amount.mul(multiplier).div(BOOST_PRECISION).mul(pool.accCakePerShare).div(
        ACC_CAKE_PRECISION
    );
    poolInfo[_pid] = pool;

    emit Deposit(msg.sender, _pid, _amount);
}
```

### withdraw

```
// Withdraw LP tokens from pool.
/// @param _pid The id of the pool. See `poolInfo`.
/// @param _amount Amount of LP tokens to withdraw.
function withdraw(uint256 _pid, uint256 _amount) external nonReentrant {
    PoolInfo memory pool = updatePool(_pid);
    UserInfo storage user = userInfo[_pid][msg.sender];

    require(user.amount >= _amount, "withdraw: Insufficient");

    uint256 multiplier = getBoostMultiplier(msg.sender, _pid);

    settlePendingCake(msg.sender, _pid, multiplier);

    if (_amount > 0) {
        user.amount = user.amount.sub(_amount);
        lpToken[_pid].safeTransfer(msg.sender, _amount);
    }

    user.rewardDebt = user.amount.mul(multiplier).div(BOOST_PRECISION).mul(pool.accCakePerShare).div(
        ACC_CAKE_PRECISION
    );
    poolInfo[_pid].totalBoostedShare = poolInfo[_pid].totalBoostedShare.sub(
        _amount.mul(multiplier).div(BOOST_PRECISION)
    );

    emit Withdraw(msg.sender, _pid, _amount);
}
```

## 辅助函数

- `onlyBoostContract`
- `poolLength()`
- `pendingCake(_pid, _user)`
- `massUpdatePools()`
- `cakePerBlock(_isRegular)`
- `cakePerBlockToBurn()`
- `harvestFromMasterChef()`
- `getBoostMultiplier(_user, _pid)`

### onlyBoostContract 地址限制

```
modifier onlyBoostContract() {
    require(boostContract == msg.sender, "Ownable: caller is not the boost contract");
    _;
}
```

使用函数修改器 `onlyBoostContract` 限制一些只能 `boostContract` 身份的调用，类似很多合约中的 onlyOwner。

### poolLength 池子总数量

```
function poolLength() public view returns (uint256 pools) {
    pools = poolInfo.length;
}
```

### pendingCake 未领取奖励

```
function pendingCake(uint256 _pid, address _user) external view returns (uint256) {
    PoolInfo memory pool = poolInfo[_pid];
    UserInfo memory user = userInfo[_pid][_user];
    uint256 accCakePerShare = pool.accCakePerShare;
    uint256 lpSupply = pool.totalBoostedShare;

    if (block.number > pool.lastRewardBlock && lpSupply != 0) {
        uint256 multiplier = block.number.sub(pool.lastRewardBlock);

        uint256 cakeReward = multiplier.mul(cakePerBlock(pool.isRegular)).mul(pool.allocPoint).div(
            (pool.isRegular ? totalRegularAllocPoint : totalSpecialAllocPoint)
        );
        accCakePerShare = accCakePerShare.add(cakeReward.mul(ACC_CAKE_PRECISION).div(lpSupply));
    }

    uint256 boostedAmount = user.amount.mul(getBoostMultiplier(_user, _pid)).div(BOOST_PRECISION);
    return boostedAmount.mul(accCakePerShare).div(ACC_CAKE_PRECISION).sub(user.rewardDebt);
}
```

### massUpdatePools 更新所有池子系数

```
function massUpdatePools() public {
    uint256 length = poolInfo.length;
    for (uint256 pid = 0; pid < length; ++pid) {
        PoolInfo memory pool = poolInfo[pid];
        if (pool.allocPoint != 0) {
            updatePool(pid);
        }
    }
}
```

### cakePerBlock 计算每个块的产出数量

```
function cakePerBlock(bool _isRegular) public view returns (uint256 amount) {
    if (_isRegular) {
        amount = MASTERCHEF_CAKE_PER_BLOCK.mul(cakeRateToRegularFarm).div(CAKE_RATE_TOTAL_PRECISION);
    } else {
        amount = MASTERCHEF_CAKE_PER_BLOCK.mul(cakeRateToSpecialFarm).div(CAKE_RATE_TOTAL_PRECISION);
    }
}
```

### cakePerBlockToBurn 计算每个块的销毁数量

```
function cakePerBlockToBurn() public view returns (uint256 amount) {
    amount = MASTERCHEF_CAKE_PER_BLOCK.mul(cakeRateToBurn).div(CAKE_RATE_TOTAL_PRECISION);
}
```

### harvestFromMasterChef 供 V1 升级 V2

```
function harvestFromMasterChef() public {
    MASTER_CHEF.deposit(MASTER_PID, 0);
}
```

从 V1 版本获取 CAKE，并且将 `PID` 写入到 V2 版本

### getBoostMultiplier 获取用户的加权乘数

```
function getBoostMultiplier(address _user, uint256 _pid) public view returns (uint256) {
    uint256 multiplier = userInfo[_pid][_user].boostMultiplier;
    return multiplier > BOOST_PRECISION ? multiplier : BOOST_PRECISION;
}
```

## 更新数据

- `updateBurnAdmin(_newAdmin)`
- `updateWhiteList(_user, _isValid)`
- `updateBoostContract(address _newBoostContract)`
- `updateBoostMultiplier(_user,_pid,_newMultiplier)`

### updateBurnAdmin 更新 burnAdmin

🔒 onlyOwner 调用

```
function updateBurnAdmin(address _newAdmin) external onlyOwner {
    require(_newAdmin != address(0), "MasterChefV2: Burn admin address must be valid");
    require(_newAdmin != burnAdmin, "MasterChefV2: Burn admin address is the same with current address");
    address _oldAdmin = burnAdmin;
    burnAdmin = _newAdmin;
    emit UpdateBurnAdmin(_oldAdmin, _newAdmin);
}

```

### updateWhiteList 更新白名单

```
function updateWhiteList(address _user, bool _isValid) external onlyOwner {
    require(_user != address(0), "MasterChefV2: The white list address must be valid");

    whiteList[_user] = _isValid;
    emit UpdateWhiteList(_user, _isValid);
}
```

### updateBoostContract 更新 boostContract

```
function updateBoostContract(address _newBoostContract) external onlyOwner {
    require(
        _newBoostContract != address(0) && _newBoostContract != boostContract,
        "MasterChefV2: New boost contract address must be valid"
    );

    boostContract = _newBoostContract;
    emit UpdateBoostContract(_newBoostContract);
}
```

### updateBoostMultiplier 更新用户的加权系数

```
function updateBoostMultiplier(
    address _user,
    uint256 _pid,
    uint256 _newMultiplier
) external onlyBoostContract nonReentrant {
    require(_user != address(0), "MasterChefV2: The user address must be valid");
    require(poolInfo[_pid].isRegular, "MasterChefV2: Only regular farm could be boosted");
    require(
        _newMultiplier >= BOOST_PRECISION && _newMultiplier <= MAX_BOOST_PRECISION,
        "MasterChefV2: Invalid new boost multiplier"
    );

    PoolInfo memory pool = updatePool(_pid);
    UserInfo storage user = userInfo[_pid][_user];

    uint256 prevMultiplier = getBoostMultiplier(_user, _pid);
    settlePendingCake(_user, _pid, prevMultiplier);

    user.rewardDebt = user.amount.mul(_newMultiplier).div(BOOST_PRECISION).mul(pool.accCakePerShare).div(
        ACC_CAKE_PRECISION
    );
    pool.totalBoostedShare = pool.totalBoostedShare.sub(user.amount.mul(prevMultiplier).div(BOOST_PRECISION)).add(
        user.amount.mul(_newMultiplier).div(BOOST_PRECISION)
    );
    poolInfo[_pid] = pool;
    userInfo[_pid][_user].boostMultiplier = _newMultiplier;

    emit UpdateBoostMultiplier(_user, _pid, prevMultiplier, _newMultiplier);
}
```

## 销毁 CAKE

计算所有需要销毁的 CAKE，并转给 `burnAdmin` 地址。

```
function burnCake(bool _withUpdate) public onlyOwner {
    if (_withUpdate) {
        massUpdatePools();
    }

    uint256 multiplier = block.number.sub(lastBurnedBlock);
    uint256 pendingCakeToBurn = multiplier.mul(cakePerBlockToBurn());

    // SafeTransfer CAKE
    _safeTransfer(burnAdmin, pendingCakeToBurn);
    lastBurnedBlock = block.number;
}
```

## 更新 CAKE 产出系数

更新销毁部分，常规池，特殊池的 CAKE 分配百分比。

```
// _burnRate        每个块的销毁比例
// _regularFarmRate 每个块的普通池分配比例
// _specialFarmRate 每个块的特殊池分配比例
// _withUpdate      是否调用"massUpdatePools"更新所有池操作。
function updateCakeRate(
    uint256 _burnRate,
    uint256 _regularFarmRate,
    uint256 _specialFarmRate,
    bool _withUpdate
) external onlyOwner {
    require(
        _burnRate > 0 && _regularFarmRate > 0 && _specialFarmRate > 0,
        "MasterChefV2: Cake rate must be greater than 0"
    );
    require(
        _burnRate.add(_regularFarmRate).add(_specialFarmRate) == CAKE_RATE_TOTAL_PRECISION,
        "MasterChefV2: Total rate must be 1e12"
    );
    if (_withUpdate) {
        massUpdatePools();
    }
    // burn cake base on old burn cake rate
    burnCake(false);

    cakeRateToBurn = _burnRate;
    cakeRateToRegularFarm = _regularFarmRate;
    cakeRateToSpecialFarm = _specialFarmRate;

    emit UpdateCakeRate(_burnRate, _regularFarmRate, _specialFarmRate);
}
```

## 逃生通道

使用 `emergencyWithdraw(_pid)` 可以进行资金逃生，该方法提取所有本金，忽略奖励，并把用户的所有数据初始化。

```
function emergencyWithdraw(uint256 _pid) external nonReentrant {
    PoolInfo storage pool = poolInfo[_pid];
    UserInfo storage user = userInfo[_pid][msg.sender];

    uint256 amount = user.amount;
    user.amount = 0;
    user.rewardDebt = 0;
    uint256 boostedAmount = amount.mul(getBoostMultiplier(msg.sender, _pid)).div(BOOST_PRECISION);
    pool.totalBoostedShare = pool.totalBoostedShare > boostedAmount ? pool.totalBoostedShare.sub(boostedAmount) : 0;

    // Note: transfer can fail or succeed if `amount` is zero.
    lpToken[_pid].safeTransfer(msg.sender, amount);
    emit EmergencyWithdraw(msg.sender, _pid, amount);
}
```

## 安全转账

```
function _safeTransfer(address _to, uint256 _amount) internal {
    if (_amount > 0) {
        // Check whether MCV2 has enough CAKE. If not, harvest from MCV1.
        if (CAKE.balanceOf(address(this)) < _amount) {
            harvestFromMasterChef();
        }
        uint256 balance = CAKE.balanceOf(address(this));
        if (balance < _amount) {
            _amount = balance;
        }
        CAKE.safeTransfer(_to, _amount);
    }
}
```
