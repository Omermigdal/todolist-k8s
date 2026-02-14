# Kubernetes Deployment Requirements

## 1. Deployments

Deploy two applications:

- **Web UI**: `todolist-vue`
- **REST API**: `todo-api`

Each deployment should include:

- InitContainer(s)
- Liveness / Readiness Probes
- Resource Requests and Limits
- Horizontal Pod Autoscaler (HPA)
- Optional enhancements (as applicable)

---

## 2. StatefulSet

Create **one StatefulSet** with the following characteristics:

- Single replica
  > No advanced application-level setup required
- Headless Service

---

## 3. Configuration Across Components

Shared and application-specific configuration should be managed using:

- ConfigMap(s)
- Secret(s)

---

## 4. Optional Components

The following components are optional but recommended:

- Network Policies
- Ingress
  - Simple setup
  - Frontend exposed at `/`

---

## 5. Configurable Parameters (`values.yaml`)

### Replicas

- Number of replicas for each deployment(default:1)

### Services

- `ServiceType` for frontend and web UI (default:LoadBlancer)

### Environment Variables

- `USER` (default : your lastname-firstname)
- `MYSQL_HOST` (default: backend)
- `MYSQL_ROOT_PASSWORD` (no default – to be set in command line)
- `API_BASE_URL`

### Optional Configuration

- Set a custom title image
  - Provide `title-image.png` via a ConfigMap

---

## Documentation Requirement

The chart **must include a `NOTES.txt` file** containing usage instructions for the deployed application.
