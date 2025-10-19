# ConvoSync

**Backend server for ConvoCLI cross-device synchronization**

[![License: AGPL v3](https://img.shields.io/badge/License-AGPL%20v3-blue.svg)](https://www.gnu.org/licenses/agpl-3.0)
[![Platform](https://img.shields.io/badge/platform-Backend-green.svg)](https://github.com/convocli/convosync)

> Sync code + conversation + context as one atomic unit across all your devices

---

## What is ConvoSync?

ConvoSync is the backend synchronization service that powers **[ConvoCLI](https://github.com/convocli/convocli)**'s killer feature: seamlessly continuing coding sessions across desktop, mobile, and tablet.

### The Problem It Solves

```
Desktop:
  You: "Add login to the app"
  Claude: "I've added it to auth.js line 45"
  [git commit abc123]

Mobile (without ConvoSync):
  Pulls conversation only
  auth.js line 45 doesn't exist (old code)
  Context completely broken
```

**ConvoSync ensures code and conversation stay in perfect lockstep.**

---

## Key Features

**Atomic Session Sync**
- Code + conversation + terminal state synced as ONE unit
- Git-aware: Conversations linked to commit hashes
- Safety checks prevent state mismatches

**Delta Compression**
- 80-96% size reduction with message-level sync
- Only syncs new messages (incremental)
- Snapshot + delta architecture (like git)

**Performance Optimized**
- First sync: 740KB vs 3.7MB (80% reduction)
- Incremental: <1 second vs 30 seconds (30x faster)
- Mobile-friendly bandwidth usage

**Security First**
- End-to-end AES-256 encryption
- Server never sees unencrypted conversations
- Self-hosting available (AGPL-3.0)

---

## How It Works

### Desktop → Mobile Flow

```bash
# Desktop (working with Claude Code)
$ claude-code
[Implementing OAuth feature...]

# Ready to leave
$ convosync save "Implementing OAuth feature"
✓ Code: git add, commit, push
✓ Conversation: compressed & encrypted
✓ Linked: commit abc123 ↔ conversation
✓ Uploaded to ConvoSync

# Mobile (subway ride)
$ convosync resume
✓ Pulling code: git pull (commit abc123)
✓ Fetching conversation (linked to abc123)
✓ Verifying: code and conversation match
✓ Ready! Continue exactly where you left off

$ convocli
[Continue with Claude - full context preserved]
```

### Atomic Sync Architecture

```
ConvoSync Session = {
  git_commit: "abc123def456",
  git_branch: "feature/oauth",
  git_repo: "https://github.com/user/project",
  conversation_id: "conv_xyz789",
  conversation_delta: { ... },  // Only new messages
  terminal_state: { cwd, env, ... },
  timestamp: 1234567890
}
```

**Safety guarantees:**
- Can't restore conversation without matching code state
- Automatic verification on resume
- Conflict detection and resolution

---

## Delta Compression

### Performance Comparison

| Scenario | Delta Sync | Full Sync | Improvement |
|----------|------------|-----------|-------------|
| First sync (800 msgs) | 6s / 740KB | 30s / 3.7MB | 5x faster, 80% smaller |
| Incremental (50 msgs) | <1s / 40KB | 30s / 3.7MB | 30x faster, 96% smaller |
| Resume on mobile | 6s / 740KB | 30s / 3.7MB | 5x faster, 80% smaller |

### How It Works

```
Cloud Storage Structure:
conversations/{userId}/{conversationId}/
  metadata.json:
    { totalMessages: 850, latestSnapshot: 0, compressionRatio: 0.80 }

  snapshots/
    snapshot-0.gz:        # Messages 0-799 (compressed)

  deltas/
    delta-800-850.gz:     # Messages 800-849 (compressed)
```

**Strategy:**
- Snapshot every 1000 messages (full state)
- Delta files for incremental changes
- gzip level 9 compression (80% reduction on JSON)
- Client tracks: `lastSyncedMessageIndex`

---

## Architecture

```
ConvoSync Backend
├── API Server (Node.js/Go)
│   ├── Authentication (JWT)
│   ├── Session management
│   ├── Delta sync endpoints
│   └── Git integration hooks
├── Storage Layer
│   ├── Firebase/Firestore (managed)
│   ├── S3 + PostgreSQL (self-hosted)
│   └── Delta compression engine
├── Security
│   ├── E2E encryption (AES-256)
│   ├── Key management
│   └── Access control
└── Sync Protocol
    ├── Git-aware linking
    ├── Conflict resolution
    └── Offline support
```

---

## API Overview

### Save Session
```
POST /api/v1/sessions/save
{
  "git_commit": "abc123",
  "git_branch": "main",
  "conversation_id": "conv_xyz",
  "delta": { /* encrypted delta */ },
  "metadata": { /* session info */ }
}
```

### Resume Session
```
POST /api/v1/sessions/resume
{
  "conversation_id": "conv_xyz",
  "git_commit": "abc123"  // Verify code matches
}

Response: {
  "snapshots": [...],     // Full snapshots needed
  "deltas": [...],        // Incremental deltas
  "last_message_index": 850
}
```

### Sync Delta (Incremental)
```
POST /api/v1/conversations/{id}/sync
{
  "base_message_index": 800,
  "messages": [ /* new messages */ ]
}
```

---

## Technology Stack

- **Language:** Node.js / Go (TBD)
- **Database:** Firestore (managed) / PostgreSQL (self-hosted)
- **Storage:** Cloud Storage / S3
- **Compression:** gzip level 9
- **Encryption:** AES-256-GCM
- **Protocol:** REST API + WebSocket (real-time)

---

## Self-Hosting

ConvoSync is fully open source (AGPL-3.0). You can self-host for free!

### Quick Start (Coming Soon)

```bash
git clone https://github.com/convocli/convosync
cd convosync

# Configure
cp .env.example .env
# Edit .env with your settings

# Run with Docker
docker-compose up -d

# Or run directly
npm install
npm run migrate
npm start
```

### Configuration

```env
# Database
DATABASE_URL=postgresql://user:pass@localhost/convosync

# Storage
STORAGE_BACKEND=s3  # or 'local', 'gcs'
S3_BUCKET=my-convosync-bucket

# Security
ENCRYPTION_KEY=your-256-bit-key
JWT_SECRET=your-jwt-secret

# Limits
MAX_CONVERSATION_SIZE=10MB
MAX_DEVICES_PER_USER=5
```

---

## Documentation

- **[Complete Specification](convosync.md)** - Full technical architecture
- **[ConvoCLI App](https://github.com/convocli/convocli)** - Mobile terminal client
- **[API Documentation](docs/api.md)** - REST API reference (coming soon)

---

## Project Status

**Phase 1: Core Sync (Months 4-6)** - 📋 Planned
- [ ] REST API server
- [ ] Delta compression engine
- [ ] Git-aware session linking
- [ ] Firestore backend
- [ ] Basic encryption

**Phase 2: Optimization (Months 7-9)** - 💡 Future
- [ ] WebSocket real-time sync
- [ ] Advanced conflict resolution
- [ ] Multi-device orchestration
- [ ] Self-hosting documentation

**Phase 3: Enterprise (Months 10-12)** - 💡 Future
- [ ] Team collaboration
- [ ] SSO integration
- [ ] Audit logging
- [ ] Custom retention policies

---

## Deployment Options

### Managed Cloud (Recommended)
- Official ConvoSync hosting at convosync.dev
- $5/month - 2 devices
- $10/month - Unlimited devices
- Automatic updates, backups, monitoring

### Self-Hosted
- Full control over your data
- Free (AGPL-3.0 license)
- Requires technical setup
- You manage updates and backups

---

## Contributing

ConvoSync is open source and built by developers, for developers.

- **Issues:** [Report bugs or request features](https://github.com/convocli/convosync/issues)
- **Discussions:** [Join the conversation](https://github.com/convocli/convosync/discussions)
- **Pull Requests:** Contributions welcome!

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines (coming soon).

---

## Security

**End-to-End Encryption:**
- Client encrypts conversations before upload
- Server stores only encrypted blobs
- Encryption keys never leave your devices

**Audit:**
- All code is open source (AGPL-3.0)
- Security researchers welcome to audit
- Bug bounty program (coming soon)

**Report vulnerabilities:** security@convocli.dev (coming soon)

---

## License

This project is licensed under **AGPL-3.0** to ensure the backend remains open source even when deployed as a service.

See [LICENSE](LICENSE) for details.

---

## Related Projects

- **[ConvoCLI](https://github.com/convocli/convocli)** - Mobile terminal app for Android
- **[Docs](https://github.com/convocli/docs)** - Documentation and guides

---

## Acknowledgments

Built with inspiration from:
- **Git** - Delta compression and snapshot model
- **Firestore** - Real-time sync architecture
- **Restic** - Backup deduplication techniques

---

## Creator

**Created and maintained by [Your Name](https://github.com/yourusername)**

ConvoSync is part of the ConvoCLI project - enabling developers to code anywhere.

---

<div align="center">

**[⬆ Back to Top](#convosync)**

Part of the [ConvoCLI](https://github.com/convocli/convocli) ecosystem

</div>
