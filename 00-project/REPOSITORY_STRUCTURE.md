# SecondOrder.fun Repository Structure

## Overview
This document describes the organization of the SecondOrder.fun repository.

## Directory Structure

```
sof-alpha/
├── .windsurf/              # Windsurf AI assistant rules and memories
│   └── rules/              # Project-specific AI rules (DO NOT MOVE)
├── backend/                # Backend API server (Fastify)
│   └── src/
│       ├── routes/         # API route handlers
│       ├── services/       # Business logic and blockchain listeners
│       └── db/             # Database queries (TursoDB)
├── contracts/              # Smart contracts (Foundry/Solidity)
│   ├── src/                # Contract source files
│   ├── script/             # Deployment scripts
│   ├── test/               # Contract tests
│   └── lib/                # External dependencies (DO NOT MODIFY)
├── docs/                   # Project documentation
│   ├── 00-project/         # Project overview and requirements
│   ├── 01-planning/        # Development plans
│   ├── 02-architecture/    # System architecture
│   ├── 03-development/     # Development guides and fixes
│   ├── 04-audits/          # Security audits
│   ├── 05-features/        # Feature documentation
│   ├── 06-technical-analysis/  # Research and analysis
│   └── 07-changelog/       # Version history
├── instructions/           # AI assistant instructions (DO NOT MOVE)
├── scripts/                # Utility scripts
├── src/                    # Frontend source (React/Vite)
│   ├── components/         # React components
│   ├── contracts/          # Contract ABIs and addresses
│   ├── hooks/              # Custom React hooks
│   ├── pages/              # Page components
│   ├── services/           # API and blockchain services
│   └── utils/              # Utility functions
└── public/                 # Static assets

## Key Files

- **README.md** - Project overview and setup instructions
- **.env** - Environment variables (DO NOT COMMIT)
- **.env.example** - Environment variable template
- **package.json** - Node.js dependencies and scripts
- **foundry.toml** - Foundry configuration
- **vite.config.js** - Vite build configuration

## Documentation Organization

### docs/00-project/
Project-level documentation:
- `project-framework.md` - Product vision and philosophy
- `REPOSITORY_STRUCTURE.md` - This file

### docs/03-development/
Development-specific documentation:
- `FIXES_APPLIED.md` - Bug fixes and solutions
- `FPMM_MARKET_CREATION_FIX.md` - InfoFi market creation fix
- `infofi-integration/` - InfoFi integration specifications

### docs/04-audits/
Security audits and reports:
- `AUDIT_REPORT_INFOFI_MARKET_CREATION.md` - InfoFi market creation audit

## Important Notes

### DO NOT MOVE
These directories contain configuration that should stay in place:
- `.windsurf/rules/` - AI assistant rules
- `instructions/` - AI assistant instructions
- `contracts/lib/` - External contract dependencies

### Temporary Files
The following are excluded from git and can be safely deleted:
- `node_modules/` - Node.js dependencies
- `dist/` - Build output
- `.DS_Store` - macOS metadata
- `*.log` - Log files
- `broadcast/` - Forge deployment artifacts
- `cache/` - Forge cache

## Cleanup Commands

```bash
# Remove temporary files
npm run clean

# Kill zombie processes
npm run kill:zombies

# Full cleanup and rebuild
npm run clean && npm install && npm run build
```

## Development Workflow

1. **Start Anvil**: `npm run anvil` or `npm run anvil:deploy`
2. **Start Backend**: `npm run dev:backend`
3. **Start Frontend**: `npm run dev:frontend`
4. **Deploy Contracts**: `cd contracts && forge script script/Deploy.s.sol --broadcast`

## Documentation Updates

When adding new documentation:
1. Place in appropriate `docs/XX-category/` directory
2. Update this file if adding new categories
3. Link from `docs/README.md` if it's a key document
4. Use descriptive filenames with hyphens (e.g., `feature-name-guide.md`)

---

Last updated: October 23, 2025
