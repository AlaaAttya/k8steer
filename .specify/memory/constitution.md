<!--
Sync Impact Report
- Version change: template (unratified) → 1.0.0
- Modified principles: all placeholders replaced with 7 ratified principles
- Added sections: Project Purpose, Target Users, Scope Boundaries, Technical Standards,
  Security Requirements, Decision-Making Rule
- Removed sections: none (template expanded)
- Templates requiring updates:
  - .specify/templates/plan-template.md ✅ (this pass)
  - .specify/templates/spec-template.md ✅ (this pass)
  - .specify/templates/tasks-template.md ⚠ pending (no constitution-specific placeholders)
  - README.md ✅ (this pass)
- Follow-up TODOs: trademark search before public launch
-->

# k8steer Constitution

## Project Purpose

k8steer is a **native macOS desktop application** for managing Kubernetes clusters.

Existing tools are either Electron-heavy (Lens), terminal-first (k9s), or native but not
optimized for **first-party macOS UX** (Krust, Rune, Kamera). k8steer exists to be the
Kubernetes app Mac engineers reach for because it feels like it shipped with macOS — not
like a website in a window.

The primary goal is to become the default daily driver for Kubernetes engineers who live on
Macs: the tool they open the moment they need to understand or act on a cluster.

## Core Principles

### I. Native First

- MUST be built with Swift 6+, SwiftUI (primary), and AppKit where SwiftUI lacks capability
  (large virtualized lists, terminal embedding, drag-and-drop).
- MUST NOT use Electron, Tauri, or an embedded browser runtime for core UI.
- Read-only external documentation MAY open in `SFSafariViewController`; feature UI MUST NOT
  be built in WebViews.
- Target feel: Finder, Xcode, Activity Monitor — first-party Apple polish, not cross-platform
  compromise.

**Rationale**: Native UI is the product thesis. A web runtime defeats performance, privacy,
and platform integration goals.

### II. Local-First and Private

- MUST NOT require accounts, cloud services, telemetry, or analytics of any kind.
- MUST keep all application data on the user's machine.
- Credentials and kubeconfigs MUST NOT leave the device except when communicating directly
  with the Kubernetes API server.
- An optional local diagnostic log file MAY exist; it MUST be user-controlled and off by
  default. No outbound analytics endpoints.

**Rationale**: Cluster credentials are sensitive. Engineers must trust that the app is not
a data exfiltration surface.

### III. Performance is a Feature

Performance targets are non-negotiable and MUST be measurable:

| Metric | Target | Measurement |
|--------|--------|-------------|
| Cold start (Apple Silicon, Release) | < 1.0 s | XCTest launch metric, M-series baseline |
| Idle memory (1 cluster, ~500 pods) | < 150 MB | Instruments snapshot |
| Scroll FPS (5,000-row pod list) | 60 fps sustained | Instruments Core Animation |
| Watch reconnect after API blip | < 3 s | Integration test against kind cluster |

**Rationale**: Sluggish cluster tools get abandoned. Performance is a retention feature.

### IV. Keyboard-First, Mouse-Friendly

- Every important action MUST be reachable via keyboard (command palette + comprehensive
  shortcuts).
- The command palette (`⌘K`) is a **v1.0 gate**, not a nice-to-have.
- Every destructive action MUST have a keyboard-accessible confirmation path.
- Mouse users MUST never feel second-class.

**Rationale**: Power users live on the keyboard; the app must not slow them down.

### V. Honest and Transparent

- MUST NOT hide what the app is doing from the user.
- MUST show the exact `kubectl`-equivalent command when relevant.
- For destructive operations, the equivalent command MUST be shown before apply/delete when
  user preference is enabled (default: on for destructive ops).
- When correctness and convenience conflict, MUST prefer correctness.

**Rationale**: Engineers distrust magic. Transparency builds trust and aids learning.

### VI. Respect the Platform (Primary Differentiator)

This principle is k8steer's competitive moat. The app MUST integrate deeply with macOS:

- Multi-window support, per-cluster windows, and Stage Manager scenes
- macOS Shortcuts actions for common operations (get pods, tail logs, switch context)
- Keychain storage for kubeconfig credentials and exec auth tokens
- Full VoiceOver, Dynamic Type, and reduced-motion support
- Native `NSUndoManager` for YAML edits
- Dark Mode and system appearance adaptation

Deferred to v1.x (not v1.0 blockers): Widgets, Menu Bar extra, Shortcuts catalog expansion.

**Rationale**: Other native K8s apps exist; k8steer wins on Apple platform depth, not
"native vs Electron" alone.

### VII. Pure Swift Stack

- All Kubernetes API communication MUST go through a Swift client library (primary:
  [Swiftkube/client](https://github.com/swiftkube/client)).
- `kubectl` subprocess is **forbidden** for core read, write, and watch operations.
- Allowed `kubectl`/`helm` subprocess uses ONLY:
  - User-initiated "run in terminal" escape hatch
  - Features with no stable Swift API yet (each exception MUST be documented in
    `research.md` for that feature)
- Rust, C++, or UniFFI cores MUST NOT be introduced unless this constitution is formally
  amended.
- A thin Swift watch-coordinator MAY be built locally when upstream client gaps exist
  (e.g., incomplete Informer support).

**Rationale**: Pure Swift keeps the stack simple, debuggable, and aligned with Principle I.

## Target Users

**Primary**: Individual engineers, SREs, and platform engineers who manage one or many
Kubernetes clusters from a Mac daily.

**Secondary**: Teams that want a consistent, high-quality native experience without forcing
everyone onto the same Electron app.

**Tertiary**: kubectl-comfortable engineers who want a GUI without leaving keyboard muscle
memory.

## Scope Boundaries

### In Scope (v1.0 MVP)

- Multi-cluster context switching (sequential connections; simultaneous watch is P2)
- Core resources: Pods, Deployments, Services, ConfigMaps, Secrets (masked), Namespaces,
  Nodes, Events
- Live list watching, detail inspection, YAML view/edit with validation
- Log streaming (single-pod v1.0; multi-pod P2)
- Port-forward and exec via native terminal panel
- Command palette and comprehensive shortcuts
- CRD discovery and generic resource browser (read-only v1.0 if Swiftkube CRD write gaps
  exist)

### Explicitly Deferred (v1.x)

- Helm management
- Metrics charts (metrics-server / Prometheus)
- Widgets, Shortcuts catalog, Menu Bar extra
- Multi-cluster diff / compare
- Windows or Linux ports

### Out of Scope (Indefinite)

- Cluster provisioning / creation
- GitOps full management (ArgoCD / Flux consoles)
- Cost analysis / FinOps
- Chaos engineering
- Full multi-user / team collaboration features

## Technical Standards

| Area | Requirement |
|------|-------------|
| Language | Swift 6+ |
| UI | SwiftUI + AppKit bridges |
| K8s client | Swiftkube/client (pinned version in Package.swift) |
| Architecture | MVVM + actor-isolated `ClusterSession` per context |
| Persistence | UserDefaults + Keychain (no Core Data in v1) |
| Testing | XCTest unit + UI tests for P1 flows; kind-based integration tests in CI |
| Distribution | Signed + notarized `.dmg` + Homebrew cask (`brew install k8steer`) |
| Minimum macOS | macOS 14 Sonoma |

Clean separation MUST exist between the UI layer and the Kubernetes data layer.

## Security Requirements

- Secrets MUST NOT be written to disk unencrypted.
- Sensitive kubeconfig fragments MUST use Keychain.
- TLS verification MUST be on by default; certificate pinning MAY be offered as an option.
- Update checks MUST NOT "phone home" without user consent (Sparkle with opt-in, or manual
  download only).
- The app MUST NOT transmit cluster data, credentials, or resource contents to any third
  party.

## Decision-Making Rule

When in doubt, choose the option that:

1. Feels more native on macOS
2. Improves performance or reduces resource usage
3. Increases user privacy and user control
4. Makes power users faster
5. Favors pure Swift over adding a second language

Any feature, library, or architectural decision that violates these principles requires
explicit discussion, documented rationale, and a Complexity Tracking entry in the
implementation plan.

## Governance

This Constitution is the highest authority in the project. It supersedes all specs, plans,
and informal conventions.

**Amendment procedure**:

- Propose change with documented rationale
- Bump constitution version per semantic versioning:
  - **MAJOR**: Principle removal or redefinition incompatible with prior compliance
  - **MINOR**: New principle or materially expanded guidance
  - **PATCH**: Clarifications, wording, non-semantic refinements
- Update dependent templates when principles affect planning or specification gates

**Compliance**:

- Every implementation plan MUST include a Constitution Check gate before Phase 0 research
  and again after Phase 1 design
- Violations MUST be listed in the plan's Complexity Tracking table with justification
- All PRs and reviews SHOULD verify compliance with applicable principles

**Naming**: Product name is **k8steer**. "Kubesteer" is a deprecated working title.
TODO: Complete trademark search before public launch (avoid confusion with KubeStellar and
similar marks).

**Version**: 1.0.0 | **Ratified**: 2026-08-24 | **Last Amended**: 2026-08-24
