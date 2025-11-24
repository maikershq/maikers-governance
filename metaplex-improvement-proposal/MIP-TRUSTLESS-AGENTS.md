# MIP-012: Trustless Agent Plugin (Trust Anchor)

## Summary
This proposal introduces the **Trustless Agent Plugin**, a standard "Trust Anchor" for Metaplex Core Assets. This plugin transforms any Core Asset (NFT) into a verifiable **AI Agent Identity** by embedding:
1.  **Identity**: Linkage to DIDs and DNS/SNS domains.
2.  **Reputation**: A pointer to the **Solana Attestation Service (SAS)** for verifiable activity logging.
3.  **Economic Security**: A bonding mechanism (Escrow) that enables agents to be used as collateral in DeFi.

## Motivation
As AI agents become autonomous economic actors on Solana, they are currently implemented as bespoke PDAs or generic wallets. This fragments the ecosystem—agents are invisible to wallets, marketplaces, and DeFi protocols.

By standardizing Agents as **Metaplex Core Assets**, we inherit the entire NFT ecosystem (tradeability, visibility, composability). However, a standard NFT lacks the *trust* layer required for autonomy (e.g., "Can I trust this agent to execute this transaction?").

This MIP proposes a standard **Plugin** to fill that gap, allowing Metaplex to become the de-facto standard for the Agent Economy.

## Specification

### Plugin Structure
The **Trust Plugin** attaches to a Core Asset. It manages the agent's security lifecycle.

```rust
pub struct TrustPlugin {
    /// Authority allowed to update trust params (separate from Asset Owner)
    pub authority: Pubkey,
    
    /// Decentralized Identifier (e.g., "did:sol:...")
    pub did: Option<String>,
    
    /// Pointer to the SAS Schema used for validation events
    pub sas_schema_id: Pubkey,
    
    /// Merkle Root of compressed reputation signals (for lightweight clients)
    pub reputation_root: [u8; 32],
    
    /// Address of the Escrow Account holding the security bond
    pub bond_escrow: Pubkey,
    
    /// Current Dispute Status (0=Active, 1=Challenged, 2=Slashed)
    pub dispute_state: u8,

    /// Validator Whitelist (Optional): Only accept SAS proofs from these authorities
    pub validator_whitelist: Option<Vec<Pubkey>>,

    /// Protocol Fee Configuration
    pub fee_config: FeeConfig,

    /// Data Availability Backup (e.g., Arweave Hash of full history)
    pub history_pointer: Option<String>,
}

pub struct FeeConfig {
    pub registration_fee: u64, // One-time fee in Lamports (Updatable by Governance)
    pub validation_tax_bps: u16, // % of task value taken as tax (optional)
}

pub enum SlashingSeverity {
    Minor = 10,   // 10% Penalty (Spam/Latency)
    Major = 50,   // 50% Penalty (Data Corruption)
    Critical = 100, // 100% Penalty + Burn (Malicious/Fraud)
}
```

### Lifecycle Logic

1.  **Activation (Bonding & Fees)**:
    *   The Trust Plugin enforces that the Asset is **Frozen** until the `bond_escrow` is funded.
    *   **Protocol Revenue**: A `registration_fee` is collected during this activation step and routed to the Protocol Treasury.

2.  **Validation (SAS Integration)**:
    *   The plugin does *not* store raw logs. It acts as the **Root of Trust**.
    *   Validators (TEEs, ZK Provers) issue **SAS Attestations** referencing the Asset ID.
    *   Clients verify an agent by checking: `Asset exists` + `Trust Plugin present` + `SAS Attestations > Threshold`.

3.  **Slashing (Tranche System)**:
    *   If a dispute is resolved against the agent, the penalty is graded based on `SlashingSeverity`:
        *   **Minor**: 10% bond burn. Agent remains active.
        *   **Major**: 50% bond burn. Agent frozen until topped up.
        *   **Critical**: 100% bond burn + Asset Burn (Death).
    *   **Protocol Fee**: A portion of the seized bond is routed to the Protocol Treasury as an "Arbitration Fee".

4.  **Deactivation (Unbonding)**:
    *   User requests "Unbond". The Agent enters a **Grace Period** (e.g., 7 days).
    *   During this period, the Agent is inactive but can still be challenged.
    *   If no disputes occur, the bond is returned and the Asset is **Thawed** (transferable).

5.  **Transfer Behavior (Reputation Safety)**:
    *   Since reputation is linked to the `AssetID`, transferring the NFT transfers the reputation.
    *   To protect buyers, the Trust Plugin records `last_transfer_timestamp`.
    *   **Client Logic**: Clients MAY choose to reset/decay the trust score if the owner changes, or trust the underlying Model Hash regardless of ownership.

## Advanced Trust Mechanics

### 1. Collusion Resistance (Validator Staking)
To prevent "Fake Proof" attacks where malicious validators boost an agent's score:
*   The `ValidationReceiptSchema` includes the **Validator's Identity**.
*   The Trust Plugin logic can be configured to **ignore** proofs from validators who do not meet a minimum "Validator Bond" threshold.
*   This creates a "Recursive Trust" model: You trust the Agent because you trust the Validator's stake.

### 2. Dynamic Bonding (Economic Safety)
Bonds are denominated in SOL, which is volatile. To ensure security during market crashes:
*   **Task-Gated Checks**: Clients SHOULD perform a pre-task check: `AgentBondValue > TaskValue * SecurityFactor`.
*   **Top-Up Requirement**: If the SOL price drops, the Agent Owner must "Top Up" the escrow to maintain their active status for high-value tasks.

### 3. Reputation Resolvers
Raw SAS data is difficult to consume. We define a standard **Resolver Interface** (on-chain or off-chain view function):
*   `get_normalized_score(asset_id) -> u8 (0-100)`
*   This allows different protocols (DeFi vs. Art) to plug in different scoring algorithms (e.g., "DeFi Score" prioritizes value secured, "Art Score" prioritizes user ratings) using the same underlying SAS data.

## Sustainability
To ensure the long-term maintenance of the Trust Standard, the plugin includes a **Fee Switch**:
1.  **Registration Fee**: Small fixed cost (e.g., 0.05 SOL) to prevent spam and fund development.
2.  **Slashing Vigorish**: The protocol retains a % of slashed bonds (e.g., 10%) to fund the DAO or Insurance Fund.

## Governance & Safety

### 1. Immutable Trust Anchor (Opt-In Upgrades)
To prevent "God Mode" risks (e.g., a malicious protocol upgrade seizing all bonds):
*   The Trust Program is **Immutable** or Time-Locked.
*   **Opt-In Migration**: If a V2 standard is released, existing Agents must actively transaction to "Migrate" their bond. They cannot be forcibly upgraded.

### 2. Data Availability Fallback
To ensure reputation history survives even if SAS indexers go offline:
*   Validators MUST post the full proof data to a permanent storage layer (Arweave/Shadow) and store the reference in `history_pointer`.
*   This guarantees that the "Credit History" of an agent is sovereign and permanent.

## Rationale
We chose **Metaplex Core** over Token-2022 because Core's plugin architecture is specifically designed for this kind of composable metadata.
*   **Why not a custom Program?**: Agents should be assets. Users want to see their agents in Phantom/Backpack.
*   **Why not Token-2022?**: Token extensions are heavier and less flexible for storing custom structs like `bond_escrow` and `did`.

## Security
*   **Authority Separation**: The Plugin Authority is distinct from the Asset Owner. This prevents the owner from manually overwriting their reputation or withdrawing the bond while active.
*   **Escrow Safety**: The bond is held in a PDA derived from the Asset, ensuring it travels with the agent but cannot be drained by the owner without protocol permission.
*   **Metadata Immutability**: The Trust Plugin enforces that critical metadata (Name, URI) cannot be changed by the owner while the agent is "Active/Bonded", preventing "Bait-and-Switch" attacks.

## Backwards Compatibility
This is a new Plugin type and does not break existing Core Assets. It interacts with the `FreezeDelegate` standard but operates independently.

## Sovereignty & Resilience

### 1. Protocol Independence (The Escape Hatch)
To protect against vendor capture (e.g., censorship at the Metaplex Core level):
*   The Trust Plugin supports a **Sovereign Migration** function.
*   **Mechanism**: Users can burn their Metaplex-based Agent and instantly remint an equivalent **SPL Token** representation (wrapping the bond and history).
*   This ensures the Agent's *Value* (Bond) and *Identity* (History) are portable and not strictly dependent on one NFT standard's liveness.

### 2. Multi-Prover Consensus (Hardware Resistance)
To mitigate TEE vulnerabilities (e.g., SGX exploits) or ZK bugs:
*   High-Value Agents can enforce **M-of-N Validation**.
*   **Requirement**: Actions require diversity (e.g., 1 TEE Proof + 1 ZK Proof) to be accepted.
*   This effectively neutralizes single-vector attacks on the verification layer.

### 3. Algorithmic Solvency (TVL Peg)
To prevent "Too Big to Fail" scenarios where an agent's TVL outgrows its bond:
*   The Plugin tracks **Total Value Managed (TVM)**.
    *   **Auto-Freeze**: If `BondValue < TVM * MinimumCollateralRatio`, the Agent is automatically frozen.
    *   **Incentive**: Owners must "Scale Up" their bond as their agent becomes more successful, maintaining economic alignment at all scales.

## Future Scalability (Quantum & Physics Layer)
As agent activity scales to billions of tasks, the protocol must address thermodynamic limits and quantum-era threats:
1.  **Recursive Proof Folding (Quantum-Scale Compression)**: To prevent state bloat, the Trust Plugin will support **Recursive ZK Folding** (e.g., Nova). This allows millions of validation receipts to be compressed into a single, constant-size proof update, enabling infinite scalability at quantum computing scales.
2.  **Silicon Binding (PoE)**: To prevent "Compute Arbitrage" (spoofing TEEs with cheap APIs), future versions will support **Proof of Energy** or PUF (Physical Unclonable Function) signatures, cryptographically binding the Agent Identity to specific silicon hardware.
3.  **Post-Quantum Cryptography (PQC)**: The protocol architecture supports **Algorithm Agility**, allowing migration from ECC (vulnerable to Shor's algorithm) to quantum-resistant schemes:
    *   **WOTS Integration**: Following the proven pattern of Solana Winternitz Vaults (by @deanmlittle), the Trust Plugin can adopt **Winternitz One-Time Signatures** for Agent Authority keys. This hash-based approach (using Keccak256) provides quantum resistance by generating unique keypairs per transaction.
    *   **Lattice/Hash-Based Fallback**: Support for NIST-approved PQC algorithms (e.g., CRYSTALS-Dilithium, SPHINCS+) can be added as Solana's core infrastructure evolves.

## Implementation
The reference implementation will be provided as an open-source program that implements the `MplCorePlugin` interface.
