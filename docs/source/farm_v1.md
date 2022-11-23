# Pancake Farm V1 源码详解

Pancake Smart Contracts V1 版本 Farm 源码的详细解读

在 bscscan 上认证的名字是 **PancakeSwap: Main Staking Contract**。

- Github 地址: <https://github.com/pancakeswap/pancake-farm>
- BscScan 地址: [0x73feaa1ee314f8c655e354234017be2193c9e24e](https://bscscan.com/address/0x73feaa1ee314f8c655e354234017be2193c9e24e#code)

## 前置介绍

Pancake Farm 代码看起来是基于 SushiSwap 的 MasterChef 代码修改而来。

### 基本介绍

Pancake Farm 是一个奖励分发合约。可以通过存 CAKE 单币和 LP Token 两种途径来得到奖励。其中第 1 个挖矿池不是 LP Token，而是 Cake Token 这个单币种，可以存 CAKE 赚 CAKE，所以第一个池属于 Staking 概念

所有用户可以进行四种基础的资产操作

- LP 池:
  - 存款
  - 取款
- CAKE 池:
  - 进入 Staking
  - 离开 Staking

除了上面四种基本的操作以外，用户还可以使用 `emergencyWithdraw` 紧急取款的行为。 `emergencyWithdraw` 是一个逃生通道，不到万不得已不要使用；该功能是把用户的本金全部取出，但是不计算收益。

PancakeSwap Farm V1 版本的核心模块是: `MasterChef`，简称`MCV1`(MasterChef Version 1)，用户可以通过 MCV1 进行流动性挖矿。这个模块控制了两个 ERC20 代币的产出，分别是 **CAKE TOKEN** 和 **SYRUP TOKEN**。所以需要将这两个代币的 Owner 权限转移给 `MCV1` 。

## Farm 内奖励流转情况

CAKE 在 MCV1 中每个区块的产出量为: `40000000000000000000` wei (40 个 CAKE)，在构造函数中设置，部署完成以后不可修改；

在 MCV2 版本里，有"销毁"CAKE 概念，但是这个销毁并不是直接销毁，而是转入一个多签钱包的地址里，是否销毁需要监控多签钱包的地址来查看。

### CAKE 代币发放

除了用户赚取的 CAKE 外，池子每次更新时候，合约都会向 `devaddr` 以及 `syrup token` 这 2 个地址，转奖励相关数量的 Token。假设某个池子两个区块间的所有奖励是 `100 CAKE`，`MasterChef`会产出 `110 CAKE`

- 向 `devaddr` 转 `100/10` 个 CAKE (池子奖励的 `10%`)
- 向 `syrup token` 转 `100` 个 CAKE，

该部分的逻辑在 `updatePool` 这个更新池子系数的函数中实现；并且给 `devaddr` 和 `syrup token` 的 CAKE 资金，会在池子每次更新时都立刻 `mint` 发掉。

`syrup token` 地址收到的币，会在用户做四种基础的资产操作时，发该用户对应的奖励。

而 `devaddr` 是一个 GnosisSafe 的多签地址。

`devaddr` 地址说明:开发者部署 MCV1 合约后，就通过合约内 `dev(address _devaddr)` 方法，替换为了一个多签地址，该地址以后也可以随时替换，增加资金的追踪难度。 MCV1 中当前 devaddr 地址是: [0xceba60280fb0ecd9a5a26a1552b90944770a4a0e](https://bscscan.com/address/0xceba60280fb0ecd9a5a26a1552b90944770a4a0e)

#### CAKE 经济分配的修改

该部分是 Pancake 的逻辑，有些 Swap Fork Pancake 后，该部分逻辑改为：假设某个池子两个区块间的所有奖励是 `100 CAKE`，`MasterChef`会产出 `100 CAKE`

- 向 `devaddr` 转 `100/10` 个 CAKE (池子奖励的 `10%`)
- 向 `syrup token` 转 `100 - 100/10` 个 CAKE

也就是说开发者账号会瓜分挖出的奖励。这里的 10 也是可以修改的，比如 OKChain 上的 cherrySwap Farm 内的分配逻辑，是改为 20%；核心代码如下

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
    uint256 cherryReward = multiplier.mul(cherryPerBlock).mul(pool.allocPoint).div(totalAllocPoint);
    cherry.mint(devaddr, cherryReward.div(20));
    cherry.mint(address(syrup), cherryReward.sub(cherryReward.div(20)));
    pool.accCherryPerShare = pool.accCherryPerShare.add(cherryReward.sub(cherryReward.div(20)).mul(1e12).div(lpSupply));
    pool.lastRewardBlock = block.number;
}
```

源代码可以在 OKChain 的浏览器内查看到: https://www.oklink.com/zh-cn/okc/address/0x8cddB4CD757048C4380ae6A69Db8cD5597442f7b

### SYRUP 代币发放

- 当用户拿 CAKE 进入 Staking 的时候
  - MINT 出 CAKE 等量的 SYRUP 币，免费空投给用户
- 当用户取 CAKE 离开 Staking 的时候
  - 销毁用户账号上要取的 CAKE 币等量 SYURUP
  - 用户取多少 CAKE，就要还多少 SYUUP

SYRUP 代币除了给用户空投以及回收代币外，还承当向用户地址发 CAKE 奖励的功能。（CAKE 代币的来源是上面介绍的每次池子更新 `mint` 发放。）

为了防止用户地址计算奖励时候的精度误差，每次向用户转账 CAKE，使用 `safeCakeTransfer` 进行转账，防止 SYRUP 代币地址因为精度误差，不足以支付给用户奖励的情况。

## 数据结构及状态变量

MasterChef 中包含两个主要的数据结构: `UserInfo` 和 `PoolInfo`

#### UserInfo : userInfo

```
// 每个用户都有这个数据信息.
struct UserInfo {
    uint256 amount;     // 用户质押的LPToken数量
    uint256 rewardDebt; // 用户已经收到的奖励数量
}
// 每个持有 LP 代币的用户的信息
// mapping 中的 uint256 是池子的 pid
// mapping 中的 address 是用户的 address
mapping (uint256 => mapping (address => UserInfo)) public userInfo;
```

`rewardDebt` 这里做一些复杂的数学运算。保证任何时间点，等待分发给用户 CAKE 的数量是：`待发奖励 pending = (user.amount * pool.accCakePerShare) - user.rewardDebt`

每当用户做四种基础的资产操作，代码内做如下逻辑：

1. 池的 `accCakePerShare` 和 `lastRewardBlock` 进行更新。
   1. 该功能在 `updatePool()` 内实现 **更新池子系数**
2. 计算待发奖励，并直接转给用户地址。
   1. `Anbang注:`:该功能最好单独抽出成独立的函数，比如 `settlePendingCake()`。
   2. Pancake 这里写的有点啰嗦了
3. 更新用户的 `amount`
4. 更新用户的 `rewardDebt`

`rewardDebt` 是用户奖励的债务，并不是用户的已发放奖励，需要明确。比如用户存款 20，取款 5；此时计算收益的时候会按照 20 进行发放。用户离开了 5，还剩下 15；这里的 `rewardDebt` 的意思是，15 的奖励部分。在以后再计算的时候，需要减去这部分债务，因为 15 奖励已经发掉了，所以`rewardDebt`记录了用户的奖励债务值。同理，如果用户此时取款时 20，而不是 5，则债务为 0；因为用户已经没有剩余金额没有取出了。

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

- allocPoint :分配点数，质押池的分配比例，可以通过调整它来设置 LP 挖矿的权重
- lastRewardBlock :最后一次分配 CAKE 的块，代码计算时代表"上一次分配奖励的区块数"
- accCakePerShare :从池子创建时质押 1 个 wei LPToken,到目前为止；在当前池子的 CAKE 收益

**池子奖励计算方法如下**（2 个区块之间的奖励）：

- accCakePerShare 添加时候到默认值都是 0，更新方法如下
- 2 个区块之间的池子奖励是:
  - `cakeReward = 单个块的 CAKE 奖励 * (pool.allocPoint / totalAllocPoint) * 两个块之间的块数`
- 池子的总量是 lpSupply 个，则 1 wei LP 的在两个区块间的质押奖励是
  - `(cakeReward / lpSupply)`
- 用户的资产是一直放在池子内的，除了当前两个区块之间的奖励，之前的块也是有奖励的，所以这里的 pool.accCakePerShare 需要累加。
  - `pool.accCakePerShare = pool.accCakePerShare + (cakeReward * 1e12 / lpSupply)`
  - 这里引入 `1e12` 是为了精度，所以计算用户奖励以及给用户发奖励的时候，可以通过除以 `1e12` 得到真实值；
  - `Anbang注:`:引入精度时候，一般使用`1e18`运算，而且定义为一个常量。比如可以使用常量`ACC_CAKE_PRECISION = 1e18`做精度处理，每次需要用的时候使用变量即可。
  - 这里 Pancake 处理的不是很好。

**用户奖励计算方法如下**（2 个区块之间的奖励）：

用户依赖`user.amount`，`user.rewardDebt` 和 `pool.accCakePerShare`计算实际收益。原理如下:

- 用户在质押 LPToken 时，`user.amount` 结合 `pool.accCakePerShare` 记下来作为起始点位，表示已经发给用户的奖励（`user.rewardDebt`）.
- 用户在解押 LPToken 时，用`user.amount` 结合最新 `pool.accCakePerShare` 计算这么多 amount 的累积奖励。
  - 因为用户存入的钱并不是池子刚创建就第一笔存进来的，而 `accCakePerShare`是从池子创建时质押 1 wei LPToken 的收益。所以还需要去掉 `user.amount` 在当前区间的起始块已经收到的奖励。
- 2 个区块之间的实际收益：`user.amount * pool.accCakePerShare - user.rewardDebt`
  - 这里就是上面 UserInfo 部分的那个公式。

`TODO:` 上面计算的用户待发奖励结算的功能，封装在一个 `settlePendingCake()` 方法内，避免用户四次基础操作时候，都啰嗦的计算。

#### 其他状态变量

```
CakeToken public cake;  // CAKE  TOKEN
SyrupBar public syrup;  // SYRUP TOKEN 每当池子更新时，会将待发给池子的n个CAKE转给该地址

address public devaddr; // 开发者地址

uint256 public cakePerBlock;          // 每个区块产出的 CAKE 数量
uint256 public BONUS_MULTIPLIER = 1;  // CAKE 的奖金乘数

IMigratorChef public migrator;        // 迁移合约，权限很大。

uint256 public totalAllocPoint = 0;   // 总分配点数。必须是所有LP池的所有分配点数的总和。

uint256 public startBlock;            // 开始挖 CAKE 的区块号
```

- `devaddr` : 每当池子更新时，会额外增发 10% 给该地址
  - 需要项目方负责人给用于接受额外奖励的地址（最好多签钱包），部署时候就可以设置，后期可随时更新。
- `cakePerBlock`: 每个块的产出数量，部署合约时候在构造函数中传入。
  - 一旦设置，不可修改。Pancake 内是 40 个 CAKE 币/块的产出速度。
- `BONUS_MULTIPLIER` 是给区块数量增加权重的系数:
  - 而在计算两个块之间的块数时，代码用的是 `(toBlock - fromBlock) * BONUS_MULTIPLIER`，可以通过 `BONUS_MULTIPLIER` 值对块数量进行加权。合约内该值是 `1`.
  - 该值的意义是，通过模拟加速区块数量，来增加 CAKE 的产出。
    - 比如设置为 0 的时候，可以关闭 CAKE 的产出。
    - 设置为 2 的时候，可以让每天 CAKE 的产出为 `40 * 2 = 80 个`
  - `BONUS_MULTIPLIER` 可以在 `updateMultiplier` 中修改。（仅 owner 地址有权限）
- `migrator` 迁移合约的迁移器地址
  - pancake 似乎是基于 Sushi 改的，这里参考了的 SUSHI 的逻辑，Pancake 并没有从别的地方迁移 LP 过来的需求；当前 MCV1 合约内的该状态变量，是 0 地址。对于 Pancake 来说，这个迁移功能是把当前的 LP 迁移到其他合约内，但是因为`MCV1`拥有 CAKE 的铸币权，所以只有产出 CAKE，就不能完全废弃`MCV1`。
  - sushiswap 当时的逻辑：sushiswap 一开始借助的是 uniswap 的流动性，因此上面的 lpToken 传过来的其实是 UniswapPair，然后通过 UniswapPair 拿到具体的交易对里的两个 token，然后在 sushi 中创建 SushiSwapPair（都是 IUniswapV2Pair 接口的实现类），然后将用户在 Uniswap 的流动性赎回(先转给 Uniswap，然后调用 burn，注意这里 burn 的对象是 pair，这样 Uniswap 会把两个质押 token 还到 SushiSwapPair 的地址)，最后调用 SushiSwapPair 的 mint 给用户增发 SushiSwapPair 的流动性，从而完成用户流动性的迁移。
- `totalAllocPoint`: 记录全局的总分配点数，不可以修改，由每个池子内的分配点数决定。
- `startBlock`: 开始产出 CAKE 的区块号。
  - 在添加流动池的时候，用来赋值给 `lastRewardBlock`
    - `lastRewardBlock = block.number > startBlock ? block.number : startBlock;`
    - 在更新池参数，以及计算用户未发放 CAKE 时候有判断是否到 CAKE 产币区块。

## 构造函数

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

MVV1 部署时初始化的参数如下

```
_cake (address): 0x0e09fabb73bd3ade0a17ecc321fd13a19e81ce82
_syrup (address): 0x009cf7bc57584b7998236eff51b98a168dcea9b0
_devaddr (address): 0xb9fa21a62fc96cb2ac635a051061e2e50d964051
_cakePerBlock (uint256): 40000000000000000000
_startBlock (uint256): 703820
```

### 参数解析如下

- CakeToken: CAKE 代币地址
- SyrupBar: SYRUP 代币地址
- devaddr: 开发者地址，用于接收额外的 10% CAKE 奖励
- cakePerBlock: 每个区块产出的 CAKE 数量
- startBlock: 开始挖矿的区块号

注意这里的: `cakePerBlock` 是初始化之后无法修改的。如果做合约改写，这里需要注意下。除了上面 5 个值的操作，构造函数内还创建了 CAKE 的 Staking 池。

### 构造函数内创建 Staking Pool

把 CAKE Token 的地址作为 LP，并把该信息作为 poolInfo 的第 1 个池子进行 push。将 startBlock 设置为默认的 lastRewardBlock。CAKE 的 allocPoint 设置为 1000，totalAllocPoint 对应的设置为 1000。

## updatePool:更新池子内系数

**LP 的存-取**和**Staking 的进入-退出** 这四种用户的基础操作，都会触发更新池子的奖励变量。所以先介绍这个方法：

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
    cake.mint(address(syrup), cakeReward);  // 将该池子的收益转给 syrup ，供以后发给用户
    pool.accCakePerShare = pool.accCakePerShare.add(cakeReward.mul(1e12).div(lpSupply));
    pool.lastRewardBlock = block.number;
}
```

### 前置判断

该方法接受 pid 作为参数，用于更新指定 pid 的池子。

如果当前区块还没有到达开始挖矿的区块号，则退出；因为这个方法核心是发放 CAKE 代币并更新相关的计算系数，没到开始挖矿的区块，没必要计算。（staking 池在构造函数部署时，lastRewardBlock 设置为了 startBlock；LP 池添加的时候，当前区块与 startBlock 哪个值大用哪个 ）

如果已经开始产出 CAKE 币了，获取当前池子内的 LP 余额(lpSupply)，如果 lpSupply 为 0，则更新 lastRewardBlock 后退出函数。因为池子是空的，也没有必要分发 CAKE。

如果挖矿的已经开始了，并且 lpSupply 也大于零，则执行下面逻辑（该方法的核心功能）：

### 核心功能

`multiplier = getMultiplier(pool.lastRewardBlock, block.number)`:获取 from 到 to 的 加权区块数量。

```
function getMultiplier(uint256 _from, uint256 _to) public view returns (uint256) {
    return _to.sub(_from).mul(BONUS_MULTIPLIER);
}
```

不单单是区块总数，如果 `BONUS_MULTIPLIER` 为 0,则返回 0，如果为 2，则返回两倍的区块数量，当前 Pancake 中 `BONUS_MULTIPLIER` 为 1，所以返回的就是 from 到 to 一共多少个区块数量

**计算该区块区间内池子的奖励数**：

当前 LP 池每一个块可以获取的奖励等于`一个块的产出CAKE数量 * (池子的分配点数/总分配点数)`，换成代码就是 `cakePerBlock * (pool.allocPoint/totalAllocPoint)`。

那么当前 LP 池从上一次发放奖励的区块到当前区块，合计挖出来的奖励就是 `cakeReward = multiplier * (cakePerBlock * (pool.allocPoint/totalAllocPoint))`

下面是业务代码，基于 LP 池的奖励，mint 出 CAKE 转给 devaddr 和 syrup。

```
cake.mint(devaddr, cakeReward.div(10)); // 增发奖励的 10% CAKE 给 devaddr
cake.mint(address(syrup), cakeReward);  // 将该池子的收益转给 syrup ，供以后发给用户
```

然后更新 `pool.accCakePerShare` 和 `pool.lastRewardBlock`，因为用户存的 LP 值相比`lpSupply` 可能很小，所以这里引入了精度`1e12`，这里按照编码规范应该使用常量，并且精度值应该使用`1e18`。

### 方法总结

如果执行此方法，将池子从上次发放奖励区块到当前块的收益发给 syrup，并且更新对应 pid 池子的 `accCakePerShare` 和 `lastRewardBlock`。

因为用户的存和取都会触发该方法，所以该池子只要有人使用，就会很频繁的更新和发放 CAEKE。

## CAKE Staking（重要）

`MCV1`的第一个池子就是 CAKE Staking，储存在 `poolInfo` 的第 0 项。相关功能是以下三个方法。

- `enterStaking(_amount)` : 存入 CAKE 币:质押指定金额进行挖矿
- `leaveStaking(_amount)` : 取出 CAKE 币:解押指定金额退出挖矿
- `updateStakingPool()` : 更新 Staking 池
  - ⚠️: `internal` 方法

### enterStaking: 参加 staking

```
function enterStaking(uint256 _amount) public {
    PoolInfo storage pool = poolInfo[0];
    UserInfo storage user = userInfo[0][msg.sender];
    updatePool(0); // 更新池子系数
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

分为三步走，下面三步的代码思路在用户四种基础操作会频繁用到:

1. 处理池子之前的数据: 奖励发掉，并把池子的信息更新为当前块
2. 处理用户之前的数据: 奖励发掉
3. 处理用户当前的数据:
   1. user 的账单记录
   2. pool 的代币处理

#### 1.处理池子之前的数据

首先进行 `updatePool(0)`,将 staking 池子之前奖励发掉，并把 `accCakePerShare` 和 `lastRewardBlock` 信息更新为当前块。

#### 2.处理用户之前的数据

如果在本次存入 CAKE 前，用户在 staking 里还有金额；则需要发放本次到上一次之间的 CAKE 奖励给用户地址。

- `user.rewardDebt` 是用户已经获取到的收益。是累积的值
- `pool.accCakePerShare` 是池子里 1wei 的累计收益
- 通过 `user.amount * pool.accCakePerShare` 就可以得到之前金额的累计收益；因为这个值是累积的收益值，所以还需要减掉之前已经累积发掉的奖励金额
- 所以用户本次和上次之间的收益如下(记得 accCakePerShare 需要除以精度)：
  ```
  uint256 pending = user.amount.mul(pool.accCakePerShare).div(1e12).sub(user.rewardDebt);
  ```
- 如果 pending 的值大于 0，把该金额发给用户地址。

#### 3.处理用户当前的数据

如果用户本次参加 staking 的金额大于 0，则从用户地址划转对应的金额到当前 lp 池子内，并且更新用户的 amount 值。

此时需要前端开发者注意在 staking 前需要提示用户需要将 CAKE 的 amount 金额授权给 lp 合约（为了以后省事，也可以授权一个最大者）。

---

完成以上操作后，使用当前的 `user.amount` 和 `pool.accCakePerShare` ，再次记录起点，方便下一次进行奖励转账

```
user.rewardDebt = user.amount.mul(pool.accCakePerShare).div(1e12);
```

#### 业务逻辑

上面主要逻辑完成后，就开始做业务逻辑。MINT 与用户 CAKE 币等量 syrup 币，空头给用户地址。

注意，这个空投的币，在退出 staking 的时候需要从用户的地址上销毁等量金额的 syrup 币，如果退出的时候，账号地址上没有 syrup 币，没办法退出的。

### leaveStaking: 退出 staking

在这里，用户的奖励 CAKE 和本金 CAKE 是分开发放的。因为给用户本金归还的合约是当前的 lp 池合约；给用户发奖励的是 syrup 代币合约。

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

资金操作也是分为三步走:

1. 处理池子之前的数据: 奖励发掉，并把池子的信息更新为当前块
   1. 更新池子 `accCakePerShare` 和 `lastRewardBlock`
2. 处理用户之前的数据: 奖励发掉
   1. 获取用户的待发放奖励，如果大于零，则先把奖励发给用户。
3. 处理用户当前的数据:
   1. user 的账单记录
   2. pool 的代币处理

第三步，处理用户当前的数据中，当前金额如果需要退出的金额大于 0，则从`user.amount`中扣除，并把对应值的 CAKE 还给用户地址。像之前一样，重设 user 的起点金额，每次操作都需要更新 `user.rewardDebt`记录奖励起点

剩下还有一个业务逻辑，从用户地址上销毁掉本次提取 amount 等额的 syrup 币。

### updateStakingPool: 更新 staking 池

该方法是一个 internal 方法，只有 LP 的添加和设置 `allocPoint` 时才会调用该方法。该方法主要更新 `totalAllocPoint` 和 `staking.allocPoint`

```
function updateStakingPool() internal {
    uint256 length = poolInfo.length;
    uint256 points = 0;

    // 计算除 staking 池以外的所有分配点
    for (uint256 pid = 1; pid < length; ++pid) {
        points = points.add(poolInfo[pid].allocPoint);
    }

    if (points != 0) {
        // points 自取三分之一的值
        points = points.div(3);

        // 下面操作，可以让 staking 池永远瓜分总产出的 25% CAKE
        totalAllocPoint = totalAllocPoint.sub(poolInfo[0].allocPoint).add(points);
        poolInfo[0].allocPoint = points;
    }
}
```

这是一个可以让 staking 池永远瓜分 CAKE 总产出的 25% 的方法，因为 LP 池是不断加入，可能随时调整分配点的。所以需要这个方法来控制 staking 的分配点数，从而保证 staking 池内资金无论多少，永远占 25% 的奖励。

---

## LP 相关操作（重要）

- 🔒`add(_allocPoint,_lpToken,_withUpdate)`
- 🔒`set(_pid,_allocPoint,_withUpdate)`
- `deposit(_pid,_amount)` : 将 LP 存入 MasterChef 进行挖矿。
- `withdraw(_pid,_amount)`: 从 MasterChef 中取回 LP

### 🔒 add: 添加 LP 池

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

这里有一个**辅助函数** `massUpdatePools`，内部的代码也很简单，作用是更新所有池子的奖励变量，调用的是之前介绍的 `updatePool` 方法。

```
function massUpdatePools() public {
    uint256 length = poolInfo.length;
    for (uint256 pid = 0; pid < length; ++pid) {
        updatePool(pid);
    }
}
```

该方法会更新每个池子的 `accCakePerShare` 以及 `lastRewardBlock`，并且 MINT 出 CAKE 转给 devaddr 和 syrup。所以 gas 费用比较高，这个代码初期的时候还可以全局执行，后期加入池子很多，很可能因为 gas 超出最大限制，导致没办法更新数据。

---

add 的核心是：组装 pool 信息，并 push 到 `poolInfo` 中。

lpToken 和 allocPoint 根据传入的值赋值，并更新 totalAllocPoint。lastRewardBlock 使用当前区块和 startBlock 对比，哪个值大就用哪个。

最后调用 `updateStakingPool()` 更新 staking 池，让 Staking 池永远瓜分 25%的 CAKE 产出。

### 🔒 set: 设置 LP 池

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

该方法主要是更新 `pool.allocPoint`，通过调整它来进行 CAKE 币挖矿时的每个池子瓜分权重，只能由所有者调用。

注意：这里的分配，在链下计算的时候，总 LP 池是 CAKE 的 75%总量的再分配，需要考虑 Staking 池的权重。

### deposit: 存 LP

该逻辑类似 `enterStaking`，该方法仅供 LP 进行存款，所以如果 pid 是 0，则会报错。

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

核心逻辑也是上面介绍的三步：

1. 处理池子之前的数据: 奖励发掉，并把池子的信息更新为当前块
2. 处理用户之前的数据: 奖励发掉
3. 处理用户当前的数据:
   1. user 的账单记录
   2. pool 的代币处理

### withdraw: 存 LP

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

首先判断用户在 LP 内的余额是否大于等于要提取的金额，如果余额不足则报错。

核心逻辑也是上面介绍的三步：

1. 处理池子之前的数据: 奖励发掉，并把池子的信息更新为当前块
2. 处理用户之前的数据: 奖励发掉
3. 处理用户当前的数据:
   1. user 的账单记录
   2. pool 的代币处理

## 逃生通道

所有资金相关的合约，都需要有预留逃生通道的基思维，防治极端情况的发生。

MCV1 的逃生方法是`emergencyWithdraw(_pid)`: 取出指定池子内的所有本金，挖矿奖励不要了。仅限紧急情况使用。

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

假如遇到突发情况需要使用该方法，前端一定要做好提示，该方法回只取本金，放弃挖矿奖励。前端开发者的正常逻辑里，不应该使用该方法。

## 更新方法

- `updateMultiplier(uint256 multiplierNumber)`
  - 更新 `BONUS_MULTIPLIER`，🔒:只能部署者调用
- `dev(_devaddr)`
  - 更换用于接收 10% CAKE 奖励的地址

### updateMultiplier:更新 CAKE 的奖金乘数

只能部署者调用，该方法是对区块数量的加权。如果为 0，则会导致 CAKE 产出为 0。如果为 2，则 CAKE 产出加倍。目前 Pancake 内的 `BONUS_MULTIPLIER` 值为 1.

```
function updateMultiplier(uint256 multiplierNumber) public onlyOwner {
    BONUS_MULTIPLIER = multiplierNumber;
}
```

### dev:更新接收 CAKE 奖励的地址

```
function dev(address _devaddr) public {
    require(msg.sender == devaddr, "dev: wut?");
    devaddr = _devaddr;
}
```

判断当前调用者是否为 `devaddr`，如果是则修改传入的地址，如果不是则报错.

## 辅助方法

- `poolLength()`
  - 获取合约内的池子总数量
- `pendingCake(uint256 _pid, address _user)`
  - 查看指定 user address 在指定池子下的未发放 CAKE 数量

#### poolLength:获取合约内的池子总数量

该方法需要注意，包含 Staking pool，计算 LP 池总量的时候，需要 `poolLength() - 1`;

```
function poolLength() external view returns (uint256) {
    return poolInfo.length;
}
```

#### pendingCake:未发放的 CAKE 值

查看指定 user address 在指定池子下的未发放 CAKE 数量，不更新池子系数，仅只读。

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

## 迁移相关

`MCV1` 也从来没有用过该功能。`migrator` 的地址都是 0 地址。

- `setMigrator(IMigratorChef _migrator)`:设置 migrator 合约，只能由 `MasterChef` 部署者调用
- `migrate(_pid)`: 将 lp 代币迁移到另一个 lp 合约。 任何人都可以调用。

### migrate

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

## syrup CAKE 安全转账

syrup 合约中的 CAKE `safeTransfer`，防止因为舍入错误导致池子没有足够的 CAKE。

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

## 事件相关

状态变量的更改，以及合约中用户资金的变动都需要抛出事件，这是写合约的基本觉悟。

`Anbang注`:但是 Pancake 中，仅对用户资金存取变动抛出事件，其他都没有抛出事件；像 LP pool 的添加，修改；以及 `devaddr` `migrator` 和 `BONUS_MULTIPLIER` 的修改方法内都需要抛出事件，这是最基本的编码规范。
