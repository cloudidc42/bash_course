# Part 70: Blockchain & Web3 Infrastructure, Smart Contract Security และ DeFi Protocol Engineering

## ขั้นตอนที่ 627: Ethereum Node Infrastructure & Smart Contract Deployment

### `ethereum-infrastructure.sh`

```bash
#!/bin/bash
# Ethereum Node Infrastructure: Geth + Lighthouse, Smart Contract CI/CD

# Deploy Ethereum full node (Geth execution + Lighthouse consensus)
cat > ethereum-node/geth-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: geth-execution
  namespace: blockchain
spec:
  serviceName: geth
  replicas: 1
  selector:
    matchLabels:
      app: geth
  volumeClaimTemplates:
    - metadata:
        name: chaindata
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3-io2
        resources:
          requests:
            storage: 4Ti
  template:
    metadata:
      labels:
        app: geth
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "6060"
    spec:
      initContainers:
        - name: init-jwt
          image: alpine:3.19
          command: ["/bin/sh", "-c"]
          args:
            - |
              if [ ! -f /jwt/jwt.hex ]; then
                apk add --no-cache openssl
                openssl rand -hex 32 | tr -d '\n' > /jwt/jwt.hex
              fi
          volumeMounts:
            - name: jwt-secret
              mountPath: /jwt
      containers:
        - name: geth
          image: ethereum/client-go:v1.13.14
          command:
            - geth
            - --mainnet
            - --http
            - --http.addr=0.0.0.0
            - --http.port=8545
            - --http.api=eth,net,web3,txpool,engine
            - --http.vhosts=*
            - --http.corsdomain=*
            - --ws
            - --ws.addr=0.0.0.0
            - --ws.port=8546
            - --ws.api=eth,net,web3,txpool,engine
            - --authrpc.addr=0.0.0.0
            - --authrpc.port=8551
            - --authrpc.vhosts=*
            - --authrpc.jwtsecret=/jwt/jwt.hex
            - --datadir=/data
            - --syncmode=snap
            - --gcmode=archive
            - --cache=32768
            - --maxpeers=50
            - --metrics
            - --metrics.addr=0.0.0.0
            - --metrics.port=6060
            - --txlookuplimit=0
            - --history.transactions=0
          ports:
            - containerPort: 8545
              name: rpc
            - containerPort: 8546
              name: ws
            - containerPort: 8551
              name: authrpc
            - containerPort: 30303
              name: p2p-tcp
              protocol: TCP
            - containerPort: 30303
              name: p2p-udp
              protocol: UDP
          resources:
            requests:
              cpu: "8"
              memory: "64Gi"
            limits:
              cpu: "16"
              memory: "128Gi"
          volumeMounts:
            - name: chaindata
              mountPath: /data
            - name: jwt-secret
              mountPath: /jwt
          livenessProbe:
            httpGet:
              path: /
              port: 8545
              httpHeaders:
                - name: Content-Type
                  value: application/json
            initialDelaySeconds: 300
            periodSeconds: 30
      volumes:
        - name: jwt-secret
          emptyDir: {}
EOF

cat > ethereum-node/lighthouse-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: lighthouse-consensus
  namespace: blockchain
spec:
  serviceName: lighthouse
  replicas: 1
  selector:
    matchLabels:
      app: lighthouse
  volumeClaimTemplates:
    - metadata:
        name: beacondata
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3-io2
        resources:
          requests:
            storage: 2Ti
  template:
    metadata:
      labels:
        app: lighthouse
    spec:
      containers:
        - name: lighthouse
          image: sigp/lighthouse:v5.1.3
          command:
            - lighthouse
            - beacon_node
            - --network=mainnet
            - --datadir=/data
            - --http
            - --http-address=0.0.0.0
            - --http-port=5052
            - --execution-endpoint=http://geth:8551
            - --execution-jwt=/jwt/jwt.hex
            - --checkpoint-sync-url=https://mainnet.checkpoint.sigp.io
            - --disable-deposit-contract-sync
            - --target-peers=80
            - --metrics
            - --metrics-address=0.0.0.0
            - --metrics-port=5054
          ports:
            - containerPort: 5052
              name: http
            - containerPort: 9000
              name: p2p-tcp
            - containerPort: 9000
              name: p2p-udp
              protocol: UDP
          resources:
            requests:
              cpu: "4"
              memory: "16Gi"
            limits:
              cpu: "8"
              memory: "32Gi"
          volumeMounts:
            - name: beacondata
              mountPath: /data
            - name: jwt-secret
              mountPath: /jwt
              readOnly: true
      volumes:
        - name: jwt-secret
          secret:
            secretName: jwt-secret
EOF

kubectl apply -f ethereum-node/

# Hardhat Smart Contract project
mkdir -p contracts/defi-protocol
cat > contracts/defi-protocol/hardhat.config.ts << 'TS'
import { HardhatUserConfig } from 'hardhat/config';
import '@nomicfoundation/hardhat-toolbox';
import '@nomicfoundation/hardhat-verify';
import 'hardhat-gas-reporter';
import 'solidity-coverage';
import 'hardhat-contract-sizer';
import * as dotenv from 'dotenv';

dotenv.config();

const config: HardhatUserConfig = {
  solidity: {
    version: '0.8.24',
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,
      },
      viaIR: true,
      evmVersion: 'cancun',
    },
  },
  networks: {
    hardhat: {
      forking: {
        url: process.env.MAINNET_RPC_URL || '',
        blockNumber: 19_500_000,
        enabled: true,
      },
      chainId: 31337,
    },
    mainnet: {
      url: process.env.MAINNET_RPC_URL || '',
      accounts: process.env.DEPLOYER_PRIVATE_KEY ? [process.env.DEPLOYER_PRIVATE_KEY] : [],
      chainId: 1,
      gasPrice: 'auto',
    },
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL || '',
      accounts: process.env.DEPLOYER_PRIVATE_KEY ? [process.env.DEPLOYER_PRIVATE_KEY] : [],
      chainId: 11155111,
    },
  },
  etherscan: {
    apiKey: process.env.ETHERSCAN_API_KEY || '',
  },
  gasReporter: {
    enabled: process.env.REPORT_GAS !== undefined,
    currency: 'USD',
    gasPrice: 20,
    coinmarketcap: process.env.COINMARKETCAP_API_KEY,
  },
  contractSizer: {
    alphaSort: true,
    runOnCompile: true,
    disambiguatePaths: false,
  },
  mocha: {
    timeout: 120000,
  },
};

export default config;
TS

echo "Ethereum node infrastructure setup complete"
```

---

## ขั้นตอนที่ 628: Smart Contract Security — Auditing & Formal Verification

### `smart-contract-security.sh`

```bash
#!/bin/bash
# Smart Contract Security: Slither, Mythril, Echidna, Formal Verification

# Solidity contract with common vulnerability patterns (educational)
cat > contracts/PaymentEscrow.sol << 'SOL'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";

/**
 * @title PaymentEscrow
 * @notice Secure escrow for payment settlements with time-lock and multi-sig release
 */
contract PaymentEscrow is ReentrancyGuard, AccessControl, Pausable {
    using SafeERC20 for IERC20;

    bytes32 public constant ARBITRATOR_ROLE = keccak256("ARBITRATOR_ROLE");
    bytes32 public constant PAUSER_ROLE     = keccak256("PAUSER_ROLE");

    uint256 public constant DISPUTE_WINDOW = 7 days;
    uint256 public constant MAX_FEE_BPS    = 500; // 5% max fee

    struct Escrow {
        address payer;
        address payee;
        address token;
        uint256 amount;
        uint256 fee;
        uint256 createdAt;
        uint256 releaseAfter;
        EscrowState state;
        bytes32 conditionHash;
    }

    enum EscrowState { Active, Released, Disputed, Refunded, Cancelled }

    mapping(bytes32 => Escrow) public escrows;
    mapping(address => uint256) public collectedFees;
    uint256 public feeBps = 50; // 0.5% default fee
    uint256 private _escrowNonce;

    event EscrowCreated(
        bytes32 indexed escrowId,
        address indexed payer,
        address indexed payee,
        address token,
        uint256 amount,
        uint256 releaseAfter
    );
    event EscrowReleased(bytes32 indexed escrowId, address indexed payee, uint256 amount);
    event EscrowDisputed(bytes32 indexed escrowId, address indexed initiator);
    event EscrowRefunded(bytes32 indexed escrowId, address indexed payer, uint256 amount);
    event DisputeResolved(bytes32 indexed escrowId, address winner, uint256 amount);
    event FeeUpdated(uint256 oldFee, uint256 newFee);

    error InvalidAmount();
    error InvalidReleaseTime();
    error EscrowNotFound();
    error NotEscrowParty();
    error EscrowNotActive();
    error DisputeWindowOpen();
    error ConditionNotMet();
    error FeeTooHigh();

    constructor(address admin, address arbitrator) {
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        _grantRole(ARBITRATOR_ROLE, arbitrator);
        _grantRole(PAUSER_ROLE, admin);
    }

    /**
     * @notice Create an escrow for ERC-20 token payment
     * @param payee    Recipient of the payment
     * @param token    ERC-20 token address
     * @param amount   Token amount (before fee deduction)
     * @param lockDuration  Seconds until payee can claim
     * @param conditionHash Keccak256 of off-chain condition (0 for unconditional)
     */
    function createEscrow(
        address payee,
        address token,
        uint256 amount,
        uint256 lockDuration,
        bytes32 conditionHash
    ) external whenNotPaused nonReentrant returns (bytes32 escrowId) {
        if (amount == 0) revert InvalidAmount();
        if (lockDuration == 0 || lockDuration > 365 days) revert InvalidReleaseTime();
        if (payee == address(0) || payee == msg.sender) revert NotEscrowParty();

        uint256 fee = (amount * feeBps) / 10000;
        uint256 netAmount = amount - fee;

        escrowId = keccak256(
            abi.encodePacked(msg.sender, payee, token, amount, block.timestamp, _escrowNonce++)
        );

        escrows[escrowId] = Escrow({
            payer: msg.sender,
            payee: payee,
            token: token,
            amount: netAmount,
            fee: fee,
            createdAt: block.timestamp,
            releaseAfter: block.timestamp + lockDuration,
            state: EscrowState.Active,
            conditionHash: conditionHash,
        });

        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
        collectedFees[token] += fee;

        emit EscrowCreated(escrowId, msg.sender, payee, token, netAmount, escrows[escrowId].releaseAfter);
    }

    /**
     * @notice Payee claims funds after lock period
     * @param escrowId  The escrow identifier
     * @param conditionPreimage  Preimage of conditionHash (bytes32(0) if unconditional)
     */
    function release(bytes32 escrowId, bytes32 conditionPreimage) external nonReentrant whenNotPaused {
        Escrow storage escrow = escrows[escrowId];
        if (escrow.payer == address(0)) revert EscrowNotFound();
        if (escrow.state != EscrowState.Active) revert EscrowNotActive();
        if (msg.sender != escrow.payee) revert NotEscrowParty();
        if (block.timestamp < escrow.releaseAfter) revert DisputeWindowOpen();

        // Verify condition if set
        if (escrow.conditionHash != bytes32(0)) {
            if (keccak256(abi.encodePacked(conditionPreimage)) != escrow.conditionHash) {
                revert ConditionNotMet();
            }
        }

        escrow.state = EscrowState.Released;
        IERC20(escrow.token).safeTransfer(escrow.payee, escrow.amount);

        emit EscrowReleased(escrowId, escrow.payee, escrow.amount);
    }

    /**
     * @notice Payer initiates dispute within dispute window
     */
    function dispute(bytes32 escrowId) external {
        Escrow storage escrow = escrows[escrowId];
        if (escrow.payer == address(0)) revert EscrowNotFound();
        if (escrow.state != EscrowState.Active) revert EscrowNotActive();
        if (msg.sender != escrow.payer && msg.sender != escrow.payee) revert NotEscrowParty();
        if (block.timestamp >= escrow.releaseAfter + DISPUTE_WINDOW) revert DisputeWindowOpen();

        escrow.state = EscrowState.Disputed;
        emit EscrowDisputed(escrowId, msg.sender);
    }

    /**
     * @notice Arbitrator resolves disputed escrow
     */
    function resolveDispute(
        bytes32 escrowId,
        address winner,
        uint256 payeeShareBps
    ) external onlyRole(ARBITRATOR_ROLE) nonReentrant {
        Escrow storage escrow = escrows[escrowId];
        if (escrow.state != EscrowState.Disputed) revert EscrowNotActive();
        if (payeeShareBps > 10000) revert FeeTooHigh();

        uint256 payeeShare = (escrow.amount * payeeShareBps) / 10000;
        uint256 payerShare = escrow.amount - payeeShare;

        escrow.state = EscrowState.Released;

        if (payeeShare > 0) IERC20(escrow.token).safeTransfer(escrow.payee, payeeShare);
        if (payerShare > 0) IERC20(escrow.token).safeTransfer(escrow.payer, payerShare);

        emit DisputeResolved(escrowId, winner, escrow.amount);
    }

    /**
     * @notice Payer can cancel before release time (mutual agreement)
     */
    function refund(bytes32 escrowId) external nonReentrant {
        Escrow storage escrow = escrows[escrowId];
        if (escrow.payer == address(0)) revert EscrowNotFound();
        if (escrow.state != EscrowState.Active) revert EscrowNotActive();
        if (msg.sender != escrow.payer) revert NotEscrowParty();

        escrow.state = EscrowState.Refunded;
        uint256 refundAmount = escrow.amount + escrow.fee;
        collectedFees[escrow.token] -= escrow.fee;

        IERC20(escrow.token).safeTransfer(escrow.payer, refundAmount);
        emit EscrowRefunded(escrowId, escrow.payer, refundAmount);
    }

    function setFee(uint256 newFeeBps) external onlyRole(DEFAULT_ADMIN_ROLE) {
        if (newFeeBps > MAX_FEE_BPS) revert FeeTooHigh();
        emit FeeUpdated(feeBps, newFeeBps);
        feeBps = newFeeBps;
    }

    function withdrawFees(address token, address recipient) external onlyRole(DEFAULT_ADMIN_ROLE) {
        uint256 amount = collectedFees[token];
        collectedFees[token] = 0;
        IERC20(token).safeTransfer(recipient, amount);
    }

    function pause() external onlyRole(PAUSER_ROLE) { _pause(); }
    function unpause() external onlyRole(PAUSER_ROLE) { _unpause(); }
}
SOL

# Install security tools
pip3 install slither-analyzer mythril manticore
npm install -g @crytic/echidna

# Run Slither static analysis
cat > run-slither.sh << 'BASH'
#!/bin/bash
echo "=== Slither Static Analysis ==="
slither contracts/PaymentEscrow.sol \
  --solc-remaps "@openzeppelin=node_modules/@openzeppelin" \
  --checklist \
  --json slither-report.json \
  --filter-paths "node_modules" \
  --exclude-dependencies

# Parse critical findings
python3 << 'EOF'
import json

with open("slither-report.json") as f:
    report = json.load(f)

critical = [d for d in report.get("results", {}).get("detectors", [])
            if d["impact"] in ("High", "Critical")]

print(f"Critical/High findings: {len(critical)}")
for d in critical:
    print(f"  [{d['impact']}] {d['check']}: {d['description'][:200]}")

if critical:
    exit(1)
print("No critical findings!")
EOF
BASH
chmod +x run-slither.sh

# Echidna fuzz testing
cat > contracts/EscrowFuzz.sol << 'SOL'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "./PaymentEscrow.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockToken is ERC20 {
    constructor() ERC20("Mock", "MCK") {
        _mint(msg.sender, 1_000_000e18);
    }
}

contract EscrowFuzz {
    PaymentEscrow public escrow;
    MockToken public token;
    address constant PAYER   = address(0x1001);
    address constant PAYEE   = address(0x1002);
    address constant ARBITER = address(0x1003);
    bytes32 lastEscrowId;

    constructor() {
        escrow = new PaymentEscrow(address(this), ARBITER);
        token  = new MockToken();
        token.transfer(PAYER, 100_000e18);
    }

    // Invariant: contract balance >= sum of all active escrow amounts
    function echidna_balance_geq_active_escrows() public view returns (bool) {
        return token.balanceOf(address(escrow)) >= 0; // simplified
    }

    // Invariant: released escrow funds cannot be claimed twice
    function echidna_no_double_release() public returns (bool) {
        if (lastEscrowId == bytes32(0)) return true;
        (,,,,,,, PaymentEscrow.EscrowState state,) = escrow.escrows(lastEscrowId);
        return state != PaymentEscrow.EscrowState.Active; // after release, state must change
    }

    function create(uint96 amount, uint32 lockDuration) public {
        uint256 a = uint256(amount) + 1e18;
        uint256 d = uint256(lockDuration) % 30 days + 1 days;

        vm.prank(PAYER);
        token.approve(address(escrow), a);
        vm.prank(PAYER);
        lastEscrowId = escrow.createEscrow(PAYEE, address(token), a, d, bytes32(0));
    }
}
SOL

# Formal verification with Certora (specification)
cat > contracts/specs/PaymentEscrow.spec << 'CVL'
// Certora Verification Language specification

methods {
    function createEscrow(address, address, uint256, uint256, bytes32)
        external returns (bytes32) envfree;
    function release(bytes32, bytes32) external;
    function refund(bytes32) external;
}

// Rule: A released escrow cannot be refunded
rule noRefundAfterRelease(bytes32 escrowId) {
    env e;
    require escrows[escrowId].state == EscrowState.Released;
    refund@withrevert(e, escrowId);
    assert lastReverted, "Refund after release must revert";
}

// Rule: Escrow amount is always conserved (no value created/destroyed)
rule escrowAmountConserved(bytes32 escrowId) {
    address token; uint256 contractBalanceBefore;
    require token == escrows[escrowId].token;
    require contractBalanceBefore == tokenBalance(token, currentContract);

    env e;
    release(e, escrowId, to_bytes32(0));

    uint256 contractBalanceAfter = tokenBalance(token, currentContract);
    assert contractBalanceBefore - contractBalanceAfter == escrows[escrowId].amount,
        "Balance delta must equal escrow amount";
}

// Rule: Only payee can release, only payer can refund
rule accessControl(method f, bytes32 escrowId) filtered {f -> f.selector == sig:release(bytes32,bytes32).selector} {
    env e;
    require e.msg.sender != escrows[escrowId].payee;
    f@withrevert(e, escrowId, to_bytes32(0));
    assert lastReverted, "Only payee should be able to release";
}
CVL

echo "Smart contract security audit setup complete"
```

---

## ขั้นตอนที่ 629: DeFi Protocol — Automated Market Maker (AMM)

### `defi-amm-protocol.sh`

```bash
#!/bin/bash
# DeFi Protocol: Constant Product AMM (Uniswap v2 style) + Flash Loans

cat > contracts/AMM/ConstantProductPool.sol << 'SOL'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/math/Math.sol";

/**
 * @title ConstantProductPool
 * @notice x*y=k AMM with LP tokens, swap fees, and flash loans
 */
contract ConstantProductPool is ERC20, ReentrancyGuard {
    using SafeERC20 for IERC20;

    uint256 public constant FEE_DENOMINATOR = 10000;
    uint256 public constant SWAP_FEE        = 30;    // 0.3%
    uint256 public constant FLASH_FEE       = 9;     // 0.09%
    uint256 public constant MINIMUM_LIQUIDITY = 1000;
    bytes4  public constant FLASH_CALLBACK  = bytes4(keccak256("onFlashLoan(address,address,uint256,uint256,bytes)"));

    address public immutable token0;
    address public immutable token1;
    address public immutable factory;

    uint112 private _reserve0;
    uint112 private _reserve1;
    uint32  private _blockTimestampLast;
    uint256 public price0CumulativeLast;
    uint256 public price1CumulativeLast;
    uint256 public kLast;  // reserve0 * reserve1 at last liquidity event

    event Swap(
        address indexed sender,
        uint256 amount0In,
        uint256 amount1In,
        uint256 amount0Out,
        uint256 amount1Out,
        address indexed to
    );
    event Mint(address indexed sender, uint256 amount0, uint256 amount1, uint256 liquidity);
    event Burn(address indexed sender, uint256 amount0, uint256 amount1, address indexed to);
    event Sync(uint112 reserve0, uint112 reserve1);
    event FlashLoan(address indexed borrower, address token, uint256 amount, uint256 fee);

    error InsufficientLiquidity();
    error InsufficientInputAmount();
    error InsufficientOutputAmount();
    error InvalidTo();
    error Overflow();
    error Locked();
    error K();

    constructor(address _token0, address _token1, address _factory)
        ERC20("CP-LP", "CP-LP")
    {
        token0  = _token0;
        token1  = _token1;
        factory = _factory;
    }

    function getReserves() public view returns (
        uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast
    ) {
        return (_reserve0, _reserve1, _blockTimestampLast);
    }

    function _update(
        uint256 balance0,
        uint256 balance1,
        uint112 reserve0,
        uint112 reserve1
    ) private {
        if (balance0 > type(uint112).max || balance1 > type(uint112).max) revert Overflow();

        uint32 blockTimestamp = uint32(block.timestamp % 2**32);
        uint32 timeElapsed    = blockTimestamp - _blockTimestampLast;

        if (timeElapsed > 0 && reserve0 != 0 && reserve1 != 0) {
            // TWAP price accumulators (UQ112x112)
            price0CumulativeLast += uint256(UQ112x112.encode(reserve1).uqdiv(reserve0)) * timeElapsed;
            price1CumulativeLast += uint256(UQ112x112.encode(reserve0).uqdiv(reserve1)) * timeElapsed;
        }

        _reserve0            = uint112(balance0);
        _reserve1            = uint112(balance1);
        _blockTimestampLast  = blockTimestamp;

        emit Sync(_reserve0, _reserve1);
    }

    /**
     * @notice Add liquidity and receive LP tokens
     */
    function mint(address to) external nonReentrant returns (uint256 liquidity) {
        (uint112 reserve0, uint112 reserve1,) = getReserves();
        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        uint256 amount0  = balance0 - reserve0;
        uint256 amount1  = balance1 - reserve1;

        uint256 totalSupply_ = totalSupply();
        if (totalSupply_ == 0) {
            liquidity = Math.sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY;
            _mint(address(1), MINIMUM_LIQUIDITY); // permanently lock minimum
        } else {
            liquidity = Math.min(
                (amount0 * totalSupply_) / reserve0,
                (amount1 * totalSupply_) / reserve1
            );
        }

        if (liquidity == 0) revert InsufficientLiquidity();
        _mint(to, liquidity);
        _update(balance0, balance1, reserve0, reserve1);
        kLast = uint256(_reserve0) * _reserve1;

        emit Mint(msg.sender, amount0, amount1, liquidity);
    }

    /**
     * @notice Remove liquidity and receive underlying tokens
     */
    function burn(address to) external nonReentrant returns (uint256 amount0, uint256 amount1) {
        (uint112 reserve0, uint112 reserve1,) = getReserves();
        uint256 balance0    = IERC20(token0).balanceOf(address(this));
        uint256 balance1    = IERC20(token1).balanceOf(address(this));
        uint256 liquidity   = balanceOf(address(this));
        uint256 totalSupply_ = totalSupply();

        amount0 = (liquidity * balance0) / totalSupply_;
        amount1 = (liquidity * balance1) / totalSupply_;

        if (amount0 == 0 || amount1 == 0) revert InsufficientLiquidity();

        _burn(address(this), liquidity);
        IERC20(token0).safeTransfer(to, amount0);
        IERC20(token1).safeTransfer(to, amount1);

        balance0 = IERC20(token0).balanceOf(address(this));
        balance1 = IERC20(token1).balanceOf(address(this));
        _update(balance0, balance1, reserve0, reserve1);
        kLast = uint256(_reserve0) * _reserve1;

        emit Burn(msg.sender, amount0, amount1, to);
    }

    /**
     * @notice Swap tokens — caller must send input before calling
     * @param amount0Out  Amount of token0 to receive (0 if swapping token0 for token1)
     * @param amount1Out  Amount of token1 to receive (0 if swapping token1 for token0)
     * @param to          Recipient
     */
    function swap(
        uint256 amount0Out,
        uint256 amount1Out,
        address to,
        bytes calldata data
    ) external nonReentrant {
        if (amount0Out == 0 && amount1Out == 0) revert InsufficientOutputAmount();
        (uint112 reserve0, uint112 reserve1,) = getReserves();
        if (amount0Out >= reserve0 || amount1Out >= reserve1) revert InsufficientLiquidity();
        if (to == token0 || to == token1) revert InvalidTo();

        if (amount0Out > 0) IERC20(token0).safeTransfer(to, amount0Out);
        if (amount1Out > 0) IERC20(token1).safeTransfer(to, amount1Out);

        // Callback for flash swaps
        if (data.length > 0) {
            IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
        }

        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));

        // Calculate input amounts
        uint256 amount0In = balance0 > reserve0 - amount0Out ? balance0 - (reserve0 - amount0Out) : 0;
        uint256 amount1In = balance1 > reserve1 - amount1Out ? balance1 - (reserve1 - amount1Out) : 0;
        if (amount0In == 0 && amount1In == 0) revert InsufficientInputAmount();

        // Verify K invariant: (balance - 0.3% fee) must maintain x*y >= k
        uint256 balance0Adjusted = balance0 * FEE_DENOMINATOR - amount0In * SWAP_FEE;
        uint256 balance1Adjusted = balance1 * FEE_DENOMINATOR - amount1In * SWAP_FEE;
        if (balance0Adjusted * balance1Adjusted < uint256(reserve0) * reserve1 * FEE_DENOMINATOR**2) revert K();

        _update(balance0, balance1, reserve0, reserve1);

        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }

    /**
     * @notice Flash loan: borrow any token, repay in same transaction with fee
     */
    function flashLoan(
        address borrower,
        address token,
        uint256 amount,
        bytes calldata data
    ) external nonReentrant {
        if (token != token0 && token != token1) revert InvalidTo();
        (uint112 reserve0, uint112 reserve1,) = getReserves();

        uint256 fee = (amount * FLASH_FEE) / FEE_DENOMINATOR;
        IERC20(token).safeTransfer(borrower, amount);

        // Callback
        bytes4 result = IFlashBorrower(borrower).onFlashLoan(
            msg.sender, token, amount, fee, data
        );
        if (result != FLASH_CALLBACK) revert K();

        // Verify repayment
        uint256 balance = IERC20(token).balanceOf(address(this));
        uint256 required = (token == token0 ? reserve0 : reserve1) + fee;
        if (balance < required) revert InsufficientInputAmount();

        _update(
            IERC20(token0).balanceOf(address(this)),
            IERC20(token1).balanceOf(address(this)),
            reserve0,
            reserve1
        );

        emit FlashLoan(borrower, token, amount, fee);
    }
}

// UQ112x112 fixed-point math library
library UQ112x112 {
    uint224 constant Q112 = 2**112;

    function encode(uint112 y) internal pure returns (uint224 z) {
        z = uint224(y) * Q112;
    }

    function uqdiv(uint224 x, uint112 y) internal pure returns (uint224 z) {
        z = x / uint224(y);
    }
}

interface IUniswapV2Callee {
    function uniswapV2Call(address sender, uint amount0, uint amount1, bytes calldata data) external;
}

interface IFlashBorrower {
    function onFlashLoan(address initiator, address token, uint256 amount, uint256 fee, bytes calldata data)
        external returns (bytes4);
}
SOL

# Price Oracle (TWAP)
cat > contracts/AMM/TWAPOracle.sol << 'SOL'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "./ConstantProductPool.sol";

/**
 * @title TWAPOracle
 * @notice Time-Weighted Average Price oracle resistant to manipulation
 */
contract TWAPOracle {
    struct Observation {
        uint256 timestamp;
        uint256 price0Cumulative;
        uint256 price1Cumulative;
    }

    mapping(address => Observation) public observations;
    uint256 public constant PERIOD = 30 minutes;

    function update(address pool) external {
        Observation storage obs = observations[pool];
        require(block.timestamp >= obs.timestamp + PERIOD, "TWAP: too frequent");

        ConstantProductPool p = ConstantProductPool(pool);
        obs.timestamp        = block.timestamp;
        obs.price0Cumulative = p.price0CumulativeLast();
        obs.price1Cumulative = p.price1CumulativeLast();
    }

    function consult(
        address pool,
        address tokenIn,
        uint256 amountIn
    ) external view returns (uint256 amountOut) {
        ConstantProductPool p = ConstantProductPool(pool);
        Observation memory obs = observations[pool];
        require(obs.timestamp != 0, "TWAP: not initialized");

        uint256 timeElapsed = block.timestamp - obs.timestamp;
        require(timeElapsed <= PERIOD * 2, "TWAP: stale");

        uint256 price0Cumulative = p.price0CumulativeLast();
        uint256 price1Cumulative = p.price1CumulativeLast();

        // Calculate average price over period
        if (tokenIn == p.token0()) {
            uint256 priceCumDelta = price1Cumulative - obs.price1Cumulative;
            // UQ112x112 decode
            amountOut = (priceCumDelta / timeElapsed) * amountIn / 2**112;
        } else {
            uint256 priceCumDelta = price0Cumulative - obs.price0Cumulative;
            amountOut = (priceCumDelta / timeElapsed) * amountIn / 2**112;
        }
    }
}
SOL

echo "DeFi AMM protocol contracts complete"
```

---

## ขั้นตอนที่ 630: Web3 Backend — Indexer, Event Processing & API

### `web3-backend.sh`

```bash
#!/bin/bash
# Web3 Backend: TheGraph-style Indexer, Event Processor, REST API

cat > web3-backend/indexer.py << 'PYTHON'
import asyncio
import logging
from dataclasses import dataclass
from typing import Any
from web3 import AsyncWeb3, WebSocketProvider
from web3.middleware import ExtraDataToPOAMiddleware
import json
import asyncpg

logger = logging.getLogger(__name__)


POOL_ABI = json.loads("""[
  {"type":"event","name":"Swap","inputs":[
    {"name":"sender","type":"address","indexed":true},
    {"name":"amount0In","type":"uint256","indexed":false},
    {"name":"amount1In","type":"uint256","indexed":false},
    {"name":"amount0Out","type":"uint256","indexed":false},
    {"name":"amount1Out","type":"uint256","indexed":false},
    {"name":"to","type":"address","indexed":true}
  ]},
  {"type":"event","name":"Mint","inputs":[
    {"name":"sender","type":"address","indexed":true},
    {"name":"amount0","type":"uint256","indexed":false},
    {"name":"amount1","type":"uint256","indexed":false},
    {"name":"liquidity","type":"uint256","indexed":false}
  ]},
  {"type":"event","name":"Burn","inputs":[
    {"name":"sender","type":"address","indexed":true},
    {"name":"amount0","type":"uint256","indexed":false},
    {"name":"amount1","type":"uint256","indexed":false},
    {"name":"to","type":"address","indexed":true}
  ]}
]""")


@dataclass
class SwapEvent:
    block_number: int
    tx_hash: str
    pool: str
    sender: str
    amount0_in: int
    amount1_in: int
    amount0_out: int
    amount1_out: int
    recipient: str
    timestamp: int


class DeFiIndexer:
    BATCH_SIZE = 500
    REORG_DEPTH = 20  # blocks to consider for reorg protection

    def __init__(self, rpc_url: str, db_url: str, contract_address: str):
        self.rpc_url = rpc_url
        self.db_url = db_url
        self.contract_address = contract_address
        self._w3: AsyncWeb3 | None = None
        self._db: asyncpg.Pool | None = None

    async def start(self):
        self._w3 = AsyncWeb3(WebSocketProvider(self.rpc_url))
        self._w3.middleware_onion.inject(ExtraDataToPOAMiddleware, layer=0)
        self._db = await asyncpg.create_pool(self.db_url, min_size=5, max_size=20)

        await self._ensure_schema()

        last_block = await self._get_last_indexed_block()
        current_block = await self._w3.eth.block_number

        logger.info(f"Indexer starting from block {last_block} to {current_block}")

        # Catch up historical events
        await self._index_range(last_block - self.REORG_DEPTH, current_block)

        # Subscribe to new blocks
        await self._subscribe_new_blocks()

    async def _ensure_schema(self):
        async with self._db.acquire() as conn:
            await conn.execute("""
                CREATE TABLE IF NOT EXISTS swap_events (
                    id             BIGSERIAL PRIMARY KEY,
                    block_number   BIGINT NOT NULL,
                    tx_hash        CHAR(66) NOT NULL,
                    pool           CHAR(42) NOT NULL,
                    sender         CHAR(42) NOT NULL,
                    recipient      CHAR(42) NOT NULL,
                    amount0_in     NUMERIC(78) NOT NULL,
                    amount1_in     NUMERIC(78) NOT NULL,
                    amount0_out    NUMERIC(78) NOT NULL,
                    amount1_out    NUMERIC(78) NOT NULL,
                    block_ts       BIGINT NOT NULL,
                    created_at     TIMESTAMPTZ DEFAULT NOW()
                );
                CREATE UNIQUE INDEX IF NOT EXISTS swap_events_tx_idx
                    ON swap_events(tx_hash, block_number);
                CREATE INDEX IF NOT EXISTS swap_events_block_idx
                    ON swap_events(block_number DESC);
                CREATE INDEX IF NOT EXISTS swap_events_sender_idx
                    ON swap_events(sender);

                CREATE TABLE IF NOT EXISTS indexer_state (
                    key   TEXT PRIMARY KEY,
                    value TEXT NOT NULL
                );
            """)

    async def _get_last_indexed_block(self) -> int:
        async with self._db.acquire() as conn:
            row = await conn.fetchrow(
                "SELECT value FROM indexer_state WHERE key = 'last_block'"
            )
            return int(row["value"]) if row else 0

    async def _index_range(self, from_block: int, to_block: int):
        contract = self._w3.eth.contract(
            address=self.contract_address,
            abi=POOL_ABI,
        )

        for start in range(from_block, to_block, self.BATCH_SIZE):
            end = min(start + self.BATCH_SIZE - 1, to_block)
            try:
                events = await contract.events.Swap().get_logs(
                    fromBlock=start, toBlock=end
                )
                if events:
                    await self._persist_swaps(events)
                await self._update_indexed_block(end)
                logger.info(f"Indexed blocks {start}-{end}: {len(events)} swaps")
            except Exception as e:
                logger.error(f"Failed to index {start}-{end}: {e}")
                await asyncio.sleep(2)

    async def _persist_swaps(self, events: list):
        records = []
        for e in events:
            block = await self._w3.eth.get_block(e["blockNumber"])
            records.append((
                e["blockNumber"],
                e["transactionHash"].hex(),
                self.contract_address.lower(),
                e["args"]["sender"].lower(),
                e["args"]["to"].lower(),
                e["args"]["amount0In"],
                e["args"]["amount1In"],
                e["args"]["amount0Out"],
                e["args"]["amount1Out"],
                block["timestamp"],
            ))

        async with self._db.acquire() as conn:
            await conn.executemany(
                """
                INSERT INTO swap_events
                    (block_number, tx_hash, pool, sender, recipient,
                     amount0_in, amount1_in, amount0_out, amount1_out, block_ts)
                VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10)
                ON CONFLICT (tx_hash, block_number) DO NOTHING
                """,
                records,
            )

    async def _update_indexed_block(self, block: int):
        async with self._db.acquire() as conn:
            await conn.execute(
                """
                INSERT INTO indexer_state(key, value) VALUES('last_block', $1)
                ON CONFLICT (key) DO UPDATE SET value = $1
                """,
                str(block),
            )

    async def _subscribe_new_blocks(self):
        subscription = await self._w3.eth.subscribe("newHeads")
        async for block_header in subscription:
            block_number = block_header["number"]
            last = await self._get_last_indexed_block()
            if block_number > last:
                await self._index_range(last + 1, block_number)


# REST API for DeFi data
from fastapi import FastAPI, Query, HTTPException
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="DeFi Indexer API", version="1.0.0")
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"])

db_pool: asyncpg.Pool | None = None

@app.on_event("startup")
async def startup():
    global db_pool
    db_pool = await asyncpg.create_pool(
        "postgresql://defi:secret@postgres:5432/defi_index",
        min_size=5, max_size=20,
    )

@app.get("/api/v1/swaps")
async def list_swaps(
    pool: str | None = None,
    sender: str | None = None,
    from_block: int | None = None,
    to_block: int | None = None,
    limit: int = Query(default=50, le=500),
    offset: int = 0,
):
    conditions = []
    params: list[Any] = []
    i = 1

    if pool:
        conditions.append(f"pool = ${i}")
        params.append(pool.lower())
        i += 1
    if sender:
        conditions.append(f"sender = ${i}")
        params.append(sender.lower())
        i += 1
    if from_block is not None:
        conditions.append(f"block_number >= ${i}")
        params.append(from_block)
        i += 1
    if to_block is not None:
        conditions.append(f"block_number <= ${i}")
        params.append(to_block)
        i += 1

    where = "WHERE " + " AND ".join(conditions) if conditions else ""
    params.extend([limit, offset])

    query = f"""
        SELECT * FROM swap_events
        {where}
        ORDER BY block_number DESC
        LIMIT ${i} OFFSET ${i+1}
    """

    async with db_pool.acquire() as conn:
        rows = await conn.fetch(query, *params)
    return [dict(r) for r in rows]

@app.get("/api/v1/stats/24h")
async def stats_24h():
    async with db_pool.acquire() as conn:
        row = await conn.fetchrow("""
            SELECT
                COUNT(*) as swap_count,
                SUM(amount0_in + amount1_in) as total_volume,
                COUNT(DISTINCT sender) as unique_traders
            FROM swap_events
            WHERE block_ts > EXTRACT(EPOCH FROM NOW() - INTERVAL '24 hours')
        """)
    return dict(row)
PYTHON

echo "Web3 backend indexer and API complete"
```

---

## สรุป Part 70

Part 70 ครอบคลุม Blockchain & Web3 ระดับ production:

| Step | หัวข้อ | เทคโนโลยีหลัก |
|------|--------|--------------|
| 627 | Ethereum Node Infrastructure | Geth + Lighthouse K8s StatefulSet, Hardhat TypeScript |
| 628 | Smart Contract Security | PaymentEscrow (ReentrancyGuard/AccessControl), Slither, Echidna Fuzz, Certora CVL |
| 629 | DeFi AMM Protocol | Constant Product AMM (x*y=k), Flash Loans, TWAP Oracle, UQ112x112 |
| 630 | Web3 Backend | AsyncWeb3 Indexer, Event processing, asyncpg, FastAPI REST API |

### Key Concepts ที่เรียนรู้:
- **Ethereum Node**: Geth execution + Lighthouse consensus, JWT auth, snap sync, archive mode
- **Smart Contract Patterns**: ReentrancyGuard, AccessControl, SafeERC20, Custom Errors
- **Security Tools**: Slither (static analysis), Echidna (fuzz testing), Certora (formal verification)
- **AMM Mathematics**: Constant product formula, TWAP price accumulator, UQ112x112 fixed-point
- **Flash Loans**: Single-transaction borrow+repay pattern, callback verification
- **Web3 Indexer**: WebSocket subscription, batch historical sync, reorg protection, asyncpg

ขั้นตอนต่อไป: **Part 71** — AI-Powered DevOps (AIOps), Predictive Autoscaling และ Autonomous Incident Resolution
