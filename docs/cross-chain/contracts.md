
# Contracts

## CrossChainPaymaster and L1StakeManager interfaces.

This page documents the Solidity interfaces for the two core contracts in the WeissChannels protocol.

---

## 🔧 CrossChainPaymaster

The `CrossChainPaymaster` MUST be deployed on every supported chain. It serves as the central hub for WeissChannel operations, combining four distinct roles:

| Role | Description |
|------|-------------|
| **Origin Swap Manager** | Locks user funds, tracks voucher requests, releases funds to XLPs |
| **Destination Swap Manager** | Validates vouchers during UserOp validation, releases XLP funds to users |
| **XLP Registry** | Maintains the list of registered XLPs, synchronized with L1 stake status |
| **Dispute Manager** | Initiates disputes when XLPs misbehave, coordinates with L1 for slashing |

---

### Origin Chain Functions

```solidity
interface IOriginSwapManager {
    /// @notice Lock user funds and create a voucher request
    function lockUserDeposit(
        AtomicSwapVoucherRequest calldata voucherRequest
    ) external payable;

    /// @notice XLP issues vouchers for pending requests
    function issueVouchers(
        VoucherWithRequest[] calldata vouchers
    ) external;

    /// @notice XLP issues alternative vouchers during a dispute
    function issueAltVouchers(
        VoucherWithRequest[] calldata vouchers
    ) external;

    /// @notice XLP overrides a previously issued voucher
    function overrideVoucher(
        AtomicSwapVoucherRequest calldata voucherRequest,
        AtomicSwapVoucher calldata voucherOverride
    ) external payable;

    /// @notice XLP withdraws user's locked funds after voucher execution
    function withdrawFromUserDeposit(
        AtomicSwapVoucherRequest[] calldata voucherRequests
    ) external;

    /// @notice User cancels a request and retrieves locked funds
    function cancelVoucherRequest(
        AtomicSwapVoucherRequest calldata voucherRequest
    ) external;

    /// @notice Claim fee for voucher that was issued but not consumed
    function claimUnspentVoucherFee(
        AtomicSwapVoucherRequest calldata voucherRequest
    ) external;

    /// @notice Withdraw the unspent voucher fee after dispute window
    function withdrawUnspentVoucherFee(
        AtomicSwapVoucherRequest calldata voucherRequest
    ) external;
}
```

---

### Origin Chain Events

```solidity
event VoucherRequestCreated(
    bytes32 indexed requestId,
    address indexed sender,
    AtomicSwapVoucherRequest voucherRequest
);

event VoucherIssued(
    bytes32 indexed requestId,
    address indexed sender,
    uint256 indexed senderNonce,
    AtomicSwapVoucher voucher
);

event UserDepositWithdrawn(
    bytes32 indexed requestId,
    address indexed sender,
    address indexed voucherIssuer
);

event VoucherRequestCancelled(
    bytes32 indexed requestId,
    address indexed sender
);
```

---

### Destination Chain Functions

```solidity
interface IDestinationSwapManager {
    /// @notice Withdraw funds from a voucher (during UserOp execution)
    function withdrawFromVoucher(
        AtomicSwapVoucherRequest memory voucherRequest,
        AtomicSwapVoucher memory voucher
    ) external;

    /// @notice Get status of an incoming atomic swap
    function getIncomingAtomicSwap(
        bytes32 requestId
    ) external view returns (AtomicSwapMetadataDestination memory);
}
```

---

### XLP Registry Functions

```solidity
interface IL2XlpRegistry {
    /// @notice Called by L1StakeManager when XLP stakes for this chain
    function onL1XlpChainInfoAdded(
        address l1XlpAddress,
        address l2XlpAddress
    ) external;

    /// @notice Called by L1StakeManager when XLP's stake is unlocked
    function onL1XlpStakeUnlocked(address l1XlpAddress) external;

    /// @notice Check if an L1 XLP address is registered
    function isL1XlpRegistered(address l1XlpAddress) external view returns (bool);

    /// @notice Check if an L2 XLP address is registered
    function isL2XlpRegistered(address l2XlpAddress) external view returns (bool);

    /// @notice Get paginated list of registered XLPs
    function getXlps(
        uint256 offset,
        uint256 length
    ) external view returns (XlpEntry[] memory);
}
```

---

### Dispute Functions

```solidity
interface IL2XlpDisputeManager {
    /// @notice Initiate dispute against insolvent XLP
    function disputeInsolventXlp(
        DisputeVoucher[] calldata disputeVouchers,
        VoucherWithRequest[] calldata altVouchers,
        address l2XlpAddressToSlash,
        address payable l1Beneficiary,
        uint256 chunkIndex,
        uint256 numberOfChunks,
        uint256 nonce,
        bytes32 committedRequestIdsHash,
        uint256 committedVoucherCount
    ) external payable;

    /// @notice Dispute a fraudulent voucher override
    function disputeVoucherOverride(
        DisputeVoucher[] calldata disputeVouchers,
        address l2XlpAddressToSlash,
        address payable l1Beneficiary
    ) external payable;

    /// @notice Dispute a fraudulent unspent voucher fee claim
    function disputeXlpUnspentVoucherClaim(
        DisputeVoucher[] calldata disputeVouchers,
        address payable l1Beneficiary
    ) external payable;

    /// @notice Withdraw dispute bonds after resolution
    function withdrawDisputeBonds(bytes32[] calldata requestIds) external;

    /// @notice Called by L1StakeManager after slash is confirmed
    function onXlpSlashedMessage(SlashOutput calldata slashOutput) external;
}
```

---

## 🏛️ L1AtomicSwapStakeManager

The `L1AtomicSwapStakeManager` MUST be deployed on Ethereum Mainnet. It serves as the "Supreme Court" of the WeissChannel system — the ultimate arbiter of disputes.

### Capital Efficiency: O(Networks) Staking

In custodial bridges, relayers must lock collateral proportional to the value moved. WeissChannels operate on a **Performance Bond** model:

- The XLP's L1 stake is a penalty bond, not a liquidity backing
- The stake scales with the number of networks supported: **O(Networks)**, not O(Volume)
- A single 10 ETH stake can secure unlimited volume

---

### Configuration Constants

```solidity
struct Config {
    uint256 claimDelay;            // Delay before slashed funds can be claimed
    uint256 destBeforeOriginMinGap; // Min time gap between dest proof and origin dispute
    uint256 minStakePerChain;      // Minimum stake required per supported chain
    uint256 unstakeDelay;          // Delay before unlocked stake can be withdrawn
    uint256 maxChainsPerXlp;       // Maximum chains an XLP can support
    uint256 l2SlashedGasLimit;     // Gas limit for L2 slash notifications
    uint256 l2StakedGasLimit;      // Gas limit for L2 stake notifications
    address owner;                 // Contract owner
}
```

| Parameter | Recommended Value | Rationale |
|-----------|-------------------|-----------|
| `unstakeDelay` | 8 days | Longer than maximum L2 finality (7 days) |
| `claimDelay` | 1 hour | Window for additional dispute evidence |
| `destBeforeOriginMinGap` | 1 hour | Ensures destination proof precedes origin dispute |

---

### Staking Functions

```solidity
interface IL1AtomicSwapStakeManager {
    /// @notice Register as XLP and stake for specified chains
    function addChainsInfo(
        uint256[] calldata chainIds,
        ChainInfo[] calldata chainsInfo
    ) external payable;

    /// @notice Initiate stake unlock (begins unstake delay)
    function unlockStake(uint256[] calldata chainIds) external;

    /// @notice Withdraw stake after unstake delay has passed
    function withdrawStake(
        address payable withdrawAddress,
        uint256[] calldata chainIds
    ) external;

    /// @notice Get stake info for an XLP
    function getStakeInfo(
        address xlp,
        uint256[] calldata chainIds
    ) external view returns (StakeInfo[] memory info);

    /// @notice Get all chain IDs supported by an XLP
    function getXlpChainIds(address xlp) external view returns (uint256[] memory);
}
```

---

### Dispute Resolution Functions

```solidity
interface IL1AtomicSwapStakeManager {
    /// @notice Pull and process dispute messages from L2 bridges
    function pullMessagesFromBridges(
        IL1Bridge[] calldata bridges,
        bytes[][] calldata bridgeMessagesPerBridge
    ) external;

    /// @notice Receive origin-side dispute report from L2
    function reportOriginDispute(ReportDisputeLeg calldata originLeg) external;

    /// @notice Receive destination-side proof report from L2
    function reportDestinationProof(ReportProofLeg calldata proofLeg) external;

    /// @notice Claim share of slashed stake (for insolvency disputes)
    function claimSlashShare(
        address l1XlpAddress,
        uint256 originationChainId,
        uint256 destinationChainId,
        DisputeType disputeType,
        SlashShareRole role
    ) external;

    /// @notice Claim slashed stake for single-beneficiary disputes
    function claimSlashSingle(
        address l1XlpAddress,
        uint256 originationChainId,
        uint256 destinationChainId,
        DisputeType disputeType
    ) external;
}
```

---

### Key Events

```solidity
/// @notice Emitted when XLP locks stake for chains
event StakeLocked(
    address indexed xlp,
    uint256[] chainIds,
    uint256 stake
);

/// @notice Emitted when stake unlock is initiated
event StakeUnlocked(
    address indexed account,
    uint256[] chainIds,
    uint256 withdrawTime
);

/// @notice Emitted when stake slash is scheduled
event StakeSlashScheduled(
    address indexed l1XlpAddress,
    uint256 originationChainId,
    uint256 destinationChainId,
    DisputeType disputeType,
    bytes32 requestIdsHash,
    uint256 amount,
    uint256 claimableAt
);
```

---

## 📋 Supporting Types

```solidity
enum AtomicSwapStatus {
    NONE,           // Request does not exist
    NEW,            // Request created, awaiting voucher
    VOUCHER_ISSUED, // Voucher has been issued
    CANCELLED,      // Request cancelled by user
    DISPUTE,        // Request is under dispute
    PENALIZED,      // XLP was slashed
    SUCCESSFUL,     // Swap completed successfully
    UNSPENT         // Voucher was not consumed
}

enum DisputeType {
    INSOLVENT_XLP,
    VOUCHER_OVERRIDE,
    UNSPENT_VOUCHER_FEE_CLAIM
}

enum VoucherType {
    STANDARD,
    OVERRIDE,
    ALT,
    ALT_OVERRIDE
}

struct ChainInfo {
    address paymaster;      // CrossChainPaymaster on this chain
    address l1Connector;    // L1 bridge connector
    address l2Connector;    // L2 bridge connector
    address l2XlpAddress;   // XLP's address on this L2
}

struct StakeInfo {
    uint256 stake;          // Current stake amount
    uint256 withdrawTime;   // When withdrawal is available (0 if locked)
}
```

---

## 🔗 Related Pages

- [Architecture](architecture.md) — Protocol lifecycle and data flow
- [Security](security.md) — How dispute resolution protects users
- [ERC-4337](../core-standards/erc-4337.md) — The underlying account abstraction standard

