# Feature Specification: Cluster Connection and Pod Browser

**Feature Branch**: `001-cluster-pod-browser`

**Created**: 2026-08-24

**Status**: Draft

**Input**: User description: "P1 cluster connection and pod browser MVP: load kubeconfig contexts, connect to one cluster, list pods across namespaces with live watch, command palette stub with Switch Context action"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Connect to a cluster from kubeconfig (Priority: P1)

An engineer opens k8steer for the first time. The app reads their existing `~/.kube/config`,
shows available contexts, and lets them connect to one cluster without creating an account or
entering credentials the app stores elsewhere.

**Why this priority**: Without a working cluster connection, no other feature delivers value.
This is the foundation of the entire product.

**Independent Test**: Can be fully tested by launching the app with a valid kubeconfig,
selecting a context, and confirming the app reports a successful connection to that cluster's
API server.

**Acceptance Scenarios**:

1. **Given** a valid kubeconfig with at least one context, **When** the user launches
   k8steer, **Then** all contexts from the kubeconfig are listed with their cluster and
   namespace defaults visible.
2. **Given** multiple contexts in kubeconfig, **When** the user selects a context,
   **Then** the app connects to that cluster's API server and shows connection status
   (connected / connecting / failed with reason).
3. **Given** a context whose API server is unreachable, **When** the user selects it,
   **Then** the app shows a clear error message without crashing and allows retry or
   context switch.
4. **Given** standard kubeconfig auth (cert, token, exec), **When** the user connects,
   **Then** authentication succeeds using the same credentials kubectl would use.

---

### User Story 2 - Browse pods across all namespaces (Priority: P1)

A connected engineer views a scrollable list of all pods in the cluster, sees key status at
a glance, and can inspect a pod's details without leaving the app.

**Why this priority**: Pod browsing is the most common daily task and validates list
performance, watch reliability, and native UI feel.

**Independent Test**: Can be tested by connecting to a cluster with known pods, verifying the
list matches expected pod count and status, opening a pod detail view, and confirming smooth
scrolling with hundreds of rows.

**Acceptance Scenarios**:

1. **Given** a connected cluster with pods in multiple namespaces, **When** the user opens
   the pod browser, **Then** all pods across all namespaces appear in a single list with
   name, namespace, status, node, and age columns.
2. **Given** a pod list with 500+ entries, **When** the user scrolls rapidly,
   **Then** scrolling remains smooth with no visible stutter.
3. **Given** a pod in the list, **When** the user selects it, **Then** a detail panel shows
   pod metadata, container statuses, labels, and conditions.
4. **Given** a running watch connection, **When** a pod is created or deleted in the cluster,
   **Then** the list updates within 3 seconds without manual refresh.

---

### User Story 3 - Switch context via command palette (Priority: P2)

A power user presses a keyboard shortcut to open the command palette and switches to a
different cluster context without using the mouse.

**Why this priority**: Validates keyboard-first workflow (Constitution Principle IV) early,
even as a minimal stub before the full palette ships in v1.0.

**Independent Test**: Can be tested by pressing the command palette shortcut, typing a
context name, selecting it, and confirming the app reconnects to the new cluster.

**Acceptance Scenarios**:

1. **Given** the app is running with multiple contexts available, **When** the user presses
   `⌘K`, **Then** a command palette overlay appears with a search field focused.
2. **Given** the command palette is open, **When** the user types a partial context name,
   **Then** matching contexts are filtered in real time.
3. **Given** a filtered context in the palette, **When** the user selects "Switch Context",
   **Then** the app disconnects from the current cluster, connects to the selected context,
   and refreshes the pod list.

---

### Edge Cases

- What happens when `~/.kube/config` does not exist? App shows a helpful message with
  guidance to create or point to a kubeconfig file.
- What happens when kubeconfig contains invalid YAML? App shows a parse error with the
  file path; no crash.
- What happens when the user has no RBAC permission to list pods cluster-wide? App shows
  a permission-denied error with the API server's message.
- What happens when the watch stream disconnects (network blip, API server restart)? App
  reconnects automatically within 3 seconds and resumes the list without user action.
- What happens when switching contexts while a watch is active? Previous watch is cancelled
  cleanly; new context connects and starts a fresh watch.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST read contexts from the user's kubeconfig file on launch (Principle
  II — local-first).
- **FR-002**: System MUST connect to the Kubernetes API server using the selected context's
  credentials without shelling out to kubectl (Principle VII).
- **FR-003**: System MUST display connection status (connecting, connected, failed) for the
  active context.
- **FR-004**: System MUST list all pods across all namespaces when connected, showing at
  minimum: name, namespace, phase/status, node, and age.
- **FR-005**: System MUST maintain a live watch on the pod list and reflect create/update/delete
  events without manual refresh.
- **FR-006**: System MUST allow the user to select a pod and view its detail (metadata,
  containers, labels, conditions).
- **FR-007**: System MUST virtualize the pod list so scrolling remains smooth with 500+
  entries (Principle III).
- **FR-008**: System MUST provide a command palette accessible via `⌘K` with at least a
  "Switch Context" action (Principle IV).
- **FR-009**: System MUST show clear, actionable error messages when connection or listing
  fails (Principle V).
- **FR-010**: System MUST NOT transmit kubeconfig contents or cluster data to any third party
  (Principle II, Security Requirements).

### Key Entities

- **KubeContext**: A named entry from kubeconfig — cluster reference, user/auth reference,
  default namespace. Attributes: name, cluster URL, auth method, is-active.
- **ClusterConnection**: An active session to one context's API server. Attributes: context
  name, status, last-error, connected-at.
- **PodSummary**: A lightweight pod representation for the list view. Attributes: name,
  namespace, phase, node, age, ready count, restart count.
- **PodDetail**: Full pod inspection data. Attributes: metadata, spec summary, container
  statuses, conditions, labels, annotations.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users with a valid kubeconfig can connect to a cluster and see pods within
  10 seconds of first launch.
- **SC-002**: Pod list updates reflect cluster changes within 3 seconds of the event
  occurring.
- **SC-003**: Scrolling a list of 500 pods feels smooth with no dropped frames on Apple
  Silicon hardware.
- **SC-004**: Users can switch cluster context via command palette in under 5 seconds
  (keyboard-only, no mouse).
- **SC-005**: Connection and permission errors display a message the user can act on (retry,
  switch context, or check credentials) without app restart.

## Assumptions

- Users have macOS 14 or later and a valid kubeconfig at `~/.kube/config` (or `KUBECONFIG`
  env var path).
- Users have network access to their cluster API server from the Mac running k8steer.
- Users have at least `list pods` RBAC permission cluster-wide or in namespaces they care
  about; permission errors are acceptable for restricted contexts.
- This MVP connects to one cluster at a time; simultaneous multi-cluster watch is out of
  scope (deferred to P2 per constitution).
- Command palette in this feature is a stub — only "Switch Context" is required; other
  actions come in later features.
