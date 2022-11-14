# Pancake Farm V1 源码详解

Pancake Smart Contracts V1 版本 Farm 源码的详细解读

在 bscscan 上认证的名字是 **PancakeSwap: Main Staking Contract**。

- Github 地址: <https://github.com/pancakeswap/pancake-farm>
- BscScan 地址: [0x73feaa1ee314f8c655e354234017be2193c9e24e](https://bscscan.com/address/0x73feaa1ee314f8c655e354234017be2193c9e24e#code)

## 前置介绍

Pancake Farm 代码看起来是基于 SushiSwap 的 MasterChef 代码修改而来。

Pancake Farm 是一个奖励分发合约。可以通过存 CAKE 和 LP Token 两种途径来得到 CAKE 和 SYRUP 奖励。其中第 1 个池子不是 LP Token，而是 Cake Token，可以存 CAKE 赚 CAKE，属于 Staking 概念

所有用户可以进行四种基础的资产操作

- LP 池:
  - 存款
  - 取款
- CAKE 池:
  - 进入 Staking
  - 离开 Staking

PancakeSwap Farm V1 版本的核心模块是: `MasterChef`，用户可以通过它进行流动性挖矿。这个模块控制了两个 ERC20 代币的产出，分别是 **CAKE TOKEN** 和 **SYRUP TOKEN**。所以需要将这两个代币的 Owner 权限转移给 `MasterChef` 。

除了用户赚取的 CAKE 外，池子每次更新时候，合约都会向设置的 `devaddr` 以及 `syrup token` 这 2 个地址转奖励相关数量的 Token。

假设某个池子从上次更新到这次之间的所有奖励是 `cakeReward`

- 其中向`devaddr`转 `cakeReward*0.1` 个 CAKE
- 其中向`syrup token`转 `cakeReward` 个 CAKE

也就是说每次 CAKE 给某个池子 100 个 CAKE 的时候，会向配置的开发者地址转 10 个 CAKE，以及向 `syrup token`也转 `cakeReward`个 CAKE，该部分的逻辑在后面介绍的 `updatePool`这个更新池子的奖励变量的函数中。

## 代码详解

### 状态变量

MasterChef 中包含两个主要的数据结构: `UserInfo` 和 `PoolInfo`

#### UserInfo : userInfo

```
// 每个用户都有这个数据信息.
struct UserInfo {
    uint256 amount;     // 用户质押的LPToken数量
    uint256 rewardDebt; // 用户已经收到的奖励数量
}
// 每个持有 LP 代币的用户的信息。
mapping (uint256 => mapping (address => UserInfo)) public userInfo;
```

`rewardDebt` 这里做一些复杂的数学运算。保证任何时间点，等待分发给用户 CAKE 的数量是：`待发奖励 = (user.amount * pool.accCakePerShare) - user.rewardDebt`

每当用户将 LP 代币存入或提取到池中时，代码内做如下操作：

1. 池的 `accCakePerShare` 和 `lastRewardBlock` 进行更新。
2. 计算待发奖励，并直接转给用户地址。
3. 更新用户的 `amount`
4. 更新用户的 `rewardDebt`

> 注：accCakePerShare 在 PoolInfo 内详细介绍。

#### PoolInfo : poolInfo

```
// 每个LP池的信息
struct PoolInfo {
    IBEP20 lpToken;           // LP代币的合约地址，是标准ERC20代币
    uint256 allocPoint;       // 该LP池的分配点数。CAKE按块分配
    uint256 lastRewardBlock;  // 最后一次分配 CAKEs 的区块号
    uint256 accCakePerShare;  // 质押1个LP的全局CAKE收益；计算时会乘以 1e12
}

PoolInfo[] public poolInfo;   // poolInfo 储存所有LP池的信息
```

- allocPoint :质押池的分配比例，可以通过调整它来设置 LP 挖矿的权重
- lastRewardBlock :最后一次分配 CAKE 的块，代码计算时代表"上一次分配奖励的区块数"
- accCakePerShare :从池子创建时质押 1 个 wei LPToken ,到目前为止；在当前池子的 CAKE 收益
  - accCakePerShare 默认值是 0，更新方法如下
  - 2 个区块之间的池子奖励是:
    - `cakeReward = 单个块的 CAKE 奖励 * (pool.allocPoint / totalAllocPoint) * 两个块之间的块数`
  - 池子的总量是 lpSupply 个，则 1 wei LP 的在两个区块间的质押奖励是
    - `(cakeReward / lpSupply)`
  - 用户的资产是一直放在池子内的，除了当前两个区块之间的奖励，之前的块也是有奖励的，所以这里的 pool.accCakePerShare 需要累加。
    - `pool.accCakePerShare = pool.accCakePerShare + (cakeReward * 1e12 / lpSupply)`
    - 这里引入 `1e12` 是为了精度，所以计算用户奖励以及给用户发奖励的时候，可以通过除以 `1e12` 得到真实值

用户依赖`amount`和`accCakePerShare`计算实际收益。原理如下:

- 用户在质押 LPToken 时，把当前 accCakePerShare 记下来作为起始点位.
- 用户在解押 LPToken 时，用最新 accCakePerShare 减去起始点位，就可以得到用户在进入和退出之间 1 个 wei LP 的收益。再结合用户的 amount 就可以得到最终收益。

#### 其他状态变量

```
CakeToken public cake;  // CAKE  TOKEN
SyrupBar public syrup;  // SYRUP TOKEN

address public devaddr; // 开发者地址，每当池子更新时，会将待发给用户的n个CAKE转给用户。
                        // 此时会额外增发 10% 给开发者地址，该地址是收益地址，可以再次设置

uint256 public cakePerBlock;          // 每个区块产出的 CAKE 数量
uint256 public BONUS_MULTIPLIER = 1;  // CAKE 的奖金乘数

IMigratorChef public migrator;        // 迁移合约，权限很大。只有MasterChef部署地址才可以调用迁移

uint256 public totalAllocPoint = 0;   // 总分配点数。必须是所有LP池的所有分配点数的总和。

uint256 public startBlock;            // 开始挖 CAKE 的区块号
```

这里的 `BONUS_MULTIPLIER` 是一个块的奖金乘数，前面介绍 2 个区块之间的池子奖励是:

- `cakeReward = 单个块的 CAKE 奖励 * (pool.allocPoint / totalAllocPoint) * 两个块之间的块数`
- 而在计算两个块之间的块数时，代码用的是 `(toBlock - fromBlock) * BONUS_MULTIPLIER`，可以通过 `BONUS_MULTIPLIER` 值对块数量进行加权。

### 构造函数

代码如下: MasterChef 部署时需要初始化的参数

```
constructor(
    CakeToken _cake,
    SyrupBar _syrup,
    address _devaddr,
    uint256 _cakePerBlock,
    uint256 _startBlock
) public {
    cake = _cake;
    syrup = _syrup;
    devaddr = _devaddr;
    cakePerBlock = _cakePerBlock;
    startBlock = _startBlock;

    // 质押池
    poolInfo.push(PoolInfo({
        lpToken: _cake,
        allocPoint: 1000,
        lastRewardBlock: startBlock,
        accCakePerShare: 0
    }));

    totalAllocPoint = 1000;
}
```

#### 参数解析如下

- CakeToken: 用于挖矿的 CAKE 代币地址
- SyrupBar: 用于挖矿的 SYRUP 代币地址
- devaddr: 开发者地址，用于接收额外的 10% CAKE 奖励
- cakePerBlock: 每个区块产出的 CAKE 数量
- startBlock: 开始挖矿的区块号

注意这里的: `cakePerBlock` 是初始化之后无法修改的。如果做合约改写，这里需要注意下。除了上面 5 个值的操作，构造函数内还创建了 CAKE Staking 池。

#### 构造函数内做的事情

把 CAKE Token 的地址作为 LP，并把该信息作为 poolInfo 的第 1 个池子进行 push。将 startBlock 设置为默认的 lastRewardBlock。CAKE 的 allocPoint 设置为 1000，totalAllocPoint 对应的设置为 1000. accCakePerShare 默认为 0。

### updatePool:更新池子的奖励变量

**LP 的存-取**和**Staking 的进入-退出** 都会触发更新池子的奖励变量。所以先介绍这个方法：

```
function updatePool(uint256 _pid) public {
    PoolInfo storage pool = poolInfo[_pid];
    if (block.number <= pool.lastRewardBlock) {
        return;
    }
    uint256 lpSupply = pool.lpToken.balanceOf(address(this));
    if (lpSupply == 0) {
        pool.lastRewardBlock = block.number;
        return;
    }
    uint256 multiplier = getMultiplier(pool.lastRewardBlock, block.number);
    uint256 cakeReward = multiplier.mul(cakePerBlock).mul(pool.allocPoint).div(totalAllocPoint);
    cake.mint(devaddr, cakeReward.div(10)); // 增发奖励的 10% CAKE 给 devaddr
    cake.mint(address(syrup), cakeReward);  // 增发奖励的等量 CAKE 给 syrup
    pool.accCakePerShare = pool.accCakePerShare.add(cakeReward.mul(1e12).div(lpSupply));
    pool.lastRewardBlock = block.number;
}
```

该方法接受 pid 作为参数，用于更新指定 pid 的池子。

如果当前区块还没有到达开始挖矿的区块号，则退出；因为这个方法是用发放 CAKE 代币，没到开始挖矿的区块，就没必要计算了。（staking 池在构造函数部署时，lastRewardBlock 设置为了 startBlock；LP 池添加的时候，如果当前区块小于 startBlock 则使用 startBlock，否则使用添加时的区块号 ）

获取当前池子内的 LP 余额(lpSupply)，如果 lpSupply 为 0，则更新 lastRewardBlock 后退出函数。因为池子是空的，也没有必要分发奖励。

如果挖矿的已经开始了，并且 lpSupply 也大于零，则执行下面逻辑：

---

`multiplier = getMultiplier(pool.lastRewardBlock, block.number)`:获取 from 到 to 的 reward multiplier

```
function getMultiplier(uint256 _from, uint256 _to) public view returns (uint256) {
    return _to.sub(_from).mul(BONUS_MULTIPLIER);
}
```

默认的 `BONUS_MULTIPLIER` 为 1，所以默认返回是区块数量的差值，表示 from 到 to 一共多少个块。

当前 LP 池每一个块可以获取的奖励等于 `cakePerBlock * (pool.allocPoint/totalAllocPoint)`，那么当前 LP 池从上一次发放奖励的区块到当前区块，合计挖出来的奖励 **cakeReward** 就等于 `multiplier * (cakePerBlock * (pool.allocPoint/totalAllocPoint))`

下面是业务代码，基于 LP 池的奖励，mint 出 CAKE 转给 devaddr 和 syrup。

```
cake.mint(devaddr, cakeReward.div(10)); // 增发奖励的 10% CAKE 给 devaddr
cake.mint(address(syrup), cakeReward);  // 增发奖励的等量 CAKE 给 syrup
```

然后更新 `pool.accCakePerShare` 和 `pool.lastRewardBlock`

方法总结：如果执行此方法，则对应 pid 池子的 `accCakePerShare` 和 `lastRewardBlock` 会更新；也会根据奖励，mint CAKE 转给 `devaddr` 和 `syrup` 地址。

### CAKE Staking

- `enterStaking(_amount)` : 存入 CAKE 代币:质押指定金额进行挖矿
- `leaveStaking(_amount)` : 取出 CAKE 代币:解押指定金额退出挖矿
- `updateStakingPool()` : 更新 Staking 池
  - ⚠️: `internal` 方法

#### enterStaking: 参加 staking

代码如下:

```
function enterStaking(uint256 _amount) public {
    PoolInfo storage pool = poolInfo[0];
    UserInfo storage user = userInfo[0][msg.sender];
    updatePool(0);
    if (user.amount > 0) {
        uint256 pending = user.amount.mul(pool.accCakePerShare).div(1e12).sub(user.rewardDebt);
        if(pending > 0) {
            safeCakeTransfer(msg.sender, pending);
        }
    }
    if(_amount > 0) {
        pool.lpToken.safeTransferFrom(address(msg.sender), address(this), _amount);
        user.amount = user.amount.add(_amount);
    }
    user.rewardDebt = user.amount.mul(pool.accCakePerShare).div(1e12);

    syrup.mint(msg.sender, _amount);
    emit Deposit(msg.sender, 0, _amount);
}
```

首先进行 `updatePool(0)`,此时 staking 池子的`accCakePerShare` 和 `lastRewardBlock` 会更新。

---

如果在本次存入 CAKE 前，用户在 staking 里还有金额；则需要发放本次到上一次之间的 CAKE 奖励给用户地址。

- `pool.accCakePerShare` 是池子里 1wei 的累计收益
- 通过 `user.amount * pool.accCakePerShare` 就可以得到之前金额的累计收益；因为这个值是累积的收益值，所以还需要减掉之前已经累积发掉的奖励金额
- `user.rewardDebt` 是用户已经获取到的收益。也是累积的值
- 所以用户本次和上次之间的收益如下(记得 accCakePerShare 需要除以 1e12)：

  ```
  uint256 pending = user.amount.mul(pool.accCakePerShare).div(1e12).sub(user.rewardDebt);
  ```

如果 pending 的值大于 0，把该金额发给用户地址。

---

如果本次参加 staking 的金额大于 0，则从用户地址划转对应的金额到当前 lp 池子内，并且更新用户的 amount 值。此时需要前端开发者注意在 staking 前需要提示用户需要将 CAKE 的 amount 金额授权给 lp 合约（为了以后省事，也可以授权一个最大者）。

---

完成以上操作后，需要使用当前的 `user.amount` 和 `pool.accCakePerShare` 记录起点，方便下一次进行奖励转账

```
user.rewardDebt = user.amount.mul(pool.accCakePerShare).div(1e12);
```

然后 MINT 一些 syrup 币，空头给用户地址；注意，这个在退出 staking 的时候需要从用户的地址上销毁等量金额的 syrup 币，如果退出的时候，账号地址上没有 syrup 币，没办法退出的。

总结一下逻辑，代码思路以后还会频繁用到:

1. 首先更新池子 `accCakePerShare` 和 `lastRewardBlock`
2. 之前金额:如果在本次存入 CAKE 前，用户在 staking 里还有金额；则需要发放本次到上一次之间的 CAKE 奖励给用户地址。
3. 当前金额:如果本次参加 staking 的金额大于 0，则从用户地址划转对应的金额到当前 lp 池子内，并且更新用户的 amount 值。
4. 辅助以后:重设 user 的起点金额，用 `rewardDebt` 保存。

#### leaveStaking: 退出 staking

在这里，用户的奖励 CAKE 和本金 CAKE 是分开发放的。

```
function leaveStaking(uint256 _amount) public {
    PoolInfo storage pool = poolInfo[0];
    UserInfo storage user = userInfo[0][msg.sender];
    require(user.amount >= _amount, "withdraw: not good");
    updatePool(0);
    uint256 pending = user.amount.mul(pool.accCakePerShare).div(1e12).sub(user.rewardDebt);
    if(pending > 0) {
        safeCakeTransfer(msg.sender, pending);
    }
    if(_amount > 0) {
        user.amount = user.amount.sub(_amount);
        pool.lpToken.safeTransfer(address(msg.sender), _amount);
    }
    user.rewardDebt = user.amount.mul(pool.accCakePerShare).div(1e12);

    syrup.burn(msg.sender, _amount);
    emit Withdraw(msg.sender, 0, _amount);
}
```

先校验用户在 staking 池内是否有足额的 amount 进行取出，否则报错

1. 首先更新池子 `accCakePerShare` 和 `lastRewardBlock`
2. 之前金额:获取用户的待发放奖励，如果大于零，则先把奖励发给用户。
3. 当前金额:如果需要退出的金额大于 0，则从`user.amount`中扣除，并把对应值的 CAKE 还给用户地址。
4. 辅助以后:像之前一样，重设 user 的起点金额，每次操作都需要更新 `user.rewardDebt`记录奖励起点

上面 4 步就是主要逻辑，这里的 staking 还有一个业务逻辑. 从用户地址上销毁掉本次提取 amount 等额的 syrup 币

#### updateStakingPool: 更新 staking 池

该方法是一个 internal 方法，只有 LP 的添加和设置`allocPoint`时才会调用该方法。该方法主要更新 `totalAllocPoint` 和 staking 的 allocPoint

```
function updateStakingPool() internal {
    uint256 length = poolInfo.length;
    uint256 points = 0;
    for (uint256 pid = 1; pid < length; ++pid) {
        points = points.add(poolInfo[pid].allocPoint);
    }
    if (points != 0) {
        points = points.div(3);
        totalAllocPoint = totalAllocPoint.sub(poolInfo[0].allocPoint).add(points);
        poolInfo[0].allocPoint = points;
    }
}
```

循环除 Staking 池以外的所有池子，累加他们的 allocPoint(分配点数) 赋值给 points，如果累加出来的 points 不等于零，则做如下操作

1.  totalAllocPoint = totalAllocPoint - Staking.allocPoint + points/3
2.  Staking.allocPoint = points/3

`TODO:` 这里为什么这样做，需要详细解释。

### LP 的操作方法

- 🔒`add(_allocPoint,_lpToken,_withUpdate)`
- 🔒`set(_pid,_allocPoint,_withUpdate)`
- `deposit(_pid,_amount)` : 将 LP 存入 MasterChef 进行挖矿。
- `withdraw(_pid,_amount)`: 从 MasterChef 中取回 LP

#### add: 添加 LP 池

将新的 lp 添加到池中，只能由所有者调用。

```
function add(uint256 _allocPoint, IBEP20 _lpToken, bool _withUpdate) public onlyOwner {
    if (_withUpdate) {
        massUpdatePools();
    }
    uint256 lastRewardBlock = block.number > startBlock ? block.number : startBlock;
    totalAllocPoint = totalAllocPoint.add(_allocPoint);
    poolInfo.push(PoolInfo({
        lpToken: _lpToken,
        allocPoint: _allocPoint,
        lastRewardBlock: lastRewardBlock,
        accCakePerShare: 0
    }));
    updateStakingPool();
}
```

---

这里有一个**辅助函数** `massUpdatePools`，内部的代码也很简单，作用是更新所有池子的奖励变量，调用之前介绍的`updatePool`。

```
function massUpdatePools() public {
    uint256 length = poolInfo.length;
    for (uint256 pid = 0; pid < length; ++pid) {
        updatePool(pid);
    }
}
```

每一个池子都会更新 `accCakePerShare` 以及 `lastRewardBlock`，并且 MINT 出 CAKE 转给 devaddr 和 syrup。所以 gas 费用比较高，这个代码初期的时候还可以全局执行，后期加入池子很多，很可能因为 gas 超出最大限制，导致没办法更新数据。

---

add 的核心是：组装 pool 信息，并 push 到 `poolInfo` 中。

lpToken 和 allocPoint 根据传入的值赋值，并更新 totalAllocPoint，accCakePerShare 默认为 0。lastRewardBlock 使用当前区块和 startBlock 对比，哪个值大就用哪个。

最后调用 `updateStakingPool()` 更新全局的 totalAllocPoint 和 staking.allocPoint

#### set: 设置 LP 池

该方法主要是更新 `pool.allocPoint`，通过调整它来进行 CAKE 币挖矿时，每个池子的瓜分权重，只能由所有者调用。

通过掉整 `allocPoint` 可以改变池子瓜分 CAKE 的权重，从而影响到 APR。

```
function set(uint256 _pid, uint256 _allocPoint, bool _withUpdate) public onlyOwner {
  if (_withUpdate) {
      massUpdatePools();
  }
  uint256 prevAllocPoint = poolInfo[_pid].allocPoint;
  poolInfo[_pid].allocPoint = _allocPoint;
  if (prevAllocPoint != _allocPoint) {
      totalAllocPoint = totalAllocPoint.sub(prevAllocPoint).add(_allocPoint);
      updateStakingPool();
  }
}
```

逻辑是取出之前的 allocPoint 供后面更新 totalAllocPoint。然后使用传入的`_allocPoint`更新当前值。设置 `totalAllocPoint` 后，再次调用`updateStakingPool()`更新 `staking.allocPoint`

#### deposit: 存 LP

该逻辑类似 `enterStaking`

```
function deposit(uint256 _pid, uint256 _amount) public {
    require (_pid != 0, 'deposit CAKE by staking');

    PoolInfo storage pool = poolInfo[_pid];
    UserInfo storage user = userInfo[_pid][msg.sender];

    updatePool(_pid);
    if (user.amount > 0) {
        uint256 pending = user.amount.mul(pool.accCakePerShare).div(1e12).sub(user.rewardDebt);
        if(pending > 0) {
            safeCakeTransfer(msg.sender, pending);
        }
    }
    if (_amount > 0) {
        pool.lpToken.safeTransferFrom(address(msg.sender), address(this), _amount);
        user.amount = user.amount.add(_amount);
    }
    user.rewardDebt = user.amount.mul(pool.accCakePerShare).div(1e12);
    emit Deposit(msg.sender, _pid, _amount);
}
```

该方法仅供 LP 进行存款，所以如果 pid 是 0，则会报错。

核心逻辑也是

1. 首先更新池子 `accCakePerShare` 和 `lastRewardBlock`
2. 之前金额:先判断之前是否有 LP 存款，如果有先把之前金额的待发 CAKE 奖励给发掉。
3. 当前金额:如果当前的存款 `_amount > 0` ,则从用户账号把金额拿到池子里来，并给用户的账号上加上当前存款金额。
4. 辅助以后:重设 user 的起点金额，用 `rewardDebt` 保存。

#### withdraw: 存 LP

该逻辑类似 `leaveStaking`，同样该方法也是仅供 LP 池使用的，如果是 Staking CAKE 需要使用 `leaveStaking`。

```
function withdraw(uint256 _pid, uint256 _amount) public {

    require (_pid != 0, 'withdraw CAKE by unstaking');
    PoolInfo storage pool = poolInfo[_pid];
    UserInfo storage user = userInfo[_pid][msg.sender];
    require(user.amount >= _amount, "withdraw: not good");

    updatePool(_pid);
    uint256 pending = user.amount.mul(pool.accCakePerShare).div(1e12).sub(user.rewardDebt);
    if(pending > 0) {
        safeCakeTransfer(msg.sender, pending);
    }
    if(_amount > 0) {
        user.amount = user.amount.sub(_amount);
        pool.lpToken.safeTransfer(address(msg.sender), _amount);
    }
    user.rewardDebt = user.amount.mul(pool.accCakePerShare).div(1e12);
    emit Withdraw(msg.sender, _pid, _amount);
}
```

首先判断用户在 LP 内的余额是否大于等于要提取的金额，如果余额不足报错。

核心逻辑类似之前的 `leaveStaking`,先 `updatePool(_pid)`更新池子的 accCakePerShare 和 lastRewardBlock。

1. 首先更新池子 `accCakePerShare` 和 `lastRewardBlock`
2. 之前金额:先把之前金额的待发 CAKE 奖励给发掉。
3. 当前金额:把用户要提取的 LP 数量还给用户，并给用户的账号上减去取款金额。
4. 辅助以后:重设 user 的起点金额，用 `rewardDebt` 保存。

### 更新方法

- `updateMultiplier(uint256 multiplierNumber)`
  - 更新 `BONUS_MULTIPLIER`
  - 🔒:只能部署者调用
- `dev(_devaddr)`
  - 更换用于接收 CAKE 奖励的地址

#### updateMultiplier:更新 CAKE 的奖金乘数

只能部署者调用

```
function updateMultiplier(uint256 multiplierNumber) public onlyOwner {
    BONUS_MULTIPLIER = multiplierNumber;
}
```

#### dev:更新接收 CAKE 奖励的地址

```
function dev(address _devaddr) public {
    require(msg.sender == devaddr, "dev: wut?");
    devaddr = _devaddr;
}
```

判断当前调用者是否为 `devaddr`,如果是则修改传入的地址，如果不是则报错.

### 辅助方法

**任何人都可以调用**的辅助方法。

- `poolLength()`
  - 获取合约内的池子总数量
- `pendingCake(uint256 _pid, address _user)`
  - 查看指定 user address 在指定池子下的未领取 CAKE 数量
- `safeCakeTransfer(_to, _amount)`
  - CAKE 的`safeTransfer`，防止因为舍入错误导致池子没有足够的 CAKE。

#### poolLength:获取合约内的池子总数量

```
function poolLength() external view returns (uint256) {
    return poolInfo.length;
}
```

#### pendingCake:未发放的 CAKE 值

查看指定 user address 在指定池子下的未领取 CAKE 数量

```
function pendingCake(uint256 _pid, address _user) external view returns (uint256) {
    PoolInfo storage pool = poolInfo[_pid];
    UserInfo storage user = userInfo[_pid][_user];
    uint256 accCakePerShare = pool.accCakePerShare;
    uint256 lpSupply = pool.lpToken.balanceOf(address(this));
    if (block.number > pool.lastRewardBlock && lpSupply != 0) {
        uint256 multiplier = getMultiplier(pool.lastRewardBlock, block.number);
        uint256 cakeReward = multiplier.mul(cakePerBlock).mul(pool.allocPoint).div(totalAllocPoint);
        accCakePerShare = accCakePerShare.add(cakeReward.mul(1e12).div(lpSupply));
    }
    return user.amount.mul(accCakePerShare).div(1e12).sub(user.rewardDebt);
}
```

因为没有更新池子，所以需要计算上一次发放奖励的数量+到当前块的 Cake 奖励。核心逻辑和 staking/LP 的存取一样。仅仅是只读方法。

#### safeCakeTransfer:CAKE 安全转转

CAKE 的`safeTransfer`，防止因为舍入错误导致池子没有足够的 CAKE。

```
function safeCakeTransfer(address _to, uint256 _amount) internal {
    syrup.safeCakeTransfer(_to, _amount);
}
```

`syrup.safeCakeTransfer` 的代码如下

```
function safeCakeTransfer(address _to, uint256 _amount) public onlyOwner {
    uint256 cakeBal = cake.balanceOf(address(this));
    if (_amount > cakeBal) {
        cake.transfer(_to, cakeBal);
    } else {
        cake.transfer(_to, _amount);
    }
}
```

- 如果合约内的余额大于等于传入的 amount，则使用 amount 值转账
- 如果合约内的余额小于传入的 amount，则使用合约内的余额进行转账。

### 逃生通道

`emergencyWithdraw(_pid)`: 取出指定池子内的所有本金，挖矿奖励不要了。仅限紧急情况使用，用作合约的资金逃生通道。

#### emergencyWithdraw:紧急撤离

```
function emergencyWithdraw(uint256 _pid) public {
    PoolInfo storage pool = poolInfo[_pid];
    UserInfo storage user = userInfo[_pid][msg.sender];
    pool.lpToken.safeTransfer(address(msg.sender), user.amount);
    emit EmergencyWithdraw(msg.sender, _pid, user.amount);
    user.amount = 0;
    user.rewardDebt = 0;
}
```

通过指定的 `_pid` 获取池子信息和用户信息。将用户地址下的 amount 还给用户。并将用户的 amount 和 rewardDebt 全部重置为 0。该方法不计算用户的 CAKE 收益。

假如遇到突发情况需要使用该方法，需要前端提示用户，该方法回只取本金，放弃挖矿奖励。仅仅作为资金的逃生通道。

### 迁移相关的方法

- `setMigrator(IMigratorChef _migrator)`:设置 migrator 合约，只能由 `MasterChef` 部署者调用
- `migrate(_pid)`: 将 lp 代币迁移到另一个 lp 合约。 任何人都可以调用。

#### setMigrator 接口

```
interface IMigratorChef {
    function migrate(IBEP20 token) external returns (IBEP20);
}
```

根据注释，作用如下：

- 执行从传统 PancakeSwap 到 CakeSwap 的 LP 代币迁移。
- 取当前 LP 代币地址，返回新的 LP 代币地址。
- 迁移者应该对调用者的 LP 令牌具有完全访问权限。
- 返回新的 LP 代币地址。
- XXX Migrator 必须有权访问 PancakeSwap LP 代币。
- CakeSwap 必须铸造完全相同数量的 CakeSwap LP 代币，否则会发生不好的事情。 传统的 PancakeSwap 不会这样做，所以要小心！

#### migrate

将 lp 代币迁移到另一个 lp 合约。 任何人都可以调用。

管理员会先设置迁移器，然后针对单个质押池进行迁移。迁移流程先对迁移器进行授权（safeApprove），后面执行由 migrator 控制，migrator 会返回一个新的 LPToken，然后重置质押池

```
function migrate(uint256 _pid) public {
    require(address(migrator) != address(0), "migrate: no migrator");
    PoolInfo storage pool = poolInfo[_pid];
    IBEP20 lpToken = pool.lpToken;
    uint256 bal = lpToken.balanceOf(address(this));
    lpToken.safeApprove(address(migrator), bal);
    IBEP20 newLpToken = migrator.migrate(lpToken);
    require(bal == newLpToken.balanceOf(address(this)), "migrate: bad");
    pool.lpToken = newLpToken;
}
```

- 首先要判断是否设置了 migrator 地址，然后根据传入的 pid，查找到
  `pool` 和 `lpToken`
- 通过 `lpToken` 进而获取到当前合约的 lpToken 余额。(用于后面转账和做验证)
- 因为我们的 migrator 合约需要把旧池子内的 LP 移到新池子，所以需要对 migrator 合约授权。
- 从当前合约将 lpToken 余额转到新合约中。
- 在新合约中获取新的地址 newLpToken
- ‼️ 重要：判断当前地址在新合约内的余额，是否等于之前旧合约的余额，如果余额不一样，报错，回滚数据。
- 一切都完美运行后，将新 LP 地址，写到 `pool.lpToken` 中。
