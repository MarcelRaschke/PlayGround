# NFT AST Research Framework

Deep research into Abstract Syntax Tree (AST) applications for generative NFT systems, smart contract analysis, decentralized metadata, and provenance tracking. This framework integrates blockchain, procedural generation, computer vision, and compliance infrastructure for production deployment.

## Four Research Domains

### 1. Smart Contract AST Analysis
**Security-focused parsing of EVM bytecode and source code for vulnerability detection and code quality metrics.**

- Reentrancy detection, integer overflow patterns, access control flaws
- Cyclomatic complexity measurement and gas optimization analysis
- Dependency graph generation from contract imports
- Interactive HTML reports with slither/mythril integration
- CI/CD integration points (Etherscan API, GitHub Actions)
- Tools: Python (slither, mythril, echidna) or Rust (tree-sitter, ethers-rs)

**Deliverables:** Parser wrapper, rule engine, report generator, CLI tool, Etherscan integration

### 2. Generative AST-Based NFTs
**Deterministic procedural art generation using AST as generative rule set.**

- TokenID → ChaCha20 PRNG → Context-free Grammar → Rendering Pipeline → Artwork
- L-systems for fractal generation, parametric rules with seed derivation
- Feature extraction: color palettes, geometry parameters, animation timing
- Rendering backends: SVG (web), WebGL (real-time audio-reactive), Canvas (2D animation)
- On-chain verification: Solidity contract with deterministic feature generation
- Off-chain verification: hash-based proof validation using Python
- Audio-visual synchronization for performance use (Blue Dimension radio integration)

**Deliverables:** Grammar DSL, SVG renderer, WebGL engine, Solidity contracts, verification suite

### 3. Metadata & URI Resolution
**Parsing, validating, and resolving NFT metadata across multiple decentralized storage backends.**

- Multi-gateway fallback strategy: IPFS → IPNS → Arweave → HTTPS → data:
- ERC-721/ERC-1155 standards with extended cy8er schema (creator, provenance, rights, AI attestation)
- JSON Schema validation with comprehensive error handling
- Content-addressable verification (IPFS hash matching, integrity checks)
- Audit trail versioning: track all metadata mutations, store immutably to Arweave
- Caching strategy: Redis with 24-hour TTL, exponential backoff retry logic
- Batch resolution API for bulk metadata operations

**Deliverables:** Python resolver library, JSON Schema definitions, IPFS integration, audit trail system

### 4. On-Chain Provenance & Artist Attestation
**Immutable proof of creation, integrity verification, and artist certification for @®† brand.**

- Smart contracts: ArtistRegistry (pseudonym + KYC), ProvenanceNFT (artifact proof), CertificateOfAuthenticity
- Digital signatures: ECDSA/EdDSA artist attestation with signature verification
- Merkle proofs: batch provenance verification tree with interactive proof generation
- Certificate generation: on-chain + PDF export via reportlab
- EU AI Act compliance: automated attestation forms, high-risk model documentation
- GDPR compliance: anonymization functions, artifact deletion requests, data minimization
- Comprehensive audit reports: timeline, integrity checks, compliance status

**Deliverables:** Solidity contracts, Python signature/verification suite, certificate generator, compliance framework

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│              On-Chain Layer (Ethereum/L2)                   │
├─────────────────────────────────────────────────────────────┤
│ ArtistRegistry │ ProvenanceNFT │ GenerativeNFT │ Certificate │
└────────────────────────┬────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
┌───────▼────────┐ ┌────▼──────────┐ ┌───▼──────────────┐
│ IPFS/Arweave   │ │ Merkle Proof  │ │ Oracle Services  │
│ Pinning Service│ │ Verification  │ │ (Chainlink/Band) │
└────────────────┘ └───────────────┘ └──────────────────┘
        │                                     │
        └─────────────────┬───────────────────┘
                         │
        ┌────────────────▼────────────────┐
        │    Off-Chain Services           │
        ├─────────────────────────────────┤
        │ • Metadata Resolver (Python)    │
        │ • Generative Engine (WebGL/SVG) │
        │ • Signature Verification        │
        │ • Compliance Check (EU AI Act)  │
        │ • Audit Trail Generator        │
        └─────────────────────────────────┘
        │                │                │
   ┌────▼──┐      ┌──────▼────────┐  ┌───▼──────────┐
   │ Redis │      │ PostgreSQL    │  │ Elasticsearch│
   │(Cache)│      │ (Provenance)  │  │ (Audit Logs) │
   └───────┘      └───────────────┘  └──────────────┘
```

## Technology Stack

### Blockchain & Smart Contracts
- Solidity 0.8+, OpenZeppelin contracts (ERC-721, ERC-1155, Ownable)
- Foundry for testing/deployment, Hardhat for development
- Chainlink oracles for cross-chain verification
- Layer 2: Arbitrum, Optimism for cost-efficient minting

### AST Parsing & Analysis
- Python: slither (static analysis), mythril (dynamic analysis), echidna (fuzzing), tree-sitter
- Rust: ethers-rs, tree-sitter-solidity, proptest
- Go: go-ethereum (geth), solc bindings

### Generative Systems
- Procedural generation: ChaCha20 PRNG, context-free grammars, L-systems
- Rendering: SVG (resvg), WebGL (Three.js), Canvas (p5.js)
- Audio analysis: librosa, Web Audio API for reactive visuals
- Determinism: strict seed-based output, reproducible across platforms

### Decentralized Storage
- IPFS: go-ipfs, kubo for local pinning
- Arweave: js-arweave for permanent storage
- Pinata/Estuary: API integration for redundancy
- DNS/IPNS: domain name resolution to IPFS content

### Verification & Cryptography
- eth_keys, py-ecc for ECDSA/EdDSA signatures
- merkletree (Merkle proof generation), hashlib for content verification
- JSON Schema validation, jsonschema library

### Database & Caching
- PostgreSQL for immutable provenance records
- Redis for metadata caching, session management
- Elasticsearch for audit trail search
- Arweave GraphQL for historical queries

### Compliance & Infrastructure
- Kubernetes: Helm charts for scalable deployment
- ArgoCD for GitOps-based deployments
- Terraform for infrastructure-as-code
- Cloudflare: DDoS protection, zero-trust reverse proxy

## Compliance Framework

### EU AI Act (High-Risk Classification)
- Automated attestation forms for AI-generated artwork
- Training data documentation and audit trails
- Human oversight mechanisms for certification
- Model card generation (bias, performance, limitations)
- Comprehensive documentation repository

### GDPR/revDSG (Switzerland)
- Data minimization: store only essential artist/provenance data
- Right to deletion: anonymization functions for removed artifacts
- Pseudonymity: non-obvious usernames (e.g., mraschke_admin)
- Data residency: Swiss/EU data centers, no US cloud providers
- Privacy-by-design: end-to-end encryption for sensitive metadata

### Smart Contract Security
- Formal verification for critical contracts (VeriFresh, Coq)
- Audit trail immutability: no contract upgrades without governance
- Access control: multi-sig for artist registry modifications
- Rate limiting: prevent metadata spam attacks

### Artistic Integrity
- Artist attestation: cryptographic proof of creation
- Attribution immutability: provenance chain cannot be altered
- Creative commons licensing: enforce original intent
- Derivative work tracking: link back to source artifacts

## Roadmap

### Phase 0: Specification (Current)
Research and specification across four domains. Architectural design, technology selection, compliance framework definition, decision records.

### Phase 1: Smart Contract Foundation (Weeks 1-2)
Deploy ArtistRegistry, ProvenanceNFT, GenerativeNFT contracts. Implement Merkle tree structures. Begin Etherscan integration.

### Phase 2: Metadata Infrastructure (Weeks 2-3)
Deploy metadata resolver service. Implement IPFS/Arweave pinning. Build caching layer. Audit trail versioning system.

### Phase 3: Generative Engine (Weeks 3-4)
Complete SVG/WebGL renderers. Deploy ChaCha20-based PRNG. Implement signature verification. Audio-visual synchronization.

### Phase 4: @®† Brand Launch (Weeks 4+)
Integrate with artist registry. Deploy certificate authority. Full compliance stack (EU AI Act, GDPR). Production Kubernetes rollout.

## References

- Solidity Documentation: https://docs.soliditylang.org/
- ERC-721 Standard: https://eips.ethereum.org/EIPS/eip-721
- ERC-1155 Standard: https://eips.ethereum.org/EIPS/eip-1155
- IPFS Specification: https://ipfs.tech/
- Merkle Trees: https://en.wikipedia.org/wiki/Merkle_tree
- Context-Free Grammars: https://en.wikipedia.org/wiki/Context-free_grammar
- L-Systems: https://en.wikipedia.org/wiki/L-system
- EU AI Act: https://www.europarl.europa.eu/topics/en/article/20230601/eu-ai-act-passed
- GDPR: https://gdpr-info.eu/
- revDSG (Swiss GDPR): https://www.edoeb.admin.ch/edoeb/de/home/datenschutz/gesetze/revdsg.html

## Getting Started

See individual SPEC.md files in each domain subdirectory for detailed implementation guidance:

- `ast-analysis/SPEC.md` - Vulnerability detection and code quality
- `generative/SPEC.md` - Procedural art generation pipeline
- `metadata/SPEC.md` - Decentralized metadata resolution
- `provenance/SPEC.md` - Artist attestation and certificate authority
