# Generative AST-Based NFTs Specification

## Core Concept

Generative AST-based NFTs treat the token as a generative rule set rather than a stored media file. The artwork emerges deterministically from the token ID through a pipeline: ChaCha20 PRNG → Context-Free Grammar → Feature Extraction → Rendering → Output. This approach creates infinite unique artworks, infinite reproducibility, and minimal on-chain storage footprint.

### Design Principles

1. **Determinism**: TokenID alone determines complete artwork. Same ID always produces identical output.
2. **Reproducibility**: Artwork can be regenerated on any device, any time (IPFS-proof)
3. **Minimal Storage**: No media files stored; only rule set and seed derivation algorithm
4. **Verifiability**: On-chain hash comparison proves artwork authenticity
5. **Audio-Reactivity**: Visuals respond to music parameters (BPM, frequency spectrum, waveform)

## Grammar Definition

### Context-Free Grammar (CFG) Structure

Define NFT artwork through formal grammar rules. Each rule expands to terminals (drawing operations) or non-terminals (recursive rules).

```
Start     → ArtisticSystem
ArtisticSystem → MainStructure + EphemeralLayer + AudioReactivity
MainStructure  → Geometry | Gradient | Fractal
Geometry       → Circle(radius, color) | Rectangle(width, height, color) | Line(x1, y1, x2, y2, color)
Gradient       → LinearGradient(color1, color2, angle) | RadialGradient(color1, color2, center)
Fractal        → MandelbrotSet | JuliaSet | LSystem
EphemeralLayer → Particle(count, type) | Noise(octaves, persistence)
AudioReactivity → ReactToFrequency(freq_band, parameter) | ReactToBeat(bpm_divisor, parameter)
```

### Parametric Rules with Seed Derivation

Each rule carries parameters derived deterministically from seed:

```python
def expand_rule(rule_name, seed, depth=0):
    """
    Expand grammar rule deterministically based on seed
    
    rule_name: string identifying the rule
    seed: integer from ChaCha20 PRNG
    depth: recursion depth for termination
    """
    if depth > MAX_DEPTH:
        return terminal_for_rule(rule_name, seed)
    
    # Deterministic choice of production
    productions = GRAMMAR[rule_name]
    choice_index = seed % len(productions)
    chosen_production = productions[choice_index]
    
    # Extract parameters from seed
    param_seed = mix_seed(seed, rule_name)
    parameters = derive_parameters(chosen_production, param_seed)
    
    # Recursively expand non-terminals
    subrules = extract_nonterminals(chosen_production)
    expanded = []
    for subrule in subrules:
        sub_seed = mix_seed(param_seed, subrule)
        expanded.append(expand_rule(subrule, sub_seed, depth + 1))
    
    return DrawingOperation(chosen_production, parameters, expanded)
```

### L-Systems for Fractal Generation

Lindenmayer systems generate plant-like structures and recursive fractals:

```
Axiom: F
Rules:
  F → F+G
  G → G-F
  + → RotateRight(90°)
  - → RotateLeft(90°)

Iteration 0: F
Iteration 1: F+G
Iteration 2: F+G+G-F
Iteration 3: F+G+G-F+G-F+G
```

**Implementation**:
```python
class LSystem:
    def __init__(self, axiom: str, rules: Dict[str, str], turtle_state: TurtleGraphics):
        self.axiom = axiom
        self.rules = rules
        self.turtle = turtle_state
    
    def expand(self, iterations: int) -> str:
        current = self.axiom
        for _ in range(iterations):
            current = ''.join(self.rules.get(char, char) for char in current)
        return current
    
    def render(self, system_string: str) -> List[DrawCommand]:
        commands = []
        for char in system_string:
            if char == 'F':
                commands.append(self.turtle.forward())
            elif char == 'G':
                commands.append(self.turtle.forward(distance=0.5))
            elif char == '+':
                commands.append(self.turtle.rotate(45))
            elif char == '-':
                commands.append(self.turtle.rotate(-45))
        return commands
```

## Seed Derivation Pipeline

### Stage 1: ChaCha20 PRNG

Use ChaCha20 stream cipher to generate deterministic pseudo-random numbers from token ID:

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
import struct

class DeterministicPRNG:
    def __init__(self, token_id: int):
        # Token ID as 256-bit seed (padded if necessary)
        self.seed = struct.pack('>Q', token_id).ljust(32, b'\x00')
        self.nonce = b'\x00' * 12
        self.state_index = 0
    
    def next_bytes(self, count: int) -> bytes:
        """Generate deterministic random bytes from token ID"""
        cipher = Cipher(
            algorithms.ChaCha20(self.seed, self.nonce),
            mode=modes.NullMode()
        )
        encryptor = cipher.encryptor()
        return encryptor.update(b'\x00' * count)
    
    def next_float(self, min_val: float = 0.0, max_val: float = 1.0) -> float:
        """Generate float in [min_val, max_val]"""
        random_bytes = self.next_bytes(4)
        random_int = struct.unpack('>I', random_bytes)[0]
        normalized = random_int / 0xFFFFFFFF
        return min_val + normalized * (max_val - min_val)
    
    def next_int(self, min_val: int, max_val: int) -> int:
        """Generate integer in [min_val, max_val)"""
        return min_val + int(self.next_float() * (max_val - min_val))
```

### Stage 2: Feature Extraction

Map PRNG output to artistic features:

```python
class FeatureExtractor:
    def __init__(self, prng: DeterministicPRNG):
        self.prng = prng
    
    def extract_features(self) -> ArtisticFeatures:
        return ArtisticFeatures(
            primary_color=self._extract_color(),
            secondary_color=self._extract_color(),
            tertiary_color=self._extract_color(),
            color_palette_type=self.prng.next_int(0, 5),  # 5 palette types
            geometry_type=self.prng.next_int(0, 8),       # 8 geometry types
            fractal_depth=self.prng.next_int(3, 8),
            animation_speed=self.prng.next_float(0.1, 2.0),
            symmetry_order=self.prng.next_int(1, 6),
            noise_scale=self.prng.next_float(0.01, 1.0),
            scale_factor=self.prng.next_float(0.5, 2.0),
            rotation_angle=self.prng.next_float(0, 360),
            particle_density=self.prng.next_int(10, 500),
        )
    
    def _extract_color(self) -> Color:
        """Extract 24-bit RGB color"""
        r = self.prng.next_int(0, 256)
        g = self.prng.next_int(0, 256)
        b = self.prng.next_int(0, 256)
        return Color(r, g, b)
```

### Stage 3: AST Construction

Build abstract syntax tree of drawing operations:

```python
class ASTBuilder:
    def __init__(self, features: ArtisticFeatures, prng: DeterministicPRNG):
        self.features = features
        self.prng = prng
    
    def build(self) -> DrawingAST:
        root = DrawingNode("Artwork")
        
        # Background layer
        bg_color = Color(self.features.primary_color.r // 2,
                        self.features.primary_color.g // 2,
                        self.features.primary_color.b // 2)
        root.add_child(DrawingNode("Background", 
                                   attrs={"fill": bg_color}))
        
        # Main geometry
        geometry = self._build_geometry()
        root.add_child(geometry)
        
        # Particle effects
        particles = self._build_particles()
        root.add_child(particles)
        
        # Animated elements (audio-reactive)
        animation = self._build_animation()
        root.add_child(animation)
        
        return root
    
    def _build_geometry(self) -> DrawingNode:
        node = DrawingNode("Geometry")
        geo_type = self.features.geometry_type
        
        if geo_type == 0:  # Mandelbrot
            node.add_child(self._build_mandelbrot())
        elif geo_type == 1:  # Spirals
            node.add_child(self._build_spiral())
        elif geo_type == 2:  # Trees (L-system)
            node.add_child(self._build_tree())
        # ... etc for other types
        
        return node
    
    def _build_mandelbrot(self) -> DrawingNode:
        """Generate Mandelbrot set"""
        # Implementation using iteration count per pixel
        pass
    
    def _build_spiral(self) -> DrawingNode:
        """Generate parametric spirals"""
        pass
    
    def _build_tree(self) -> DrawingNode:
        """Generate L-system tree"""
        lsystem = LSystem("F", {"F": "F+G", "G": "G-F"})
        iterations = self.features.fractal_depth
        system_string = lsystem.expand(iterations)
        # Render to drawing node
        pass
```

## Rendering Pipeline

### SVG Rendering (Web)

Generate scalable vector graphics for web display:

```python
class SVGRenderer:
    def __init__(self, width: int = 1024, height: int = 1024):
        self.width = width
        self.height = height
        self.svg = SVG(width, height)
    
    def render_ast(self, ast: DrawingAST) -> str:
        """Convert AST to SVG string"""
        self._render_node(ast.root, self.svg)
        return self.svg.to_string()
    
    def _render_node(self, node: DrawingNode, svg):
        if node.type == "Background":
            svg.rect(0, 0, self.width, self.height, 
                    fill=node.attrs["fill"].to_hex())
        elif node.type == "Circle":
            svg.circle(node.attrs["x"], node.attrs["y"], 
                      node.attrs["radius"], 
                      fill=node.attrs["color"].to_hex())
        elif node.type == "Path":
            svg.path(node.attrs["d"], stroke=node.attrs["stroke"].to_hex())
        
        for child in node.children:
            self._render_node(child, svg)
```

### WebGL Rendering (Real-Time Audio-Reactive)

GPU-accelerated rendering with audio parameter binding:

```glsl
#version 300 es
precision highp float;

uniform float uTime;
uniform float uBeat;           // Current beat value [0..1]
uniform float uFrequencyBand;  // Frequency data [0..1]
uniform float uScale;
uniform vec3 uPrimaryColor;

in vec2 vPosition;
out vec4 fragColor;

void main() {
    vec2 uv = vPosition / vec2(1024.0);
    
    // Audio-reactive distortion
    float audioInfluence = mix(uFrequencyBand, uBeat, 0.3);
    float distortion = sin(uv.x * 10.0 + uTime) * audioInfluence;
    
    vec3 color = uPrimaryColor;
    color += vec3(audioInfluence) * 0.3;  // Brighten on beat
    
    fragColor = vec4(color, 1.0);
}
```

**JavaScript Integration**:
```javascript
class WebGLAudioReactiveRenderer {
    constructor(canvas, audioContext) {
        this.gl = canvas.getContext('webgl2');
        this.audioContext = audioContext;
        this.analyser = audioContext.createAnalyser();
        this.frequencyData = new Uint8Array(this.analyser.frequencyBinCount);
    }
    
    animate() {
        this.analyser.getByteFrequencyData(this.frequencyData);
        
        const averageFrequency = this.frequencyData.reduce((a, b) => a + b) / 
                                 this.frequencyData.length / 255;
        
        this.gl.uniform1f(this.uniforms.uFrequencyBand, averageFrequency);
        this.gl.uniform1f(this.uniforms.uTime, Date.now() / 1000);
        
        this.gl.drawArrays(this.gl.TRIANGLES, 0, this.vertexCount);
        requestAnimationFrame(() => this.animate());
    }
}
```

### Canvas/P5.js (2D Animation)

Browser-based 2D animation with smooth transitions:

```javascript
let prng;
let features;
let ast;
let currentFrame = 0;

function setup() {
    createCanvas(1024, 1024);
    
    // Derive features from token ID
    const tokenId = window.location.hash.substr(1) || "1";
    prng = new DeterministicPRNG(tokenId);
    features = extractFeatures(prng);
    ast = buildAST(features, prng);
}

function draw() {
    background(features.primaryColor);
    
    // Animation
    translate(width / 2, height / 2);
    rotate(currentFrame * features.animationSpeed * 0.01);
    
    // Render geometry
    renderAST(ast);
    
    // Audio reaction
    if (audioActive) {
        const beatIntensity = getAudioBeat();
        scale(1 + beatIntensity * 0.1);
    }
    
    currentFrame++;
}
```

## On-Chain Verification

### Solidity Smart Contract

Deploy contract to verify NFT generation on-chain:

```solidity
pragma solidity ^0.8.0;

contract GenerativeNFT is ERC721 {
    // ChaCha20 implementation (simplified; use battle-tested library)
    function chacha20(uint256 tokenId) 
        internal 
        pure 
        returns (bytes32 seed) 
    {
        // Deterministic PRNG seed from token ID
        seed = keccak256(abi.encodePacked("chacha20", tokenId));
    }
    
    function deriveFeatures(uint256 tokenId) 
        public 
        pure 
        returns (FeatureSet memory) 
    {
        bytes32 seed = chacha20(tokenId);
        
        uint256 seedValue = uint256(seed);
        
        return FeatureSet({
            primaryColor: uint24(seedValue & 0xFFFFFF),
            geometryType: uint8((seedValue >> 24) % 8),
            fractalDepth: uint8(((seedValue >> 32) % 5) + 3),
            animationSpeed: uint8((seedValue >> 40) % 20),
            symmetryOrder: uint8(((seedValue >> 48) % 5) + 1)
        });
    }
    
    function verifyArtwork(
        uint256 tokenId,
        bytes32 expectedArtworkHash
    ) public pure returns (bool) {
        // Regenerate features
        FeatureSet memory features = deriveFeatures(tokenId);
        
        // Hash the features (off-chain provides full hash)
        bytes32 computedHash = keccak256(abi.encode(features));
        
        return computedHash == expectedArtworkHash;
    }
    
    struct FeatureSet {
        uint24 primaryColor;
        uint8 geometryType;
        uint8 fractalDepth;
        uint8 animationSpeed;
        uint8 symmetryOrder;
    }
}
```

## Off-Chain Verification

### Python Verification Suite

```python
def verify_artwork(token_id: int, expected_hash: str) -> bool:
    """
    Verify that rendered artwork matches expected hash
    
    Args:
        token_id: NFT token ID
        expected_hash: SHA-256 hash of expected artwork file
    
    Returns:
        True if artwork regenerated on-device matches expected
    """
    # Regenerate artwork locally
    prng = DeterministicPRNG(token_id)
    features = extract_features(prng)
    ast = build_ast(features, prng)
    
    # Render to SVG
    renderer = SVGRenderer()
    svg_output = renderer.render_ast(ast)
    
    # Compute hash
    generated_hash = hashlib.sha256(svg_output.encode()).hexdigest()
    
    return generated_hash == expected_hash
```

## @®† Integration

### Artist Registry Connection

Link generative NFTs to artist pseudonym:

```python
class GenerativeArtworkWithArtist:
    def __init__(self, token_id: int, artist_address: str):
        self.token_id = token_id
        self.artist_address = artist_address  # @®† registry entry
        self.artwork = GenerativeArtwork(token_id)
    
    def get_metadata(self) -> dict:
        return {
            "name": f"Generation #{self.token_id}",
            "description": "Generative artwork from @®† artist",
            "image": f"ipfs://{self.artwork.ipfs_hash}",
            "attributes": [
                {"trait_type": "Artist", "value": self.artist_address},
                {"trait_type": "Geometry", "value": self.artwork.features.geometry_type},
                {"trait_type": "Fractal Depth", "value": self.artwork.features.fractal_depth},
                {"trait_type": "Animation Speed", "value": self.artwork.features.animation_speed},
            ],
            "artist": {
                "address": self.artist_address,
                "registry_contract": "0x...",  # ProvenanceNFT contract
                "signed_by": "artist_public_key"
            }
        }
```

## Rendering Outputs

### SVG Export
```xml
<svg width="1024" height="1024" viewBox="0 0 1024 1024">
  <defs>
    <linearGradient id="grad1">
      <stop offset="0%" style="stop-color:rgb(255,0,0);stop-opacity:1" />
      <stop offset="100%" style="stop-color:rgb(0,0,255);stop-opacity:1" />
    </linearGradient>
  </defs>
  <rect width="1024" height="1024" fill="url(#grad1)"/>
  <path d="M 100 100 L 200 200 L 300 100" stroke="black" fill="none"/>
  <!-- ... more elements -->
</svg>
```

### WebGL Canvas
Real-time rendering with 60 FPS audio reactivity, GPU-accelerated transforms

### Static PNG/JPG
Rendered at minting time, stored on IPFS for gallery preview

## Phase Implementation

### Phase 1: SVG Foundation (Week 1)
- Basic shape rendering (circles, rectangles, paths)
- Color extraction and gradients
- L-system tree generation
- IPFS integration for artwork storage
- Metadata generation

### Phase 2: Animation (Week 2)
- P5.js canvas animations
- Frame-based timing
- Interactive parameter sliders
- Export animated GIF

### Phase 3: Audio-Visual Sync (Week 3)
- Web Audio API integration
- Real-time frequency analysis
- WebGL shader development
- BPM detection and beat alignment
- Blue Dimension radio stream integration

### Phase 4: @®† Launch (Week 4+)
- Artist registry authentication
- Batch minting infrastructure
- Certificate generation
- Marketplace listing (OpenSea, Rarible)
- Dashboard for artists to view generated works

## Performance Targets

- SVG generation: < 100ms per artwork
- PNG render: < 500ms
- WebGL animation: 60 FPS with audio input
- Verification: < 50ms on-device
- Batch generation: 1000 artworks/minute

## Security Considerations

- PRNG must use cryptographically secure seed derivation
- Grammar expansion must terminate (bounded recursion depth)
- Rendering must handle edge cases (extreme feature values)
- Memory limits for large coordinate lists
- Prevents infinite loops in L-system generation
