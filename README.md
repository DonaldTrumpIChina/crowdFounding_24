# CrowdFunding - ERC20 众筹合约

## 项目概述

CrowdFunding 是一个基于 Solidity 的去中心化众筹合约，支持使用 ERC20 代币进行项目众筹。合约具备创建众筹项目、用户参与众筹、项目方领取资金、未达标退款等核心功能，并通过分批退款机制优化 Gas 成本，避免区块 Gas 限制导致的 DoS 攻击。

## 核心特性



* 支持创建多个独立众筹项目，每个项目有明确的筹款目标和截止时间

* 仅合约所有者可创建项目、领取资金和发起退款

* 参与者通过 ERC20 代币参与众筹，资金安全由 OpenZeppelin 的 SafeERC20 保障

* 众筹达标且截止后，项目方可领取全部筹款

* 众筹未达标且截止后，支持分批向参与者退款（避免 Gas 超限）

* 完善的错误处理和事件通知机制

* 关键操作权限控制（基于 OpenZeppelin Ownable）

## 技术依赖



* Solidity ^0.8.28（兼容 SafeMath 内置特性）

* OpenZeppelin Contracts：


  * `IERC20`：ERC20 代币标准接口

  * `Ownable`：权限管理

  * `SafeERC20`：安全的 ERC20 转账操作

## 合约架构

### 数据结构



| 结构名    | 字段                   | 说明                                   |
| --------- | ---------------------- | -------------------------------------- |
| `Project` | `targetAmount`         | 众筹目标金额（ERC20 代币数量）         |
|           | `deadline`             | 众筹截止时间戳                         |
|           | `raisedAmount`         | 已筹集金额                             |
|           | `isCrowdFundingActive` | 众筹项目状态（是否活跃）               |
|           | `lastContributorIndex` | 退款进度索引（记录已退款的参与者位置） |
|           | `contributions`        | 参与者地址到贡献金额的映射             |
|           | `contributors`         | 参与者地址列表                         |

### 核心状态变量



* `MAX_LOOP_LENGTH`：单次退款最大处理人数（默认 500，避免 Gas 超限）

* `i_token`：用于众筹的 ERC20 代币合约地址（不可变）

* `s_projects`：项目 ID 到项目详情的映射

* `index`：下一个新项目的 ID（自增生成）

### 错误类型



| 错误标识                          | 触发场景                             |
| --------------------------------- | ------------------------------------ |
| `CrowdFunding_hasClosed`          | 众筹已截止仍尝试参与                 |
| `CrowdFunding_zeroAmount`         | 参与众筹时转账金额为 0               |
| `CrowdFunding_crowdFundingActive` | 众筹未结束或不满足退款条件时发起退款 |
| `CrowdFunding_cannotClaim`        | 未达标或未截止时尝试领取资金         |

### 事件通知



| 事件名                | 参数                                                         | 说明                   |
| --------------------- | ------------------------------------------------------------ | ---------------------- |
| `ContributeMade`      | `_index`（项目 ID）、`_contributor`（参与者地址）、`_amount`（贡献金额） | 参与者成功参与众筹     |
| `FundsClaimed`        | `_index`（项目 ID）、`_amount`（领取金额）                   | 项目方成功领取众筹资金 |
| `CrowdFundingStarted` | `_index`（项目 ID）、`_targetAmount`（目标金额）、`_duration`（众筹时长） | 新众筹项目创建成功     |

## 核心功能

### 1. 合约部署



```
constructor(address token) Ownable(msg.sender)
```



* 功能：初始化众筹合约，指定用于众筹的 ERC20 代币地址

* 参数：`token` - ERC20 代币合约地址

* 权限：部署者自动成为合约所有者

### 2. 创建众筹项目（仅所有者）



```
function startCrowdFunding(uint256 \_target, uint256 \_d) public onlyOwner
```



* 功能：创建新的众筹项目

* 参数：


  * `_target` - 众筹目标金额（ERC20 代币数量）

  * `_d` - 众筹时长（秒）

* 逻辑：生成新项目 ID，初始化项目状态，发射 `CrowdFundingStarted` 事件

### 3. 参与众筹（公开）



```
function contribute(uint256 \_id, uint256 \_amount) public moreThanZero(\_amount) beforeDeadline(\_id)
```



* 功能：用户向指定项目贡献 ERC20 代币

* 参数：


  * `_id` - 项目 ID

  * `_amount` - 贡献金额（需大于 0）

* 前置条件：


  * 众筹未截止（`beforeDeadline` 修饰器）

  * 贡献金额大于 0（`moreThanZero` 修饰器）

* 逻辑：

1. 首次参与的用户加入参与者列表

2. 记录用户贡献金额和项目总筹集金额

3. 从用户地址安全转账 ERC20 代币到合约

4. 发射 `ContributeMade` 事件

### 4. 领取众筹资金（仅所有者）



```
function claimFunds(uint256 \_id) public onlyOwner
```



* 功能：众筹达标且截止后，项目方领取全部筹集资金

* 参数：`_id` - 项目 ID

* 前置条件：


  * 已筹集金额 ≥ 目标金额

  * 众筹已截止

* 逻辑：

1. 标记项目为非活跃状态

2. 向合约所有者转账全部筹集资金

3. 发射 `FundsClaimed` 事件

### 5. 分批退款（仅所有者）



```
function repayToken(uint256 \_id) public onlyOwner returns(bool)
```



* 功能：众筹未达标且截止后，向参与者分批退款（避免 Gas 超限）

* 参数：`_id` - 项目 ID

* 前置条件：


  * 已筹集金额 < 目标金额

  * 众筹已截止

  * 项目处于活跃状态

* 逻辑：

1. 每次最多处理 `MAX_LOOP_LENGTH` 个参与者的退款

2. 更新退款进度索引

3. 全部退款完成后标记项目为非活跃状态

4. 返回值：`true` 表示所有参与者已完成退款，`false` 表示仍有未退款参与者

### 6. 查看项目信息（公开视图函数）



| 函数名                    | 参数                                  | 返回值                                         | 说明                                |
| ------------------------- | ------------------------------------- | ---------------------------------------------- | ----------------------------------- |
| `getCrowdFundingProgress` | `_id`（项目 ID）                      | `(uint256 raisedAmount, uint256 targetAmount)` | 项目筹款进度（已筹金额 / 目标金额） |
| `getCrowdFundingToken`    | 无                                    | `address`                                      | 众筹使用的 ERC20 代币地址           |
| `getCrowdFundingDeadline` | `_id`（项目 ID）                      | `uint256`                                      | 项目截止时间戳                      |
| `getCrowdFundingStatus`   | `_id`（项目 ID）                      | `bool`                                         | 项目是否活跃                        |
| `getUserContribution`     | `_id`（项目 ID）、`_user`（用户地址） | `uint256`                                      | 特定用户对项目的贡献金额            |

## 使用流程

### 1. 项目方操作流程



1. 部署 `CrowdFunding` 合约，指定众筹用 ERC20 代币地址

2. 调用 `startCrowdFunding(_target, _d)` 创建众筹项目（获取项目 ID）

3. 向参与者宣传项目 ID 和众筹信息

4. 众筹截止后：

* 若达标：调用 `claimFunds(_id)` 领取资金

* 若未达标：多次调用 `repayToken(_id)` 直至全部退款完成（返回 `true`）

### 2. 参与者操作流程



1. 批准合约使用自己的 ERC20 代币（调用 ERC20 的 `approve(contractAddress, amount)`）

2. 调用 `contribute(_id, _amount)` 参与指定项目众筹

3. 众筹结束后：

* 若项目达标：等待项目方领取资金

* 若项目未达标：等待项目方发起退款，资金将自动退回自己的钱包

## 安全注意事项



1. 权限控制：仅合约所有者可创建项目、领取资金和发起退款，确保资金安全

2. 资金安全：使用 OpenZeppelin 的 `SafeERC20` 进行代币转账，避免转账失败导致的资金锁定

3. Gas 优化：通过 `MAX_LOOP_LENGTH` 限制单次退款的参与者数量，避免区块 Gas 限制导致的 DoS 攻击

4. 输入验证：所有关键操作都有前置条件检查（如金额非零、众筹状态有效等）

5. 事件追踪：所有核心操作都有对应的事件，方便前端追踪和审计

## 许可证

本合约基于 MIT 许可证开源，详见合约头部 `SPDX-License-Identifier: MIT`。

