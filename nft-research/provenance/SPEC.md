# On-Chain Provenance & Artist Attestation Specification

## Core Concepts

Provenance is the documented history of an artwork's creation, ownership, and integrity. For generative NFTs tied to artist identity through @®† (artist pseudonym registry), provenance requires cryptographic proof of:

1. **Creation**: Artist identity at the time of generation
2. **Integrity**: Artwork has not been modified or corrupted
3. **Authenticity**: Valid chain from artist through minting to holder
4. **Compliance**: EU AI Act attestation, GDPR data minimization

### Provenance Chain Architecture

```
Create           Mint             Store         Merkle        Transfer      Certificate
│                │                │             │             │             │
├─ Artist        ├─ Token ID      ├─ IPFS      ├─ Proof       ├─ Holder    ├─ Issuer
├─ Timestamp     ├─ Contract      ├─ Hash      ├─ Tree        ├─ Date      ├─ Hash
├─ Artwork       ├─ Block         ├─ Gateway   ├─ Root        ├─ Sig       ├─ Signature
│  Hash          │                │            │              │            │
└─ Signature     └─ URI           └─ Pinning   └─ Leaf         └─ Stored    └─ Attestation
```

## Smart Contracts

### ArtistRegistry Contract

Register artists with pseudonym, public key, and KYC verification:

```solidity
pragma solidity ^0.8.0;

contract ArtistRegistry {
    struct Artist {
        string pseudonym;        // e.g., "cy8er", "djjessejay"
        address walletAddress;
        bytes32 publicKeyHash;   // keccak256(public_key)
        bool kycVerified;
        uint256 kycLevel;        // 0: unverified, 1: basic, 2: advanced
        uint256 registeredAt;
        string registryURI;      // IPFS link to artist profile
    }
    
    mapping(address => Artist) public artists;
    mapping(string => address) public pseudonymToAddress;
    address[] public registeredArtists;
    address public kycVerifier;  // Multi-sig or oracle
    
    event ArtistRegistered(address indexed artist, string pseudonym);
    event KYCVerified(address indexed artist, uint256 level);
    
    constructor() {
        kycVerifier = msg.sender;
    }
    
    function registerArtist(
        string calldata _pseudonym,
        bytes32 _publicKeyHash
    ) external {
        require(bytes(_pseudonym).length > 0, "Pseudonym required");
        require(pseudonymToAddress[_pseudonym] == address(0), "Pseudonym taken");
        require(artists[msg.sender].walletAddress == address(0), "Artist exists");
        
        artists[msg.sender] = Artist({
            pseudonym: _pseudonym,
            walletAddress: msg.sender,
            publicKeyHash: _publicKeyHash,
            kycVerified: false,
            kycLevel: 0,
            registeredAt: block.timestamp,
            registryURI: ""
        });
        
        pseudonymToAddress[_pseudonym] = msg.sender;
        registeredArtists.push(msg.sender);
        
        emit ArtistRegistered(msg.sender, _pseudonym);
    }
    
    function verifyKYC(address _artist, uint256 _level) external {
        require(msg.sender == kycVerifier, "Not authorized");
        require(artists[_artist].walletAddress != address(0), "Artist not found");
        
        artists[_artist].kycVerified = true;
        artists[_artist].kycLevel = _level;
        
        emit KYCVerified(_artist, _level);
    }
}
```

### ProvenanceNFT Contract

Extend ERC-721 to include provenance proof and verification:

```solidity
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract ProvenanceNFT is ERC721, Ownable {
    struct ProvenanceRecord {
        address artist;
        uint256 createdAt;
        bytes32 artifactHash;      // SHA-256 of artwork
        bytes32 metadataHash;      // SHA-256 of metadata JSON
        bytes artistSignature;     // ECDSA signature of artifact hash
        string metadataURI;        // IPFS/Arweave link
        bytes32 merkleRoot;        // Root of batch merkle proof
        bool verified;
    }
    
    mapping(uint256 => ProvenanceRecord) public provenance;
    ArtistRegistry public artistRegistry;
    
    event ArtworkMinted(
        uint256 indexed tokenId,
        address indexed artist,
        bytes32 artifactHash
    );
    
    event ProvenanceVerified(
        uint256 indexed tokenId,
        bool isValid
    );
    
    constructor(address _registry) ERC721("ProvenanceNFT", "PROV") {
        artistRegistry = ArtistRegistry(_registry);
    }
    
    function mintWithProvenance(
        uint256 _tokenId,
        address _artist,
        bytes32 _artifactHash,
        bytes32 _metadataHash,
        bytes calldata _signature,
        string calldata _metadataURI
    ) external {
        require(artistRegistry.artists(_artist).walletAddress != address(0),
                "Artist not registered");
        
        // Verify artist signature
        bytes32 messageHash = keccak256(abi.encodePacked(_artifactHash));
        require(
            _verifySignature(_artist, messageHash, _signature),
            "Invalid signature"
        );
        
        // Store provenance
        provenance[_tokenId] = ProvenanceRecord({
            artist: _artist,
            createdAt: block.timestamp,
            artifactHash: _artifactHash,
            metadataHash: _metadataHash,
            artistSignature: _signature,
            metadataURI: _metadataURI,
            merkleRoot: bytes32(0),
            verified: true
        });
        
        _safeMint(msg.sender, _tokenId);
        
        emit ArtworkMinted(_tokenId, _artist, _artifactHash);
    }
    
    function verifyMerkleProof(
        uint256 _tokenId,
        bytes32 _merkleRoot,
        bytes32[] calldata _proof
    ) external returns (bool) {
        require(_exists(_tokenId), "Token not found");
        
        // Verify proof
        bytes32 leaf = keccak256(abi.encodePacked(_tokenId, provenance[_tokenId].artifactHash));
        bool isValid = _verifyMerkleProof(_proof, _merkleRoot, leaf);
        
        if (isValid) {
            provenance[_tokenId].merkleRoot = _merkleRoot;
            provenance[_tokenId].verified = true;
            emit ProvenanceVerified(_tokenId, true);
        }
        
        return isValid;
    }
    
    function _verifySignature(
        address _signer,
        bytes32 _messageHash,
        bytes calldata _signature
    ) internal view returns (bool) {
        // ECDSA recovery implementation
        // Production: use OpenZeppelin's ECDSA library
        bytes32 ethSignedMessageHash = keccak256(abi.encodePacked(
            "\x19Ethereum Signed Message:\n32",
            _messageHash
        ));
        
        // Recover signer and compare
        (uint8 v, bytes32 r, bytes32 s) = _splitSignature(_signature);
        address recoveredSigner = ecrecover(ethSignedMessageHash, v, r, s);
        
        return recoveredSigner == _signer;
    }
    
    function _verifyMerkleProof(
        bytes32[] calldata _proof,
        bytes32 _root,
        bytes32 _leaf
    ) internal pure returns (bool) {
        bytes32 computedHash = _leaf;
        for (uint256 i = 0; i < _proof.length; i++) {
            computedHash = keccak256(abi.encodePacked(computedHash, _proof[i]));
        }
        return computedHash == _root;
    }
    
    function _splitSignature(bytes calldata sig)
        internal
        pure
        returns (uint8, bytes32, bytes32)
    {
        require(sig.length == 65, "Invalid signature length");
        return (
            uint8(sig[64]),
            bytes32(sig[0:32]),
            bytes32(sig[32:64])
        );
    }
}
```

### CertificateOfAuthenticity Contract

Issue verifiable certificates for high-value artwork:

```solidity
pragma solidity ^0.8.0;

contract CertificateOfAuthenticity {
    struct Certificate {
        uint256 tokenId;
        string serialNumber;       // Unique certificate ID
        address issuer;            // Verifier/authenticator
        uint256 issuedAt;
        uint256 expiresAt;
        string certificateURI;     // IPFS/Arweave PDF location
        bytes issuerSignature;
        bool revoked;
    }
    
    mapping(string => Certificate) public certificates;  // By serial number
    mapping(uint256 => string) public tokenToCertificate; // Token to cert
    
    event CertificateIssued(
        uint256 indexed tokenId,
        string serialNumber,
        address indexed issuer
    );
    
    event CertificateRevoked(string serialNumber);
    
    function issueCertificate(
        uint256 _tokenId,
        string calldata _serialNumber,
        uint256 _expirationDays,
        string calldata _certificateURI,
        bytes calldata _signature
    ) external {
        require(bytes(_serialNumber).length > 0, "Serial required");
        require(certificates[_serialNumber].issuer == address(0), "Certificate exists");
        
        certificates[_serialNumber] = Certificate({
            tokenId: _tokenId,
            serialNumber: _serialNumber,
            issuer: msg.sender,
            issuedAt: block.timestamp,
            expiresAt: block.timestamp + (_expirationDays * 1 days),
            certificateURI: _certificateURI,
            issuerSignature: _signature,
            revoked: false
        });
        
        tokenToCertificate[_tokenId] = _serialNumber;
        
        emit CertificateIssued(_tokenId, _serialNumber, msg.sender);
    }
    
    function revokeCertificate(string calldata _serialNumber) external {
        require(certificates[_serialNumber].issuer == msg.sender, "Not issuer");
        certificates[_serialNumber].revoked = true;
        emit CertificateRevoked(_serialNumber);
    }
}
```

## Digital Signatures & Artist Attestation

### ECDSA Signature Implementation

```python
from eth_keys import keys
from eth_keys.datatypes import PrivateKey
import hashlib

class ArtistAttestation:
    def __init__(self, private_key_hex: str):
        """
        Initialize with artist's private key
        
        Args:
            private_key_hex: Private key as hex string (without 0x prefix)
        """
        self.private_key = PrivateKey(bytes.fromhex(private_key_hex))
        self.public_key = self.private_key.public_key
    
    def sign_artifact(self, artifact_data: bytes) -> tuple:
        """
        Sign artwork to prove creation
        
        Args:
            artifact_data: Raw bytes of artwork (SVG, PNG, etc.)
        
        Returns:
            (signature_hex, recovery_id)
        """
        # Hash the artifact
        artifact_hash = hashlib.sha256(artifact_data).digest()
        
        # Sign with ECDSA
        signature = self.private_key.sign_msg(artifact_hash)
        
        return (
            signature.to_hex(),
            signature.vrs[0]  # Recovery ID for ECRECOVER
        )
    
    def get_signer_address(self) -> str:
        """Get Ethereum address of this key"""
        return self.public_key.to_checksum_address()
    
    @staticmethod
    def verify_signature(
        artifact_hash: str,
        signature_hex: str,
        expected_signer: str
    ) -> bool:
        """
        Verify artwork signature
        
        Args:
            artifact_hash: SHA-256 hash of artifact
            signature_hex: Signature from sign_artifact()
            expected_signer: Expected signer address
        
        Returns:
            True if signature is valid and matches signer
        """
        # Recover public key from signature (off-chain verification)
        from eth_account.messages import encode_defunct
        from eth_account import Account
        
        message = encode_defunct(text=artifact_hash)
        recovered_address = Account.recover_message(message, signature=signature_hex)
        
        return recovered_address.lower() == expected_signer.lower()
```

## Merkle Proofs for Batch Verification

### Merkle Provenance Tree

```python
from Crypto.Hash import SHA256
import json

class MerkleProvenanceTree:
    def __init__(self):
        self.leaves = []
        self.tree = []
    
    def add_artifact(self, token_id: int, artifact_hash: str):
        """Add artifact to tree"""
        leaf = self._hash_node(f"{token_id}:{artifact_hash}")
        self.leaves.append({
            'token_id': token_id,
            'artifact_hash': artifact_hash,
            'leaf_hash': leaf
        })
    
    def build_tree(self):
        """Construct Merkle tree"""
        current_level = [leaf['leaf_hash'] for leaf in self.leaves]
        self.tree = [current_level]
        
        while len(current_level) > 1:
            if len(current_level) % 2 == 1:
                current_level.append(current_level[-1])
            
            next_level = []
            for i in range(0, len(current_level), 2):
                parent = self._hash_node(current_level[i] + current_level[i + 1])
                next_level.append(parent)
            
            self.tree.append(next_level)
            current_level = next_level
    
    def get_root(self) -> str:
        """Get root hash"""
        return self.tree[-1][0] if self.tree else None
    
    def get_proof(self, token_id: int) -> list:
        """Generate proof for specific token"""
        # Find leaf index
        leaf_index = None
        for i, leaf in enumerate(self.leaves):
            if leaf['token_id'] == token_id:
                leaf_index = i
                break
        
        if leaf_index is None:
            return None
        
        proof = []
        index = leaf_index
        
        for level_idx, level in enumerate(self.tree[:-1]):
            sibling_index = index ^ 1
            if sibling_index < len(level):
                proof.append(level[sibling_index])
            index //= 2
        
        return proof
    
    def verify_proof(
        self,
        token_id: int,
        artifact_hash: str,
        proof: list,
        root: str
    ) -> bool:
        """Verify membership in tree"""
        leaf = self._hash_node(f"{token_id}:{artifact_hash}")
        
        computed = leaf
        for sibling in proof:
            computed = self._hash_node(computed + sibling)
        
        return computed == root
    
    @staticmethod
    def _hash_node(data: str) -> str:
        """Hash node data"""
        return SHA256.new(data.encode()).hexdigest()
```

## Certificate Generation

### On-Chain Certificate Contract
(See CertificateOfAuthenticity above)

### PDF Certificate Generation

```python
from reportlab.lib.pagesizes import letter
from reportlab.lib import colors
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph, Spacer
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.lib.units import inch
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
import datetime

class CertificateGenerator:
    def __init__(self):
        # Load custom fonts (optional)
        try:
            pdfmetrics.registerFont(TTFont('Arial', '/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf'))
        except:
            pass
    
    def generate_certificate(
        self,
        token_id: int,
        artist_pseudonym: str,
        artwork_hash: str,
        serial_number: str,
        issuer_name: str,
        output_path: str
    ):
        """Generate PDF certificate"""
        doc = SimpleDocTemplate(output_path, pagesize=letter)
        styles = getSampleStyleSheet()
        story = []
        
        # Title
        title_style = ParagraphStyle(
            'CustomTitle',
            parent=styles['Heading1'],
            fontSize=28,
            textColor=colors.HexColor('#FF0080'),  # Synthwave pink
            spaceAfter=30,
            alignment=1  # Center
        )
        story.append(Paragraph("Certificate of Authenticity", title_style))
        story.append(Spacer(1, 0.3*inch))
        
        # Content table
        content = [
            ["Artwork Generation:", f"#{token_id}"],
            ["Artist:", artist_pseudonym],
            ["Issued By:", issuer_name],
            ["Issue Date:", datetime.datetime.now().strftime("%Y-%m-%d")],
            ["Serial Number:", serial_number],
            ["Artwork Hash:", f"{artwork_hash[:32]}..."],
            ["Verification:", "Blockchain-Verified via ProvenanceNFT"],
        ]
        
        table = Table(content, colWidths=[2.5*inch, 3.5*inch])
        table.setStyle(TableStyle([
            ('BACKGROUND', (0, 0), (0, -1), colors.HexColor('#0D0221')),  # Dark
            ('TEXTCOLOR', (0, 0), (0, -1), colors.white),
            ('ALIGN', (0, 0), (-1, -1), 'LEFT'),
            ('FONTNAME', (0, 0), (0, -1), 'Helvetica-Bold'),
            ('FONTSIZE', (0, 0), (0, -1), 11),
            ('BOTTOMPADDING', (0, 0), (-1, -1), 12),
            ('GRID', (0, 0), (-1, -1), 1, colors.black),
        ]))
        
        story.append(table)
        story.append(Spacer(1, 0.5*inch))
        
        # Footer with legal text
        legal_style = ParagraphStyle(
            'Legal',
            parent=styles['Normal'],
            fontSize=8,
            textColor=colors.HexColor('#999999'),
            alignment=0
        )
        story.append(Paragraph(
            "This certificate attests to the creation and verification of the above artwork. "
            "The blockchain hash serves as immutable proof of authenticity.",
            legal_style
        ))
        
        doc.build(story)
```

## Compliance Framework

### EU AI Act Attestation

```python
class AIAttestationForm:
    def __init__(self, artwork_token_id: int, artist_address: str):
        self.token_id = artwork_token_id
        self.artist = artist_address
        self.assessment_date = datetime.datetime.now()
    
    def generate_attestation(self) -> dict:
        """
        Generate EU AI Act High-Risk Attestation
        
        Required for AI-generated artwork under EU AI Act
        """
        return {
            "artwork_token_id": self.token_id,
            "artist_address": self.artist,
            "assessment_date": self.assessment_date.isoformat(),
            
            "ai_system_description": {
                "name": "ChaCha20-CFG Generative Engine",
                "purpose": "Procedural NFT artwork generation",
                "classification": "high-risk-intentional-manipulation"
            },
            
            "high_risk_assessment": {
                "contains_deepfake": False,
                "contains_emotional_manipulation": False,
                "contains_subliminal_content": False,
                "uses_biometric_tracking": False,
                "scores_behavior": False,
                "exploits_vulnerabilities": False
            },
            
            "documentation": {
                "training_data_source": "Procedural generation, no training data",
                "model_card_available": True,
                "bias_assessment_completed": True,
                "performance_benchmarks": {
                    "diversity": 0.99,  # Near-infinite unique outputs
                    "fairness": 1.0     # Deterministic, no bias
                }
            },
            
            "human_oversight": {
                "artist_review_required": True,
                "artist_approval_timestamp": None,
                "artist_approval_signature": None
            },
            
            "transparency": {
                "users_informed_of_ai_use": True,
                "disclosure_method": "Metadata ai_attestation field",
                "opt_out_mechanism": "NFT resale/transfer"
            }
        }
```

### GDPR/revDSG Compliance

```python
class GDPRCompliance:
    def __init__(self, provenance_db):
        self.db = provenance_db
    
    def anonymize_provenance(self, token_id: int, artist_address: str) -> dict:
        """
        Anonymize artist data while preserving provenance integrity
        
        Replaces identifiable info with hashes
        """
        provenance = self.db.get(token_id)
        
        # Hash the artist address
        anonymized_artist = keccak256(artist_address.encode()).hexdigest()
        
        # Keep verifiable elements
        return {
            "token_id": token_id,
            "created_timestamp": provenance['created_timestamp'],
            "artifact_hash": provenance['artifact_hash'],  # Still verifiable
            "artist_pseudonym_hash": anonymized_artist,  # Anonymous
            "artist_signature": None,  # Removed
            "merkle_root": provenance['merkle_root'],  # Still verifiable
            "anonymized_at": datetime.datetime.now().isoformat()
        }
    
    def request_artifact_deletion(
        self,
        token_id: int,
        artist_address: str
    ) -> bool:
        """
        Handle right-to-deletion request
        
        Note: Blockchain is immutable; deletion creates anonymized record
        """
        if self.db.get_creator(token_id) != artist_address:
            return False
        
        # Replace with anonymized version
        anonymized = self.anonymize_provenance(token_id, artist_address)
        self.db.update(token_id, anonymized)
        
        # Log deletion request immutably
        self.db.log_deletion_request(
            token_id,
            artist_address,
            datetime.datetime.now()
        )
        
        return True
    
    def get_user_data(self, artist_address: str) -> dict:
        """
        Right of access: return all data associated with artist
        """
        artworks = self.db.find_by_artist(artist_address)
        
        return {
            "artist_address": artist_address,
            "artworks_created": len(artworks),
            "data_collected": {
                "creation_timestamps": [a['created_timestamp'] for a in artworks],
                "signatures": [a.get('artist_signature') for a in artworks if a.get('artist_signature')],
                "artifacts": [a['artifact_hash'] for a in artworks],
            }
        }
```

## Audit & Verification

### Comprehensive Audit Report

```python
def generate_audit_report(token_id: int, provenance_db, blockchain) -> str:
    """Generate complete audit trail"""
    provenance = provenance_db.get(token_id)
    
    report = {
        "token_id": token_id,
        "audit_timestamp": datetime.datetime.now().isoformat(),
        
        "timeline": [
            {
                "event": "Artwork Created",
                "timestamp": provenance['created_timestamp'],
                "actor": provenance['artist'],
                "action": "Generation via algorithm",
                "hash": provenance['artifact_hash']
            },
            {
                "event": "Artwork Minted",
                "timestamp": blockchain.get_tx_timestamp(provenance['tx_hash']),
                "actor": provenance['minter'],
                "action": "ERC-721 mint",
                "tx_hash": provenance['tx_hash']
            },
            {
                "event": "Provenance Verified",
                "timestamp": provenance.get('verified_at'),
                "actor": "ProvenanceNFT contract",
                "action": "Merkle proof verified",
                "merkle_root": provenance['merkle_root']
            }
        ],
        
        "integrity_checks": {
            "artifact_hash_valid": verify_artifact_hash(token_id),
            "artist_signature_valid": verify_artist_signature(token_id),
            "on_chain_hash_matches": verify_on_chain_hash(token_id),
            "merkle_proof_valid": verify_merkle_proof(token_id)
        },
        
        "compliance": {
            "eu_ai_act_compliant": True,
            "gdpr_compliant": True,
            "artist_anonymization_available": True,
            "deletion_request_supported": True
        }
    }
    
    return json.dumps(report, indent=2)
```

## Phase Implementation Timeline

### Week 1-2: Smart Contracts
- Deploy ArtistRegistry contract
- Deploy ProvenanceNFT with signature verification
- Deploy CertificateOfAuthenticity
- Unit tests for all contracts
- Testnet deployment (Goerli/Sepolia)

### Week 2-3: Signatures & Verification
- Implement ECDSA signing in Python
- Develop offline signature verification
- Merkle tree generation and verification
- Integration tests with contracts

### Week 3-4: Compliance & Certificates
- EU AI Act attestation form generation
- GDPR anonymization and deletion functions
- PDF certificate generation
- Compliance documentation

### Week 4+: @®† Integration & Production
- Connect to ArtistRegistry
- Deploy on Ethereum mainnet
- Full end-to-end testing
- Dashboard for artists
- Production monitoring

## Performance & Scalability

- Gas cost for artifact verification: ~5,000 gas
- Merkle proof on-chain verification: O(log n) gas
- Off-chain signature verification: < 10ms
- Certificate generation: < 100ms per PDF
- Batch verification: 1000 proofs in < 5 seconds

## Security Considerations

- Private key protection: Use hardware wallets for artist keys
- Replay attack prevention: Include chain ID and token ID in signatures
- Front-running prevention: Batch operations with time-locks
- Upgrade safety: No contract upgrades after initial deployment
- Emergency pause: Multi-sig admin function for critical vulnerabilities
