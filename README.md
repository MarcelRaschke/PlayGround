# PlayGround

Underground experimentation space for music production systems, cybersecurity research, computer vision, and production-grade infrastructure.

Built by cy8er & DJ Jesse Jay. Created with [OMGithub](https://omgithub.com).

---

## Vision

PlayGround is a unified repository bridging creative (music/visual) and technical (security/infrastructure) expertise. It serves as:

- A DJ production platform (Rekordbox, Ableton, TouchDesigner)
- A cybersecurity research lab (OSINT, threat intelligence)
- A computer vision experimentation ground
- A cloud infrastructure reference (Kubernetes, ArgoCD, Terraform)
- A local LLM and knowledge graph sandbox

## Structure

| Directory | Purpose | Status |
|-----------|---------|--------|
| **audio** | DJ systems, music production, Blue Dimension radio | Scaffolded |
| **vision** | Computer vision, audio-visual sync, object detection | Scaffolded |
| **infra** | Kubernetes, Terraform, ArgoCD, observability | Scaffolded |
| **security** | OSINT, threat intelligence, attack surface | Scaffolded |
| **ai** | Local LLMs, knowledge graphs, intelligent agents | Scaffolded |
| **docs** | Architecture, runbooks, threat models, ADRs | Scaffolded |

## Quick Start

### Prerequisites

- Git with LFS support (optional but recommended)
- Docker & Docker Compose
- Kubernetes CLI tools (kubectl, helm)
- Python 3.11+ with poetry/uv

### Setup

```bash
# Clone repository
git clone https://github.com/MarcelRaschke/playground
cd playground

# Initialize development environment
./scripts/init-dev.sh

# Install dependencies per module
cd audio && pip install -r requirements.txt
cd ../vision && pip install -r requirements.txt
# ... repeat for other modules
```

## Key Principles

**Production-Ready:** All code is documented, tested, and ready for real-world use.

**Security First:** Non-obvious usernames, authenticated proxies, TLS everywhere, zero-trust.

**Local-First:** Offline-capable systems (llama.cpp, local databases, edge computing).

**Compliance-Conscious:** EU AI Act, GDPR/revDSG, responsible AI, audit trails.

**Transparent Documentation:** Architecture decisions, threat models, runbooks, and ADRs.

## Technology Stack

### Audio & Music
- **Rekordbox/CDJ**: QEMU emulation, OSC control
- **Ableton Live**: Max for Live, NDI integration
- **Audio Processing**: librosa, essentia, pydub
- **Streaming**: Icecast, RTMP, HLS

### Infrastructure
- **Orchestration**: Kubernetes, Helm
- **Infrastructure-as-Code**: Terraform, Pulumi
- **GitOps**: ArgoCD, Flux CD
- **Observability**: Prometheus, Loki, Jaeger
- **Networking**: Istio service mesh, Traefik ingress

### Security & Research
- **OSINT**: Shodan, Censys, Spiderfoot
- **Scanning**: Nuclei, nmap, metasploit
- **Analysis**: Ghidra, IDA Pro, Semgrep
- **Threat Intel**: Custom correlation pipelines

### AI & ML
- **Local LLMs**: llama.cpp, Ollama, vLLM
- **Vision**: YOLO, MediaPipe, OpenCV
- **Knowledge**: Neo4j, RDF stores
- **Experimentation**: MLflow, Weights & Biases

## Entry Points by Specialty

### DJ / Music Production
Start with `audio/` and check:
- Rekordbox integration patterns
- Ableton Live Max for Live examples
- Blue Dimension radio automation

### Security / OSINT Researcher
Start with `security/` and explore:
- OSINT data collection framework
- Threat actor profiling tools
- Vulnerability intelligence pipelines

### Infrastructure / DevOps Engineer
Start with `infra/` and review:
- Kubernetes cluster configurations
- Terraform modules for common patterns
- ArgoCD deployment examples

### AI / ML Researcher
Start with `ai/` and experiment with:
- Local LLM setups and fine-tuning
- Knowledge graph construction
- Intelligent agent implementations

### Visual / Computer Vision Artist
Start with `vision/` and build:
- Audio-reactive visualizations
- Real-time object detection
- Cymatic pattern generation

## Operational Guidelines

### Development
- Use feature branches: `feature/description`
- Write clear commit messages and ADRs
- Include tests and documentation
- Run linters and security scanners (Semgrep)

### Security
- Rotate credentials regularly (sealed-secrets)
- No public IPFS APIs (authenticated reverse proxies)
- Vulnerability scans on every push (Trivy)
- Regular penetration testing of infrastructure

### AI/ML Ethics
- High-risk models require EU AI Act compliance documentation
- Model cards with intended use, limitations, bias assessment
- Regular audits for harmful outputs
- Data provenance tracking and consent management

### Deployment
- GitOps via ArgoCD (all infrastructure in version control)
- Blue-green deployments, canary releases
- Automated rollback on detection failures
- Infrastructure-as-Code changes peer-reviewed

## Links

- 🎮 **Open the project:** [OMGithub PlayGround](https://omgithub.com/MarcelRaschke/PlayGround)
- 🎙️ **Blue Dimension Radio:** [LoRa Zürich](https://www.lora.ch) — Thursdays 00:00–06:00 UTC
- 🎵 **DJ Jesse Jay:** [djjessejay.ch](https://djjessejay.ch)
- 🔒 **cy8er Security:** OSINT & threat research
- 💻 **Infrastructure:** ArgoCD clusters (private)

## Contributing

This is a personal laboratory. External contributions welcome via:
- Bug reports and security disclosures
- Feature requests with use cases
- Documentation improvements
- Code reviews via pull requests

Please respect the security posture and compliance requirements listed above.

## License

See [Lizenz](./Lizenz) file.

---

**Last Updated:** 2026-09-29  
**Maintained by:** cy8er & DJ Jesse Jay  
**Status:** Active Development

Generated with [Claude Code](https://claude.com/claude-code)
