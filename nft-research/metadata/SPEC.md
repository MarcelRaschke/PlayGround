# Metadata & URI Resolution Specification

## Overview

NFT metadata is the bridge between on-chain contracts and off-chain context. This specification defines a production-ready system for resolving, validating, versioning, and caching metadata across multiple decentralized storage backends (IPFS, Arweave, HTTPS) with automatic fallback, integrity verification, and immutable audit trails.

## Metadata Standards

### ERC-721 Standard (NFT) Baseline

```json
{
  "name": "Generation #4392",
  "description": "Procedurally generated artwork from @®† artist registry",
  "image": "ipfs://QmXxxx.../artwork.svg",
  "external_url": "https://djjessejay.ch/nft/4392",
  "attributes": [
    {
      "trait_type": "Geometry",
      "value": "Mandelbrot Set"
    },
    {
      "trait_type": "Primary Color",
      "value": "#FF5733"
    }
  ]
}
```

### ERC-1155 Standard (Semi-Fungible) Extension

```json
{
  "name": "Blue Dimension Episode 42",
  "description": "Generative audio-visual performance recording",
  "image": "ipfs://QmXxxx.../episode-42.png",
  "animation_url": "ipfs://QmXxxx.../episode-42.webm",
  "properties": {
    "bpm": 128,
    "key": "A",
    "duration_seconds": 3600
  },
  "royalties": {
    "artist": {
      "address": "0x...",
      "percentage": 10.0
    }
  }
}
```

### Extended Cy8er Schema (Generative NFT + @®† Provenance)

```json
{
  "name": "Genesis #1",
  "description": "Generative artwork from artist cy8er",
  "image": "ipfs://QmXxxx.../artwork.svg",
  "external_url": "https://djjessejay.ch/nft/1",
  
  "creator": {
    "pseudonym": "cy8er",
    "address": "0x...",
    "publicKey": "0x...",
    "registry_contract": "0x...",
    "kyc_verified": true,
    "kyc_level": "advanced"
  },
  
  "generation": {
    "algorithm": "ChaCha20-CFG",
    "seed_derivation": "keccak256(tokenId)",
    "reproducible": true,
    "verification_hash": "sha256(artwork_bytes)"
  },
  
  "provenance": {
    "created_timestamp": "2026-09-29T03:00:00Z",
    "minted_timestamp": "2026-09-29T03:15:00Z",
    "minted_by": "0x...",
    "merkle_root": "0x...",
    "certificate_contract": "0x...",
    "certificate_id": "cert-001"
  },
  
  "rights": {
    "license": "CC-BY-4.0",
    "derivatives_allowed": true,
    "commercial_use": false,
    "attribution_required": true
  },
  
  "ai_attestation": {
    "contains_ai_generated_content": true,
    "ai_model_used": "llama2-70b (local, offline)",
    "training_data_consent": "obtained",
    "eu_ai_act_compliance": "tier-2-high-risk",
    "high_risk_assessment_date": "2026-09-29"
  },
  
  "attributes": [
    {
      "trait_type": "Geometry Type",
      "value": "Mandelbrot",
      "trait_rarity": 0.15
    },
    {
      "trait_type": "Color Palette",
      "value": "Synthwave Neon",
      "trait_rarity": 0.23
    }
  ]
}
```

## URI Schemes and Resolution

### Supported URI Schemes (Priority Order)

1. **IPFS (Content-Addressable)**
   - Format: `ipfs://QmXxxx...` or `ipfs://<hash>`
   - Advantages: Censorship-resistant, content-verified, global network
   - Fallback: Multiple IPFS gateways

2. **IPNS (Mutable Pointer)**
   - Format: `ipns://k2k4r8...` (IPNS key hash)
   - Advantages: Mutable while maintaining content addressing
   - Tradeoff: Slower resolution than IPFS

3. **Arweave (Permanent Storage)**
   - Format: `ar://TxHashXxxx...`
   - Advantages: 200+ year storage guarantee, immutable
   - Cost: Per-byte storage fee, no deletion

4. **HTTPS (Centralized Fallback)**
   - Format: `https://example.com/metadata/1.json`
   - Advantages: Fast, familiar protocol
   - Risk: Single point of failure, censorship possible

5. **Data URI (Embedded)**
   - Format: `data:application/json;base64,eyJ...`
   - Advantages: No external dependency
   - Tradeoff: Limited size, not updatable

### Resolution Strategy

```python
class URIResolver:
    RESOLUTION_ORDER = [
        "ipfs",
        "ipns",
        "arweave",
        "https",
        "data"
    ]
    
    def resolve(self, uri: str, timeout: int = 30) -> dict:
        """
        Resolve metadata URI with automatic fallback
        
        Args:
            uri: NFT metadata URI (ipfs://, ar://, https://, etc.)
            timeout: Seconds to wait per gateway before fallback
        
        Returns:
            Parsed metadata dict
        
        Raises:
            ResolutionError: All resolution attempts failed
        """
        scheme = self._extract_scheme(uri)
        
        if scheme == "ipfs":
            return self._resolve_ipfs(uri, timeout)
        elif scheme == "ipns":
            return self._resolve_ipns(uri, timeout)
        elif scheme == "ar":
            return self._resolve_arweave(uri, timeout)
        elif scheme == "https":
            return self._resolve_https(uri, timeout)
        elif scheme == "data":
            return self._resolve_data_uri(uri)
        else:
            raise URIFormatError(f"Unsupported scheme: {scheme}")
    
    def _resolve_ipfs(self, uri: str, timeout: int) -> dict:
        """Resolve IPFS content through multiple gateways"""
        ipfs_hash = self._extract_hash(uri)
        
        # Multiple gateway providers for redundancy
        gateways = [
            f"https://ipfs.io/ipfs/{ipfs_hash}",
            f"https://gateway.pinata.cloud/ipfs/{ipfs_hash}",
            f"https://dweb.link/ipfs/{ipfs_hash}",
            f"http://localhost:5001/api/v0/cat?arg={ipfs_hash}"  # Local node
        ]
        
        for gateway_url in gateways:
            try:
                response = requests.get(gateway_url, timeout=timeout)
                response.raise_for_status()
                metadata = response.json()
                
                # Verify content hash
                if self._verify_content_hash(metadata, ipfs_hash):
                    return metadata
                else:
                    raise IntegrityError("Content hash mismatch")
            
            except (requests.RequestException, ValueError, IntegrityError) as e:
                continue  # Try next gateway
        
        raise ResolutionError(f"Failed to resolve IPFS: {ipfs_hash}")
    
    def _resolve_arweave(self, uri: str, timeout: int) -> dict:
        """Resolve from Arweave permanent storage"""
        tx_hash = self._extract_hash(uri)
        
        arweave_url = f"https://arweave.net/{tx_hash}"
        response = requests.get(arweave_url, timeout=timeout)
        response.raise_for_status()
        
        return response.json()
    
    def _verify_content_hash(self, content: dict, expected_hash: str) -> bool:
        """Verify content integrity"""
        content_bytes = json.dumps(content, sort_keys=True).encode()
        computed_hash = hashlib.sha256(content_bytes).hexdigest()
        
        # For IPFS, compare against the content hash stored in URI
        return computed_hash.startswith(expected_hash[:8])
```

## Schema Validation

### JSON Schema Definition

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Generative NFT Metadata (Cy8er Extension)",
  "type": "object",
  "required": ["name", "image", "creator", "generation"],
  
  "properties": {
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 256
    },
    
    "description": {
      "type": "string",
      "maxLength": 2048
    },
    
    "image": {
      "type": "string",
      "pattern": "^(ipfs://|ar://|https://|data:)"
    },
    
    "creator": {
      "type": "object",
      "required": ["pseudonym", "address"],
      "properties": {
        "pseudonym": {"type": "string"},
        "address": {
          "type": "string",
          "pattern": "^0x[a-fA-F0-9]{40}$"
        },
        "publicKey": {"type": "string"},
        "kyc_verified": {"type": "boolean"}
      }
    },
    
    "generation": {
      "type": "object",
      "required": ["algorithm", "seed_derivation"],
      "properties": {
        "algorithm": {"type": "string"},
        "seed_derivation": {"type": "string"},
        "reproducible": {"type": "boolean"},
        "verification_hash": {
          "type": "string",
          "pattern": "^[a-fA-F0-9]{64}$"
        }
      }
    },
    
    "ai_attestation": {
      "type": "object",
      "properties": {
        "contains_ai_generated_content": {"type": "boolean"},
        "eu_ai_act_compliance": {
          "type": "string",
          "enum": ["tier-1-minimal", "tier-2-high-risk", "tier-3-prohibited"]
        }
      }
    }
  }
}
```

### Validation Implementation

```python
from jsonschema import validate, ValidationError, Draft7Validator

class MetadataValidator:
    def __init__(self, schema_path: str):
        with open(schema_path) as f:
            self.schema = json.load(f)
        self.validator = Draft7Validator(self.schema)
    
    def validate(self, metadata: dict) -> ValidationResult:
        """
        Validate metadata against schema
        
        Returns:
            ValidationResult with errors and warnings
        """
        errors = []
        warnings = []
        
        # Required fields
        try:
            validate(instance=metadata, schema=self.schema)
        except ValidationError as e:
            errors.append(str(e))
        
        # Additional checks
        if "ai_attestation" in metadata:
            if metadata["ai_attestation"].get("contains_ai_generated_content"):
                if "eu_ai_act_compliance" not in metadata["ai_attestation"]:
                    warnings.append("AI content present but compliance tier missing")
        
        return ValidationResult(
            valid=len(errors) == 0,
            errors=errors,
            warnings=warnings
        )
```

## Metadata Audit Trail

### Versioning System

```python
class MetadataHistory:
    def __init__(self, token_id: int):
        self.token_id = token_id
        self.versions: List[MetadataVersion] = []
    
    def add_version(self, metadata: dict, uri: str, arweave_hash: str):
        """Record metadata version"""
        version = MetadataVersion(
            version_number=len(self.versions) + 1,
            timestamp=datetime.utcnow(),
            content_hash=hashlib.sha256(
                json.dumps(metadata, sort_keys=True).encode()
            ).hexdigest(),
            uri=uri,
            arweave_transaction=arweave_hash,
            metadata=metadata
        )
        self.versions.append(version)
    
    def get_change_history(self) -> List[dict]:
        """Return timeline of metadata changes"""
        changes = []
        
        for i in range(1, len(self.versions)):
            prev = self.versions[i-1]
            curr = self.versions[i]
            
            diff = self._compute_diff(prev.metadata, curr.metadata)
            
            changes.append({
                "version": curr.version_number,
                "timestamp": curr.timestamp.isoformat(),
                "changed_fields": diff,
                "archived_to": curr.arweave_transaction
            })
        
        return changes
    
    def _compute_diff(self, prev: dict, curr: dict) -> dict:
        """Compute field-level differences"""
        return {
            "added": {k: v for k, v in curr.items() if k not in prev},
            "removed": {k: v for k, v in prev.items() if k not in curr},
            "modified": {
                k: {"from": prev[k], "to": curr[k]}
                for k in set(prev.keys()) & set(curr.keys())
                if prev[k] != curr[k]
            }
        }
```

### Immutable Storage to Arweave

```python
class ImmutableMetadataArchive:
    def __init__(self, arweave_api_key: str):
        self.arweave = ArweaveClient(api_key=arweave_api_key)
    
    def archive_metadata(self, token_id: int, metadata: dict) -> str:
        """
        Store metadata permanently on Arweave
        
        Returns:
            Transaction hash for permanent storage
        """
        payload = {
            "token_id": token_id,
            "archived_timestamp": datetime.utcnow().isoformat(),
            "metadata": metadata,
            "content_hash": hashlib.sha256(
                json.dumps(metadata, sort_keys=True).encode()
            ).hexdigest()
        }
        
        # Store with retention guarantee
        tx = self.arweave.send_transaction(
            data=json.dumps(payload),
            tags={
                "Content-Type": "application/json",
                "Token-ID": str(token_id),
                "Archive-Type": "nft-metadata",
                "Retention-Years": "200"
            }
        )
        
        return tx.id
```

## IPFS Pinning Integration

### Redundant Pinning Service

```python
class PinningService:
    def __init__(self, pinata_api_key: str, pinata_secret: str):
        self.pinata_api_key = pinata_api_key
        self.pinata_secret = pinata_secret
        self.pinata_url = "https://api.pinata.cloud"
    
    def pin_metadata(self, metadata: dict, token_id: int) -> str:
        """
        Pin metadata to Pinata with automatic renewal
        
        Returns:
            IPFS hash
        """
        # Convert to JSON with deterministic ordering
        json_content = json.dumps(metadata, sort_keys=True)
        
        files = {
            'file': (f'metadata-{token_id}.json', json_content)
        }
        
        response = requests.post(
            f"{self.pinata_url}/pinning/pinFileToIPFS",
            files=files,
            headers={
                'pinata_api_key': self.pinata_api_key,
                'pinata_secret_api_key': self.pinata_secret
            }
        )
        
        ipfs_hash = response.json()['IpfsHash']
        
        # Set automatic renewal
        self._set_pin_policy(ipfs_hash, token_id)
        
        return ipfs_hash
    
    def _set_pin_policy(self, ipfs_hash: str, token_id: int):
        """Configure automatic renewal"""
        response = requests.put(
            f"{self.pinata_url}/pinning/hashMetadata",
            json={
                "ipfsHash": ipfs_hash,
                "name": f"Generation #{token_id}",
                "keyvalues": {
                    "token_id": str(token_id),
                    "renewal_policy": "annual"
                }
            },
            headers=self._auth_headers()
        )
        response.raise_for_status()
```

## Caching Strategy

### Redis-Based Metadata Cache

```python
class MetadataCache:
    def __init__(self, redis_host: str = "localhost", ttl_hours: int = 24):
        self.redis = redis.Redis(host=redis_host, decode_responses=True)
        self.ttl_seconds = ttl_hours * 3600
    
    def get(self, token_id: int) -> Optional[dict]:
        """Retrieve cached metadata"""
        cache_key = f"nft:metadata:{token_id}"
        cached = self.redis.get(cache_key)
        
        if cached:
            return json.loads(cached)
        return None
    
    def set(self, token_id: int, metadata: dict):
        """Cache metadata with TTL"""
        cache_key = f"nft:metadata:{token_id}"
        self.redis.setex(
            cache_key,
            self.ttl_seconds,
            json.dumps(metadata)
        )
    
    def invalidate(self, token_id: int):
        """Clear cache for token"""
        cache_key = f"nft:metadata:{token_id}"
        self.redis.delete(cache_key)
```

## Error Handling

### Custom Exception Hierarchy

```python
class MetadataError(Exception):
    """Base exception for metadata operations"""
    pass

class URIFormatError(MetadataError):
    """Invalid URI format"""
    pass

class ResolutionError(MetadataError):
    """Failed to resolve metadata from all sources"""
    pass

class ValidationError(MetadataError):
    """Metadata fails schema validation"""
    pass

class IntegrityError(MetadataError):
    """Content hash mismatch or corruption detected"""
    pass
```

### Retry Logic with Exponential Backoff

```python
class ResilientResolver:
    def resolve_with_retry(self, uri: str, max_retries: int = 3) -> dict:
        """
        Resolve with exponential backoff retry
        
        Retry delays: 1s, 2s, 4s
        """
        backoff_base = 1  # seconds
        
        for attempt in range(max_retries):
            try:
                return self.resolver.resolve(uri)
            except ResolutionError as e:
                if attempt == max_retries - 1:
                    raise
                
                wait_time = backoff_base * (2 ** attempt)
                time.sleep(wait_time)
        
        raise ResolutionError(f"Failed after {max_retries} attempts")
```

## Metadata Resolver API

### Complete Resolution Service

```python
class MetadataResolver:
    def __init__(self, cache: MetadataCache, validator: MetadataValidator):
        self.cache = cache
        self.validator = validator
        self.uri_resolver = URIResolver()
    
    def resolve(self, token_id: int, uri: str) -> dict:
        """
        Resolve, validate, cache metadata
        
        1. Check cache
        2. Resolve from URI
        3. Validate schema
        4. Cache result
        """
        # Check cache first
        cached = self.cache.get(token_id)
        if cached:
            return cached
        
        # Resolve from URI
        try:
            metadata = self.uri_resolver.resolve(uri)
        except Exception as e:
            raise ResolutionError(f"Failed to resolve {uri}: {e}")
        
        # Validate
        validation = self.validator.validate(metadata)
        if not validation.valid:
            raise ValidationError(f"Invalid metadata: {validation.errors}")
        
        # Cache
        self.cache.set(token_id, metadata)
        
        return metadata
    
    def batch_resolve(self, requests: List[Tuple[int, str]]) -> List[dict]:
        """Resolve multiple tokens in parallel"""
        with ThreadPoolExecutor(max_workers=10) as executor:
            futures = [
                executor.submit(self.resolve, token_id, uri)
                for token_id, uri in requests
            ]
            return [f.result() for f in futures]
    
    def get_history(self, token_id: int) -> List[dict]:
        """Retrieve metadata change history"""
        # Query audit trail from database
        pass
    
    def verify_chain(self, token_id: int) -> VerificationResult:
        """Verify provenance chain integrity"""
        # Check Merkle proofs, signatures, on-chain verification
        pass
```

## Phase 1 Implementation Plan

### Week 2-3: Infrastructure
- Metadata validator and schema definition
- IPFS/Arweave gateway integration
- Redis caching layer
- URI resolution with fallback strategy
- Integration tests with real IPFS nodes

### Batch Resolution API
- Parallel metadata fetching
- Error aggregation and reporting
- Performance monitoring

### Production Deployment
- Load testing (1000 concurrent requests)
- Failover testing (gateway unavailability)
- Cache coherency (update propagation)
- Monitoring dashboards (resolution time, cache hit rate)
