
## The Agentic Librarian (Path 3)
[](#2-the-storage-pipeline---a-medallion-architecture)
[](#11-build-roadmap)
[](#license)

The agent proposes, the human admits, and the code enforces.
The wiki is a deliberately small, human-vetted ground truth that everything else is allowed to build on.

The Agentic Librarian prevents AI hallucinations and stale facts from infiltrating a knowledge graph by replacing fragile prompt instructions with a deterministic data-engineering and governance pipeline.
## 🏛️ Architecture Overview
The system uses a Medallion Storage Pipeline to ensure raw data undergoes strict review before becoming trusted ground truth:

* 
* 🟫 Bronze (Raw / Ingestion): Unmodified source materials (assumed dirty).
* 🥈 Silver (Review / Proposals): AI-generated candidate state updates and schema-conformant proposals.
* 🥇 Gold (Trusted Wiki): Human-vetted, highly-interlinked knowledge pages.
* 

The workflow separates deterministic code operations (preflight hashing, validation, routing, and git checkpointing) from bounded semantic drafting and human editorial gates.
## 📂 Vault Layout & Core Primitives
The repository organizes files into schema/, raw/ (Bronze), review/ (SILVER), and wiki/ (GOLD) directories, supported by an index.md, log.md, and home.md. Operational edits occur via Proposal Markdown Documents in review/ featuring detailed frontmatter tracking revision status, provenance, and connections.
## 🗺️ Build Roadmap & Acceptance Criteria
The implementation spans from Phase 0 (Foundations) through Phase 3 (Verification - Current Target) to final integration, requiring system impenetrability, environment sandbox permissions enforcement, and execution idempotence.
For the complete configuration, directory layout templates, and full markdown schemas, please refer to the documentation blocks in the referenced source materials.
