An ERC20 staking system where users stake GopherToken and earn block-based rewards using MasterChef-style reward accounting.

Instead of looping through all stakers, it stores one global reward index and one per-user debt value.

### Create project
```shell
$ forge init token-staking-foundry
$ cd token-staking-foundry
```

### Create `GopherToken.sol`
```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.28;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract GopherToken is ERC20 {
    constructor() ERC20("GopherToken", "GPT") {
        _mint(msg.sender, 1000000 * 10 ** decimals());
    }
}
```

```shell
$ forge install OpenZeppelin/openzeppelin-contracts
```

Create remapping `remappings.txt` to translate Solidity import paths into actual filesystem paths
```
@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/
forge-std/=lib/forge-std/src/
```
### Create `GopherStaking.sol`
See https://github.com/LiamZhuangDev/token-staking-foundry/commit/8fcdc9ed30af6503ccfb1f2fce14ccb22c41f3fd
```
Formula to calculate pending rewards = (user.amount * accRewardPerShare) / decimals - user.rewardDebt
```

### Build

```shell
$ forge build
```

### Format

```shell
$ forge fmt
```

### Deploy
```
src/
├── GopherToken.sol
└── GopherStaking.sol

script/
└── Deploy.s.sol

foundry.toml
```
- Start Anvil local node
  ```shell
  anvil
  ```
- Update vars in `.env`
  ```
  DEPLOYER_PRIVATE_KEY=0x<PRIVATE_KEY_ANVIL_PROVIDED>
  RPC_URL=http://127.0.0.1:8545
  ```
- Create `Deploy.s.sol`
  ```solidity
  uint256 deployerPrivateKey = vm.envUint("DEPLOYER_PRIVATE_KEY");

  vm.startBroadcast(deployerPrivateKey);

  // deploy GopherToken
  GopherToken token = new GopherToken();

  // deploy GopherStaking
  uint256 rewardPerBlock = 1 ether;
  GopherStaking staking = new GopherStaking(address(token), rewardPerBlock);

  vm.stopBroadcast();
  ```
- Run forge script to deploy
  ```shell
  # load environment vars from the .env file into the current shell session
  $ source .env

  $ forge script script/Deploy.s.sol:Deploy --rpc-url $RPC_URL --broadcast
  ```

### Interaction

```shell
# Approve staking contract
# !!!must run first or reverted due to ERC20InsufficientAllowance
cast send <TOKEN_ADDRESS> \
  "approve(address, uint256)" \
  <STAKING_ADDRESS or PROXY_ADDRESS> \
  100000000000000000000 \
  --private-key $DEPLOYER_PRIVATE_KEY \
  --rpc-url $RPC_URL

# Stake
cast send <STAKING_ADDRESS or PROXY_ADDRESS> \
  "stake(uint256)" \
  100000000000000000000 \
  --private-key $DEPLOYER_PRIVATE_KEY \
  --rpc-url $RPC_URL

# Claim rewards only
cast send <STAKING_ADDRESS or PROXY_ADDRESS> \
  "withdraw(uint256)" \
  0 \
  --private-key $DEPLOYER_PRIVATE_KEY \
  --rpc-url $RPC_URL

# Check Deployer Token Balance
cast call <TOKEN_ADDRESS> \
  "balanceOf(address)(uint256)" \
  <DEPLOYER_ADDRESS> \
  --rpc-url $RPC_URL

# Withdraw stake
cast send <STAKING_ADDRESS or PROXY_ADDRESS> \
  "withdraw(uint256)" \
  50000000000000000000 \
  --private-key $DEPLOYER_PRIVATE_KEY \
  --rpc-url $RPC_URL
```

### Test
- Create `GopherStaking.t.sol` 
  ```solidity
  contract GopherStakingTest is Test {
      GopherToken public token;
      GopherStaking public staking;
      address public owner = address(this);
      address public alice = address(1);
      address public bob = address(2);
      uint256 public constant INITIAL_USER_BALANCE = 1_000 ether;
      uint256 public constant REWARD_PER_BLOCK = 1 ether;
      
      function setUp() public {
          // deploy contracts
          token = new GopherToken();
          staking = new GopherStaking(address(token), REWARD_PER_BLOCK);
          // give users tokens
          token.transfer(alice, INITIAL_USER_BALANCE);
          token.transfer(bob, INITIAL_USER_BALANCE);
          // fund staking contract with rewards
          uint256 rewardPool = 100_000 ether;
          token.transfer(address(staking), rewardPool);
      }

      // setUp when using UUPS proxy pattern
      function setUp() public {
          // deploy token
          token = new GopherToken();

          // deploy staking implementation
          GopherStaking implementation = new GopherStaking();

          // encode initialize call
          bytes memory initData = abi.encodeCall(GopherStaking.initialize, (admin, address(token), REWARD_PER_BLOCK));

          // deploy proxy
          ERC1967Proxy proxy = new ERC1967Proxy(address(implementation), initData);

          // interact with proxy as staking contract
          staking = GopherStaking(address(proxy));

          // give users tokens
          token.transfer(alice, INITIAL_USER_BALANCE);

          token.transfer(bob, INITIAL_USER_BALANCE);

          // fund staking contract with rewards
          uint256 rewardPool = 100_000 ether;

          token.transfer(address(staking), rewardPool);
      }

      /*//////////////////////////////////////////////////////////////
                              STAKE TESTS
      //////////////////////////////////////////////////////////////*/
      function testStake() public {
          uint256 amount = 100 ether;
          vm.startPrank(alice);
          token.approve(address(staking), amount);
          staking.stake(amount);
          vm.stopPrank();

          (uint256 stakedAmount, uint256 rewardDebt) = staking.users(alice);

          assertEq(stakedAmount, amount);
          assertEq(rewardDebt, 0);
          assertEq(staking.totalStaked(), amount);
          
          // alice wallet balance reduced
          assertEq(
              token.balanceOf(alice),
              INITIAL_USER_BALANCE - amount
          );
      }

      // other tests
  }
  ```
- Run forge test
  ```shell
  $ forge test
  ```
- Run test with Gas Report
  ```shell
  forge test --gas-report
  ```
- Test Coverage
  ```shell
  $ forge coverage
  ```
- How to find missing branches
  - Generate coverage file
  ```shell
  $ forge coverage --report debug > coverage.txt
  ```
  - and look for `hits: 0` in `coverage.txt`
  ```
  Branch (branch: 1, path: 0)
  hits: 0
  ```
  - add test function to improve the coverage

### Gas Snapshots
- creates a gas usage snapshot file for your tests, mainly used for detecting gas regressions
  ```shell
  $ forge snapshot
  ```
---
### Upgrade to use the Universal Upgradeable Proxy Standard (UUPS) proxy pattern
- Replace constructor with `initialize()`
- Inherit from OZ contracts:
  - Initializable
  - OwnableUpgradeable
  - UUPSUpgradeable
- Add `_authorizeUpgrade()`

### Updated Foundry Deploy Script for UUPS
You now deploy:
- implementation
- ERC1967Proxy
- initialize through proxy
```
User
  ↓
Proxy
  ↓ delegatecall
Implementation

Vars are defined by the implementation contract layout, but their actual values are stored in the proxy because of delegatecall.

delegatecall literally means:
Execute another contract’s bytecode in my storage context.

| Thing            | Value          |
| ---------------- | -------------- |
| Code executed    | Implementation |
| Storage modified | Proxy          |
| address(this)    | Proxy          |
| msg.sender       | User           |

```
### How UUPS proxy works when users call `stake` function
```solidity
// treat proxy as staking contract
GopherStaking staking = GopherStaking(address(proxy));
```

```
USER CALLS:

staking.stake(100)

IMPORTANT:
staking == proxy address
NOT implementation address

┌──────────────────────────┐
│ Solidity ABI Encoding    │
└──────────────────────────┘

Solidity converts:

stake(100)

into calldata:

0xa694fc3a + encoded(100)

where:
0xa694fc3a = function selector for "stake(uint256)"


========================================================
STEP 1 — EVM CALLS THE PROXY
========================================================

User
  │
  │ CALL proxyAddress
  │ calldata = stake(100)
  ▼

┌──────────────────────────┐
│ ERC1967Proxy             │
│ address = 0xPROXY        │
└──────────────────────────┘


Proxy receives:

stake(uint256)


BUT...

Proxy contract itself does NOT contain:

function stake(...)

So Solidity/EVM executes:

fallback()


========================================================
STEP 2 — PROXY FALLBACK EXECUTES
========================================================

┌──────────────────────────┐
│ fallback()               │
└──────────────────────────┘

fallback() does roughly:

implementation = storageSlot[IMPLEMENTATION_SLOT]

delegatecall(
    implementation,
    original calldata
)


So now:

delegatecall(
    0xIMPLEMENTATION,
    "stake(100)"
)


========================================================
STEP 3 — IMPLEMENTATION CODE RUNS
========================================================

┌──────────────────────────┐
│ GopherStaking            │
│ address = 0xIMPL         │
└──────────────────────────┘

Implementation code executes:

function stake(uint256 amount) {
    totalStaked += amount;
}


CRITICAL PART:

Because this is DELEGATECALL:

storage writes happen in PROXY storage


So:

totalStaked += 100

actually means:

proxy.slotX += 100


NOT:

implementation.slotX += 100


========================================================
FINAL RESULT
========================================================

Implementation storage:
┌───────────────┐
│ totalStaked=0 │  ← unchanged
└───────────────┘


Proxy storage:
┌─────────────────┐
│ totalStaked=100 │  ← actual state
└─────────────────┘


========================================================
VERY IMPORTANT MENTAL MODEL
========================================================

Implementation:
    CODE ONLY

Proxy:
    STATE/STORAGE ONLY

delegatecall means:

"Run implementation CODE against proxy STORAGE"

```
### How upgrade works later
Deploy a new implementation:
```solidity
GopherStakingV2 implV2 = new GopherStakingV2();
```
Then call through proxy:
```solidity
staking.upgradeToAndCall(address(implV2), "");
```
Since proxy storage remains untouched:
  - balances stay
  - rewards stay
  - staking positions stay

only logic changes.

### Role-based Access Control
- Inherit from `AccessControlUpgradeable` contract
- Define Roles
- Init Access Control and 
- Grant Roles

```solidity
contract GopherStaking is Initializable, AccessControlUpgradeable, UUPSUpgradeable {

    // ========================
    // Roles
    // ========================
    bytes32 public constant UPGRADER_ROLE = keccak256("UPGRADER_ROLE"); // can upgrade the contract implementation
    bytes32 public constant REWARD_MANAGER_ROLE = keccak256("REWARD_MANAGER_ROLE"); // can change reward rate and fund the reward pool

    function initialize(address admin, address _token, uint256 _rewardPerBlock) public initializer {
        __AccessControl_init();
        
        // ============================
        // Setup roles
        // ============================
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        _grantRole(UPGRADER_ROLE, admin);
        _grantRole(REWARD_MANAGER_ROLE, admin);
    }
}
```
- `DEFAULT_ADMIN_ROLE` is the super admin role in OpenZeppelin’s RBAC system.
It has special powers:
  - can grant roles
  - can revoke roles
  - is the admin of all roles by default
  - can even manage itself

  Think of it like:
  ```
  Root Admin
    ├── UPGRADER_ROLE
    ├── REWARD_MANAGER_ROLE
    └── any future roles
  ```

  Example:
  ```solidity
  _grantRole(DEFAULT_ADMIN_ROLE, admin);
  ```
  means admin can later do:
  ```solidity
  grantRole(UPGRADER_ROLE, alice);

  revokeRole(REWARD_MANAGER_ROLE, bob);
  ```

### TODOs
- multi-pool support
- rewards range [start, end]
- ETH deposit, withdraw and reward
- Cross-chain staking
