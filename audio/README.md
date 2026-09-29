# Audio / DJ Systems

DJ performance infrastructure, music production integration, and Blue Dimension radio broadcast systems.

## Components

### Rekordbox / CDJ Integration
- QEMU CDJ emulation for local testing
- Rekordbox database sync protocols
- XDJ/CDJ-3000 control surface mapping
- Rekordbox OSC endpoints for Ableton/TouchDesigner

### Track Analysis
- Local BPM/key detection (llama.cpp based)
- Audio feature extraction (librosa, essentia)
- Beatgrid generation and analysis
- Metadata enrichment pipeline

### Blue Dimension (Radio LoRa)
- Broadcast schedule and automation
- Live stream encoding (Icecast, RTMP)
- Archive management and indexing
- Listener statistics and analytics

### Ableton Live Integration
- Max for Live device definitions
- OSC control protocols
- NDI video integration for visual sync
- Live performance scripts and presets

## Infrastructure

Local development requires:
- Docker with QEMU support for CDJ emulation
- Chromaprint library for acoustic fingerprinting
- Audio interface with low-latency drivers
- JACK or PulseAudio for system-wide routing

## Production Setup

See `/infra/audio/` for Kubernetes manifests:
- Icecast streaming servers
- Audio processing workers
- Database clusters for metadata
- monitoring and alerting

---
Maintained by cy8er / DJ Jesse Jay
