# Smart Contract AST Analysis Specification

## Overview

Smart contract AST analysis is a security-focused approach to parsing Ethereum bytecode and Solidity source code, enabling automated detection of vulnerabilities, measurement of code quality metrics, and generation of interactive audit reports. This specification defines the architecture for a production-ready analysis engine suitable for NFT smart contracts, artist registries, and provenance tracking systems.

### Objectives

1. **Vulnerability Detection**: Identify high-severity and medium-severity security flaws in contract code
2. **Code Quality Metrics**: Measure cyclomatic complexity, gas efficiency, inheritance depth, and test coverage
3. **Dependency Mapping**: Generate graphs of contract dependencies and external calls
4. **Audit Trail**: Create immutable records of contract analysis history
5. **Report Generation**: Produce interactive HTML reports and JSON structured outputs
6. **CI/CD Integration**: Integrate analysis into GitHub Actions and Etherscan API workflows

## AST Parsing Strategy

### Three-Level AST Decomposition

#### Level 1: Structural Analysis
Parse contract structure into normalized AST node types:

```python
class ContractNode:
    name: str
    source_file: str
    inheritance: List[str]
    state_variables: List[StateVariable]
    functions: List[FunctionNode]
    modifiers: List[ModifierNode]
    events: List[EventNode]
    
class StateVariable:
    name: str
    type: str
    visibility: str  # public, private, internal, external
    mutability: str  # constant, immutable
    initial_value: Optional[str]

class FunctionNode:
    name: str
    parameters: List[Parameter]
    return_types: List[str]
    visibility: str
    mutability: str  # pure, view, payable
    modifiers_applied: List[str]
    body: List[Statement]
    
class ModifierNode:
    name: str
    parameters: List[Parameter]
    body: List[Statement]
```

#### Level 2: Control Flow Analysis
Build control flow graphs (CFGs) for each function:

```
Entry Node
    ↓
Condition 1 → True Branch → Statement A → Exit
    ↓
   False Branch → Loop Header → Statement B → Condition 2
                                   ↓
                            Loop Body → Back Edge
```

Track:
- Reachability: Which statements are reachable from entry?
- Loops: Identify loop structures, nesting depth
- Branches: Decision points, path complexity
- Exits: Return statements, reverts, throws

#### Level 3: Data Flow Analysis
Trace variable definitions and uses:

```python
class DataFlowEdge:
    source_variable: str
    source_statement: int  # line number
    target_statement: int
    operation: str  # read, write, pass-through
```

This enables detection of:
- Uninitialized variable reads
- Use-after-free patterns
- Tainted data flow from untrusted sources
- Integer overflow propagation paths

## Vulnerability Detection Rules

### High-Severity Patterns

#### 1. Reentrancy Vulnerability
**Pattern**: External call before state update within the same transaction

```solidity
// VULNERABLE
function withdraw() external {
    uint256 amount = balances[msg.sender];
    (bool success, ) = msg.sender.call{value: amount}("");  // External call
    require(success);
    balances[msg.sender] = 0;  // State update AFTER call
}
```

**Detection Rule**:
- Locate external call (`call`, `delegatecall`, `staticcall`, or known external function invocation)
- Check if any state variable is modified after the call in the same function
- Flag as HIGH if caller can influence control flow (e.g., fallback function)

**Mitigation**: Checks-Effects-Interactions pattern (state updates before calls)

#### 2. Integer Overflow/Underflow
**Pattern**: Arithmetic operations without bounds checking

```solidity
// VULNERABLE
function transfer(address to, uint256 amount) external {
    balances[msg.sender] -= amount;  // Underflow if amount > balance
    balances[to] += amount;  // Overflow if to's balance is MAX_UINT
}
```

**Detection Rule**:
- Track all arithmetic operations (`+`, `-`, `*`, `/`, `%`, `**`)
- Check for preceding `require()` or `assert()` statements validating operands
- Flag MEDIUM if Solidity version < 0.8.0 (automatic overflow checks in 0.8+)
- Flag HIGH if overflow can be triggered by user input

**Mitigation**: Solidity 0.8.0+, OpenZeppelin SafeMath, or explicit checks

#### 3. Unchecked External Call
**Pattern**: External call result not verified

```solidity
// VULNERABLE
function approveToken(address token) external {
    IERC20(token).approve(spender, amount);  // Return value ignored
}
```

**Detection Rule**:
- Identify all external calls (address.call(), interface function calls)
- Check if return value is assigned or checked
- Flag MEDIUM if return value is ignored
- Flag HIGH if ignored return indicates execution failure

#### 4. Access Control Flaw
**Pattern**: Missing or inadequate permission checks

```solidity
// VULNERABLE
function withdrawAdmin(uint256 amount) external {
    // Missing onlyOwner check
    msg.sender.call{value: amount}("");
}
```

**Detection Rule**:
- Identify sensitive functions (transfer, burn, mint, admin operations)
- Check for `require(msg.sender == owner)` or similar modifier
- Flag HIGH if sensitive function has no access control
- Flag MEDIUM if access control is in comment only

#### 5. Delegatecall to Untrusted Contract
**Pattern**: Dynamic delegatecall without validation

```solidity
// VULNERABLE
function execute(address target, bytes calldata data) external {
    (bool success, ) = target.delegatecall(data);  // target not validated
}
```

**Detection Rule**:
- Locate all delegatecall operations
- Check if target address is hardcoded or validated (whitelist)
- Flag HIGH if target is user-provided without checks
- Flag MEDIUM if target is from storage without access control

### Medium-Severity Patterns

#### 6. Timestamp Dependence
```solidity
// MEDIUM RISK
if (now > deadline) { ... }  // now is controllable by miner
```

**Detection**: Use of `block.timestamp` in critical logic

#### 7. Front-Running Vulnerability
```solidity
// MEDIUM RISK
function setPrice(uint256 newPrice) external {
    price = newPrice;  // Mempool observable, vulnerable to front-running
}
```

**Detection**: State modification that affects transaction ordering

#### 8. Gas Limit Denial of Service
```solidity
// MEDIUM RISK
function batchTransfer(address[] calldata recipients) external {
    for (uint i = 0; i < recipients.length; i++) {
        transfer(recipients[i], amount);  // No gas limit check
    }
}
```

**Detection**: Unbounded loops, especially in external functions

## Code Quality Metrics

### Cyclomatic Complexity (CC)
Measures the number of independent paths through function code.

```
CC = E - N + 2P
where E = edges, N = nodes, P = connected components

CC 1-4:    Low complexity (green)
CC 5-7:    Moderate complexity (yellow)  
CC 8-10:   High complexity (orange)
CC > 10:   Very high complexity (red) - refactor recommended
```

**Implementation**:
```python
def calculate_cyclomatic_complexity(cfg):
    """
    cfg: ControlFlowGraph with nodes and edges
    """
    edges = len(cfg.edges)
    nodes = len(cfg.nodes)
    components = count_connected_components(cfg)
    return edges - nodes + 2 * components
```

### Gas Analysis
Estimate gas consumption per function:

- Opcode cost table (PUSH1=3 gas, SSTORE=20000 gas, etc.)
- Loop unrolling analysis for fixed-size iterations
- Cold storage access (2100 gas) vs warm access (100 gas)
- External call overhead (21000 gas + dynamic costs)

**Risk Levels**:
- < 50,000 gas: Low
- 50k-100k: Moderate
- 100k-500k: High
- > 500k: Very High (consider splitting function)

### Inheritance Depth
```
Depth 1: Contract (no inheritance)
Depth 2: Contract inherits from 1 base
Depth 3+: Multiple levels (caution - complexity increases)
```

**Recommendation**: Limit to depth 3, prefer composition over inheritance

### Test Coverage
Parse hardhat/truffle test files to map coverage:

```python
class CoverageMetric:
    total_statements: int
    covered_statements: int
    percentage: float
    uncovered_lines: List[int]
```

**Targets**:
- Critical contracts: > 95% coverage
- Standard contracts: > 80% coverage
- Utility contracts: > 70% coverage

## Dependency Graph

Generate directed graph of contract dependencies:

```
ArtistRegistry
    ├─→ ERC721 (OpenZeppelin)
    ├─→ Ownable (OpenZeppelin)
    └─→ SafeMath (internal import)

ProvenanceNFT
    ├─→ ERC721 (OpenZeppelin)
    ├─→ ArtistRegistry (internal)
    └─→ Merkle (internal library)
```

**Output Format** (GraphML for visualization):
```xml
<?xml version="1.0"?>
<graphml>
  <graph>
    <node id="ArtistRegistry"/>
    <node id="ERC721"/>
    <edge source="ArtistRegistry" target="ERC721" label="inherits"/>
  </graph>
</graphml>
```

## Report Format

### JSON Output Structure

```json
{
  "analysis": {
    "contract": "ArtistRegistry",
    "source_file": "contracts/ArtistRegistry.sol",
    "timestamp": "2026-09-29T03:00:00Z",
    "compiler_version": "0.8.19"
  },
  "vulnerabilities": [
    {
      "id": "reentrancy-001",
      "severity": "HIGH",
      "function": "withdraw",
      "line": 45,
      "description": "Reentrancy vulnerability detected",
      "code_snippet": "function withdraw() external { ... }",
      "remediation": "Apply checks-effects-interactions pattern",
      "cwe": "CWE-246"
    }
  ],
  "metrics": {
    "cyclomatic_complexity": 8,
    "gas_estimate": 120000,
    "inheritance_depth": 2,
    "test_coverage_percent": 85.5
  },
  "dependencies": [
    {
      "name": "ERC721",
      "type": "external",
      "source": "openzeppelin-contracts/ERC721.sol"
    }
  ]
}
```

### Interactive HTML Report

Structure:
- Executive Summary (vulnerabilities count, metrics overview)
- Vulnerability List with code highlighting and drill-down
- Metrics Dashboard (charts, gauges, trend indicators)
- Dependency Graph (interactive visualization)
- Coverage Report (line-by-line annotation)
- Attestation Section (analysis timestamp, analyzer version, confidence scores)

## Tools & Implementation

### Python Stack (Recommended)

**Primary Tools**:
- `slither`: Abstract syntax tree analysis engine (ConsenSys)
- `mythril`: Symbolic execution for vulnerability detection
- `echidna`: Fuzzing-based property testing
- `tree-sitter`: Language-agnostic parser

**Integration**:
```bash
pip install slither-analyzer mythril echidna tree-sitter tree-sitter-solidity
```

**Wrapper Implementation**:
```python
import slither
from mythril.mythril import Mythril
from echidna.runner import EchidnaRunner

class SmartContractAnalyzer:
    def __init__(self, contract_path: str):
        self.slither = slither.Slither(contract_path)
        self.mythril = Mythril()
        self.echidna = EchidnaRunner()
    
    def run_analysis(self) -> AnalysisReport:
        vulnerabilities = self.slither.run_detectors()
        symbolic_results = self.mythril.analyze(self.contract_path)
        fuzz_results = self.echidna.run(self.contract_path)
        return self.aggregate_results(vulnerabilities, symbolic_results, fuzz_results)
```

### Rust Alternative

**Benefits**: Performance, memory safety, native binary compilation

**Stack**:
- `tree-sitter-solidity`: Parser
- `ethers-rs`: Blockchain interaction
- `proptest`: Property-based testing

```toml
[dependencies]
tree-sitter = "0.20"
tree-sitter-solidity = "0.19"
ethers = "2.0"
proptest = "1.0"
```

## Integration Points

### Etherscan API Integration
```python
class EtherscanIntegration:
    def fetch_contract_source(self, address: str) -> str:
        """Retrieve contract source from Etherscan"""
        response = requests.get(
            f"https://api.etherscan.io/api?module=contract&action=getsourcecode&address={address}"
        )
        return response.json()[0]['SourceCode']
    
    def submit_analysis_result(self, address: str, report: AnalysisReport):
        """Submit analysis results to contract comments"""
        # Store on IPFS, reference from contract events
        pass
```

### GitHub Actions CI/CD
```yaml
name: Smart Contract Analysis
on: [push, pull_request]
jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run AST Analysis
        run: |
          pip install slither-analyzer
          slither contracts/ --json analysis.json
      - name: Upload Report
        run: |
          ipfs add analysis.json
```

### Automated Testing Pipeline
```python
class ContinuousAnalysis:
    def analyze_contract_on_commit(self, commit_hash: str):
        """Run analysis on every contract commit"""
        contract_files = self.git.get_changed_files(commit_hash, pattern="*.sol")
        for file in contract_files:
            report = self.analyzer.analyze(file)
            self.store_report(commit_hash, report)
            if report.has_high_severity_issues():
                self.notify_team(f"HIGH SEVERITY: {file}")
```

## Phase 1 Implementation Plan

### Week 1-2: Foundation

**Parser Wrapper** (Days 1-3)
- Integrate slither, mythril, echidna
- Normalize outputs to internal AST format
- Build three-level decomposition pipeline
- Create unit tests for parser

**Rule Engine** (Days 4-7)
- Implement vulnerability detection rules (high + medium severity)
- Create rule registry with enabled/disabled toggles
- Performance optimization for large contracts
- Test against known vulnerable contracts (etherscan samples)

**Report Generator** (Days 8-10)
- JSON output formatter
- HTML template rendering
- Interactive visualization setup
- PDF export capability

**CLI Tool** (Days 11-14)
- Command-line interface: `ast-analyze contracts/ --format html`
- Configuration file support (yaml/toml)
- Batch processing of multiple contracts
- Progress reporting and logging

### Integration Testing (Day 15)
- End-to-end testing with sample NFT contracts
- Performance benchmarking (contracts/second)
- Accuracy validation against known vulnerabilities
- Documentation and README

## Performance Targets

- **Analysis Speed**: < 5 seconds for 500-line contract
- **Memory Usage**: < 512 MB for large contracts
- **Report Generation**: < 1 second HTML generation
- **Batch Processing**: 100 contracts in < 5 minutes

## Security Considerations

- **Analyzer Compromise**: Use only official tool releases, verify checksums
- **Input Validation**: Sanitize contract paths, prevent directory traversal
- **Output Leakage**: Redact sensitive information (API keys, deployment strategies)
- **Supply Chain**: Regular updates to detection rules, vulnerability databases

## Success Criteria

1. Detects 95%+ of known vulnerabilities in test suite
2. < 10% false positive rate on clean contracts
3. All high-severity issues caught before deployment
4. CI/CD integration working without false failures
5. HTML reports provide actionable remediation guidance
6. Performance acceptable for production-scale analysis
