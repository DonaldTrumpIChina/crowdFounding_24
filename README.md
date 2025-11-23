CrowdFunding - 基于 Solidity 的众筹智能合约
项目简介
该项目是一个基于 Solidity ^0.8.28 开发的去中心化众筹智能合约，支持创建众筹项目、用户参与众筹、项目方领取资金、未达标时向 contributors 退款等核心功能。合约集成 OpenZeppelin 安全组件，确保资金操作安全性，并通过分批退款机制避免 gas 限制导致的 DOS 问题，适用于基于 ERC20 代币的各类众筹场景。
核心特性
安全可靠：使用 OpenZeppelin 的 SafeERC20 处理代币转账，避免重入攻击；通过 Ownable 实现权限管理，核心操作仅合约拥有者可执行。
功能完整：支持创建众筹项目、用户贡献代币、查看项目进度、达标领资、未达标分批退款等全流程功能。
gas 优化：退款功能限制单次循环最大处理 500 名贡献者，避免因贡献者过多导致 gas 超出区块限制。
清晰的错误处理：通过自定义错误提示替代字符串报错，降低 gas 消耗并提升调试效率。
状态透明：提供多个视图函数，支持查询项目进度、截止时间、用户贡献金额等关键信息。
环境依赖
编程语言：Solidity ^0.8.28
依赖库：
OpenZeppelin Contracts ^5.0.0（核心依赖）：
@openzeppelin/contracts/token/ERC20/IERC20.sol（ERC20 代币接口）
@openzeppelin/contracts/access/Ownable.sol（权限管理）
@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol（安全代币转账）
开发工具：Truffle/Hardhat（编译部署）、Remix（在线调试）
部署网络：EVM 兼容链（以太坊主网 / 测试网、BSC、Polygon 等）
合约部署
1. 准备工作
安装依赖（以 Hardhat 为例）：
bash
运行
npm install @openzeppelin/contracts@^5.0.0
确保部署账户拥有足够的 Gas 代币（如 ETH、BNB 等）。
2. 部署步骤
编译合约：
bash
运行
npx hardhat compile
部署合约（需指定 ERC20 代币地址，众筹将基于该代币进行）：
solidity
// 部署脚本示例（Hardhat）
const hre = require("hardhat");

async function main() {
  const tokenAddress = "0xYourERC20TokenAddress"; // 替换为目标ERC20代币地址
  const CrowdFunding = await hre.ethers.getContractFactory("CrowdFunding");
  const crowdFunding = await CrowdFunding.deploy(tokenAddress);

  await crowdFunding.waitForDeployment();
  console.log("CrowdFunding合约部署成功，地址：", await crowdFunding.getAddress());
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
执行部署命令：
bash
运行
npx hardhat run scripts/deploy.js --network <目标网络>
核心功能使用说明
前置条件
用户参与众筹前，需先给合约授权足够的 ERC20 代币（通过approve方法）。
核心操作（创建项目、领资、退款）仅合约拥有者（部署者）可执行。
关键函数说明
函数名	调用者	功能描述	参数说明	注意事项
startCrowdFunding(uint256 _target, uint256 _d)	合约拥有者	创建新的众筹项目	_target：众筹目标金额（代币单位）；_d：众筹时长（秒）	项目 ID 从 0 开始递增
contribute(uint256 _id, uint256 _amount)	任意用户	向指定项目贡献代币	_id：项目 ID；_amount：贡献金额（需 > 0）	需先授权合约，且在项目截止前调用
claimFunds(uint256 _id)	合约拥有者	领取达标项目的资金	_id：项目 ID	仅当项目达标（筹集金额≥目标）且已过截止时间时可调用
repayToken(uint256 _id)	合约拥有者	未达标项目向贡献者分批退款	_id：项目 ID	仅当项目未达标、已过截止时间且处于活跃状态时可调用；返回true表示所有贡献者已退款完毕
getCrowdFundingProgress(uint256 _id)	任意用户	查询项目进度	_id：项目 ID	返回（已筹集金额，目标金额）
getCrowdFundingDeadline(uint256 _id)	任意用户	查询项目截止时间	_id：项目 ID	返回时间戳（秒）
getUserContribution(uint256 _id, address _user)	任意用户	查询指定用户对项目的贡献金额	_id：项目 ID；_user：用户地址	-
getCrowdFundingToken()	任意用户	查询众筹使用的 ERC20 代币地址	-	-
getCrowdFundingStatus(uint256 _id)	任意用户	查询项目是否活跃	_id：项目 ID	-
操作流程示例
1. 创建众筹项目
bash
运行
# 拥有者创建一个目标1000代币、时长3600秒（1小时）的项目
crowdFunding.startCrowdFunding(1000, 3600)
2. 用户参与众筹
bash
运行
# 1. 用户授权合约使用100代币
erc20Token.approve(crowdFundingAddress, 100)

# 2. 向ID为0的项目贡献100代币
crowdFunding.contribute(0, 100)
3. 查看项目进度
bash
运行
# 查询ID为0的项目进度
crowdFunding.getCrowdFundingProgress(0)
# 返回示例：(500, 1000) 表示已筹集500，目标1000
4. 项目达标领资（拥有者）
bash
运行
# 项目截止后，若已达标，拥有者领取资金
crowdFunding.claimFunds(0)
5. 项目未达标退款（拥有者）
bash
运行
# 项目截止后，若未达标，拥有者执行分批退款
crowdFunding.repayToken(0)
# 若返回true，说明所有贡献者已退款；否则需多次调用直至返回true
错误说明
错误名称	触发场景
CrowdFunding_hasClosed	项目已过截止时间仍尝试贡献代币
CrowdFunding_zeroAmount	贡献金额为 0
CrowdFunding_crowdFundingActive	项目未结束或已退款完毕时调用repayToken
CrowdFunding_cannotClaim	项目未达标或未过截止时间时调用claimFunds
合约结构
solidity
CrowdFunding
├── 数据结构
│   └── Project：存储项目信息（目标金额、截止时间、已筹金额、活跃状态、贡献者信息等）
├── 常量与状态变量
│   ├── MAX_LOOP_LENGTH：单次退款最大处理贡献者数量（500）
│   ├── i_token：众筹使用的ERC20代币（不可变）
│   ├── s_projects：项目ID到Project的映射
│   └── index：下一个项目的ID（自增）
├── 修饰器
│   ├── beforeDeadline：限制仅在项目截止前调用
│   └── moreThanZero：限制金额大于0
├── 核心函数
│   ├── 构造函数：初始化代币地址和合约拥有者
│   ├── 公共函数：创建项目、贡献代币、领资、退款、查询状态等
│   └── 私有函数：_repay（实际执行退款逻辑）
└── 事件
    ├── ContributeMade：贡献代币时触发
    ├── FundsClaimed：领取资金时触发
    └── CrowdFundingStarted：创建项目时触发
注意事项
合约仅支持指定的 ERC20 代币，部署时需确认代币地址正确性。
用户参与众筹前必须先授权合约，否则contribute会失败。
退款功能需多次调用（若贡献者超过 500 人），直至返回true表示所有退款完成。
合约拥有者拥有核心操作权限，需确保私钥安全，避免权限泄露。
部署前建议在测试网（如 Sepolia、Goerli）进行完整功能测试，避免主网部署后出现问题。
众筹时长_d以秒为单位，创建时需注意时间换算（如 1 天 = 86400 秒）。
开源协议
本合约基于 MIT License 开源（ SPDX-License-Identifier: MIT ），可自由修改、分发和商用，但需保留原版权声明和许可协议。
