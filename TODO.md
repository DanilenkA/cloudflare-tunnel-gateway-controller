# TODO: Cloudflare Tunnel Gateway Controller

## Helm Chart Audit Fixes

### 🔴 CRITICAL (Must Fix Before Production)

#### Issue #1: Secret creates with empty values
**Status:** 🔄 In Progress
**Priority:** P0
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/values.schema.json` ✅ Fixed
- `charts/cloudflare-tunnel-gateway-controller/templates/secret.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/tests/secret_test.yaml` ⏳ Pending
- Create `charts/cloudflare-tunnel-gateway-controller/tests/schema_validation_test.yaml` ⏳ Pending

**Description:** Secret is created when `apiToken: ""` (empty string), but controller crashes at runtime. Need to add validation to fail at install time.

**Changes:**
- [x] Add `required: ["tunnelId"]` in cloudflare object
- [x] Add `minLength: 1` and `pattern: "^[a-zA-Z0-9-]+$"` for tunnelId
- [ ] Add empty value checks in secret.yaml template
- [ ] Create schema validation tests
- [ ] Update secret tests with edge cases

---

#### Issue #2: tunnelId not required in schema
**Status:** ✅ Fixed
**Priority:** P0
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/values.schema.json` ✅ Fixed

**Description:** `tunnelId` is not marked as required inside cloudflare object, allowing helm install without it.

**Changes:**
- [x] Add `required: ["tunnelId"]` inside cloudflare object properties
- [x] Add `minLength: 1` validation
- [x] Add pattern validation for tunnel ID format

---

#### Issue #3: NetworkPolicy allows entire internet
**Status:** ⏳ Pending
**Priority:** P0
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/templates/networkpolicy.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/tests/networkpolicy_test.yaml` ⏳ Pending

**Description:** NetworkPolicy egress rule uses `0.0.0.0/0:443` instead of specific Cloudflare IP ranges.

**Security Impact:** Compromised controller can communicate with any external server on port 443.

**Changes:**
- [ ] Replace `0.0.0.0/0` with Cloudflare IP ranges:
  - 104.16.0.0/13
  - 172.64.0.0/13
  - 173.245.48.0/20
  - 103.21.244.0/22
  - 103.22.200.0/22
  - 103.31.4.0/22
  - 141.101.64.0/18
  - 108.162.192.0/18
  - 190.93.240.0/20
  - 188.114.96.0/20
  - 197.234.240.0/22
  - 198.41.128.0/17
  - 162.158.0.0/15
  - IPv6 ranges: 2606:4700::/32, 2803:f800::/32, 2405:b500::/32, 2405:8100::/32, 2a06:98c0::/29, 2c0f:f248::/32
- [ ] Update tests to verify no 0.0.0.0/0 present

---

#### Issue #4: RBAC excessive permissions
**Status:** ⏳ Pending
**Priority:** P0
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/templates/clusterrole.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/tests/rbac_test.yaml` ⏳ Pending

**Description:** RBAC grants `update/patch` on Gateway spec when only `/status` subresource should be writable.

**Security Impact:** Controller can modify Gateway spec, violates least privilege principle.

**Changes:**
- [ ] Remove `update`, `patch` verbs from `gateways` resource
- [ ] Add separate rule for `gateways/status` with `get`, `update`, `patch`
- [ ] Similarly for `httproutes` - read-only for spec, write for status
- [ ] Update tests to verify split permissions

---

### ⚠️ SERIOUS (Should Fix for Production-Ready)

#### Issue #5: Missing startupProbe
**Status:** ⏳ Pending
**Priority:** P1
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/templates/deployment.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/tests/deployment_test.yaml` ⏳ Pending

**Description:** No startupProbe configured, pod may be killed during slow initialization.

**Changes:**
- [ ] Add startupProbe before livenessProbe
- [ ] Set `initialDelaySeconds: 0`, `periodSeconds: 5`, `failureThreshold: 12` (60s total)
- [ ] Add test for startupProbe configuration

---

#### Issue #6: Missing PodDisruptionBudget
**Status:** ⏳ Pending
**Priority:** P1
**Files:**
- Create `charts/cloudflare-tunnel-gateway-controller/templates/poddisruptionbudget.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/values.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/values.schema.json` ⏳ Pending
- Create `charts/cloudflare-tunnel-gateway-controller/tests/poddisruptionbudget_test.yaml` ⏳ Pending

**Description:** No PodDisruptionBudget means cluster upgrades can kill all replicas simultaneously.

**Changes:**
- [ ] Create PDB template with `minAvailable: 1` default
- [ ] Add `podDisruptionBudget.enabled: false` to values.yaml
- [ ] Add PDB configuration to values.schema.json
- [ ] Create comprehensive tests for PDB
- [ ] Update production-values.yaml example

---

#### Issue #7: No default resource limits
**Status:** ⏳ Pending
**Priority:** P1
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/templates/deployment.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/values.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/tests/deployment_test.yaml` ⏳ Pending

**Description:** `resources: {}` allows pod to consume all node resources.

**Changes:**
- [ ] Change deployment.yaml to set default limits when resources empty
- [ ] Default limits: cpu=200m, memory=256Mi
- [ ] Default requests: cpu=100m, memory=128Mi
- [ ] Update values.yaml to show defaults
- [ ] Add tests for default resources and custom overrides

---

#### Issue #8: Incorrect appVersion
**Status:** ⏳ Pending
**Priority:** P1
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/Chart.yaml` ⏳ Pending

**Description:** appVersion stuck at "0.0.1", should reflect actual controller version.

**Changes:**
- [ ] Update appVersion from "0.0.1" to "0.0.2"

---

#### Issue #9: AWG preStop hook dangerous
**Status:** ⏳ Pending
**Priority:** P1
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/templates/deployment.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/tests/deployment_test.yaml` ⏳ Pending

**Description:** AWG preStop hook uses `sh` instead of `/bin/sh`, `|| true` hides errors.

**Changes:**
- [ ] Change command to use `/bin/sh` (full path)
- [ ] Remove `|| true`, use `set -e` with proper if/else
- [ ] Check if interface exists before deletion
- [ ] Add logging to preStop hook
- [ ] Add test to verify /bin/sh usage and error handling

---

#### Issue #10: No secret rotation strategy
**Status:** ⏳ Pending
**Priority:** P1
**Files:**
- Create `charts/cloudflare-tunnel-gateway-controller/examples/external-secrets-operator-values.yaml` ⏳ Pending

**Description:** No support for external secret operators (ESO, Vault, etc.) for credential rotation.

**Changes:**
- [ ] Create comprehensive External Secrets Operator example
- [ ] Document SecretStore creation
- [ ] Document ExternalSecret creation
- [ ] Add AWS Secrets Manager example
- [ ] Add Vault example in comments

---

### ℹ️ RECOMMENDATIONS (Nice to Have)

#### Issue #11: AWG interfaceName can conflict
**Status:** ⏳ Pending
**Priority:** P2
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/values.yaml` ⏳ Pending

**Description:** Default interfaceName "awg-cfd-gw-ctrl0" conflicts with multiple installations.

**Changes:**
- [ ] Add warning comment about unique names
- [ ] Suggest using environment/release name in interface name
- [ ] Example: `awg-cfd-gw-ctrl-prod`, `awg-cfd-gw-ctrl-staging`

---

#### Issue #12: Missing terminationGracePeriodSeconds
**Status:** ⏳ Pending
**Priority:** P2
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/templates/deployment.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/values.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/values.schema.json` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/tests/deployment_test.yaml` ⏳ Pending

**Description:** No explicit terminationGracePeriodSeconds, defaults to 30s.

**Changes:**
- [ ] Add `terminationGracePeriodSeconds: 30` to deployment.yaml (configurable)
- [ ] Add to values.yaml with default 30
- [ ] Add to values.schema.json with min: 0, max: 3600
- [ ] Add tests for default and custom values

---

#### Issue #13: Service missing prometheus annotations
**Status:** ⏳ Pending
**Priority:** P2
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/templates/service.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/values.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/tests/service_test.yaml` ⏳ Pending

**Description:** Service lacks prometheus.io/* annotations for scraping fallback.

**Changes:**
- [ ] Add prometheus.io/scrape, port, path annotations when serviceMonitor.enabled
- [ ] Add service.annotations to values.yaml
- [ ] Add tests for prometheus annotations

---

#### Issue #14: NetworkPolicy ingress without namespace selector
**Status:** ⏳ Pending
**Priority:** P2
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/templates/networkpolicy.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/values.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/tests/networkpolicy_test.yaml` ⏳ Pending

**Description:** Ingress allows from any namespace, no restrictions on metrics access.

**Changes:**
- [ ] Add `ingressNamespaceSelector` to values.yaml
- [ ] Add `ingressPodSelector` to values.yaml
- [ ] Update networkpolicy.yaml to use selectors
- [ ] Add tests for namespace/pod selectors

---

#### Issue #15: Missing tests for edge cases
**Status:** ⏳ Pending
**Priority:** P2
**Files:**
- Create `charts/cloudflare-tunnel-gateway-controller/tests/edge_cases_test.yaml` ⏳ Pending

**Description:** No tests for edge cases like empty values, conflicts, special characters.

**Changes:**
- [ ] Test empty tunnelId (should fail)
- [ ] Test both apiToken and apiTokenSecretName (secretName wins)
- [ ] Test multiple replicas with AWG
- [ ] Test very long release names (truncation)
- [ ] Test special characters in tunnelId
- [ ] Test PDB with single replica
- [ ] Test multiple security contexts

---

#### Issue #16: production-values.yaml incomplete
**Status:** ⏳ Pending
**Priority:** P2
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/examples/production-values.yaml` ⏳ Pending

**Description:** Missing PDB, priority, DNS, service annotations, pod annotations.

**Changes:**
- [ ] Add PodDisruptionBudget configuration
- [ ] Add priorityClassName with documentation
- [ ] Add dnsPolicy and dnsConfig
- [ ] Add service.annotations
- [ ] Add podAnnotations for observability
- [ ] Add podLabels for environment identification

---

#### Issue #17: Missing troubleshooting section in README
**Status:** ⏳ Pending
**Priority:** P2
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/README.md` ⏳ Pending

**Description:** No troubleshooting section for common issues.

**Changes:**
- [ ] Add Troubleshooting section after Configuration
- [ ] Document: Controller crashes on startup
- [ ] Document: Gateway not accepting routes
- [ ] Document: NetworkPolicy blocks traffic
- [ ] Document: AWG sidecar conflicts
- [ ] Document: Resource limits causing OOMKilled
- [ ] Document: Schema validation errors
- [ ] Add debug mode instructions
- [ ] Add links to GitHub issues/discussions

---

#### Issue #18: .helmignore missing .markdownlint.yaml
**Status:** ⏳ Pending
**Priority:** P3
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/.helmignore` ⏳ Pending

**Description:** .markdownlint.yaml not excluded from package, increases size.

**Changes:**
- [ ] Add .markdownlint.yaml to .helmignore
- [ ] Add .editorconfig
- [ ] Add IDE directories (.vscode/, .idea/)
- [ ] Add editor temp files (*.swp, *.swo, *~)
- [ ] Add .DS_Store

---

#### Issue #19: values.schema.json too permissive for securityContext
**Status:** ⏳ Pending
**Priority:** P2
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/values.schema.json` ⏳ Pending

**Description:** securityContext has no validation, users can set insecure values.

**Changes:**
- [ ] Add schema for podSecurityContext properties
- [ ] Add schema for securityContext properties
- [ ] Define allowed types and minimum values
- [ ] Add enum for seccompProfile.type

---

#### Issue #20: Missing dnsPolicy/dnsConfig
**Status:** ⏳ Pending
**Priority:** P2
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/templates/deployment.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/values.yaml` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/values.schema.json` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/tests/deployment_test.yaml` ⏳ Pending

**Description:** No DNS customization options for controller pods.

**Changes:**
- [ ] Add dnsPolicy to values.yaml (default: ClusterFirst)
- [ ] Add dnsConfig to values.yaml
- [ ] Add to deployment.yaml template
- [ ] Add to values.schema.json with enum for dnsPolicy
- [ ] Add tests for custom DNS configuration

---

## Finalization Tasks

### Documentation Updates
**Status:** ⏳ Pending
**Priority:** P1
**Files:**
- `charts/cloudflare-tunnel-gateway-controller/Chart.yaml` ⏳ Pending
- Create `charts/cloudflare-tunnel-gateway-controller/CHANGELOG.md` ⏳ Pending
- `charts/cloudflare-tunnel-gateway-controller/README.md.gotmpl` ⏳ Pending (if exists)

**Changes:**
- [ ] Update Chart.yaml artifacthub.io/changes annotation with all fixes
- [ ] Create comprehensive CHANGELOG.md
- [ ] Add Best Practices section to README
- [ ] Add Security Hardening section
- [ ] Add AWG Sidecar Considerations
- [ ] Regenerate README.md with helm-docs if using .gotmpl

---

## Testing & Validation

### Per-Batch Testing
**Status:** ⏳ Pending
**Priority:** P0

**Commands to run after each batch:**
```bash
# Lint chart
helm lint charts/cloudflare-tunnel-gateway-controller/

# Run unit tests
helm unittest charts/cloudflare-tunnel-gateway-controller/

# Validate schema
helm template test charts/cloudflare-tunnel-gateway-controller/ \
  --values charts/cloudflare-tunnel-gateway-controller/examples/production-values.yaml

# Dry-run install
helm install test-release charts/cloudflare-tunnel-gateway-controller/ \
  --namespace test \
  --create-namespace \
  --dry-run \
  --debug
```

### Final Validation
**Status:** ⏳ Pending
**Priority:** P0

**Before merge:**
- [ ] All helm unittest tests pass
- [ ] helm lint returns no errors
- [ ] Schema validation works correctly
- [ ] All examples install without errors
- [ ] Chart packages without warnings
- [ ] README.md regenerated and accurate
- [ ] CHANGELOG.md complete

---

## Commit Strategy

### Batch 1.1: Validation and Secrets (P0)
**Files:** values.schema.json, secret.yaml, tests/*
**Commit Message:** `fix(helm): add validation for tunnelId and apiToken`

### Batch 1.2: RBAC Permissions (P0)
**Files:** clusterrole.yaml, tests/rbac_test.yaml
**Commit Message:** `fix(helm): limit RBAC to status subresource only`

### Batch 1.3: NetworkPolicy Security (P0)
**Files:** networkpolicy.yaml, tests/networkpolicy_test.yaml
**Commit Message:** `fix(helm): restrict NetworkPolicy to Cloudflare IP ranges`

### Batch 2.1: Deployment Improvements (P1)
**Files:** deployment.yaml, values.yaml, tests/deployment_test.yaml
**Commit Message:** `feat(helm): add startupProbe, resource limits, and improve AWG preStop hook`

### Batch 2.2: High Availability (P1)
**Files:** poddisruptionbudget.yaml, values.yaml, values.schema.json, tests/*
**Commit Message:** `feat(helm): add PodDisruptionBudget for high availability`

### Batch 2.3: Version Update (P1)
**Files:** Chart.yaml
**Commit Message:** `chore(helm): update appVersion to 0.0.2`

### Batch 2.4: External Secrets Documentation (P1)
**Files:** examples/external-secrets-operator-values.yaml
**Commit Message:** `docs(helm): add External Secrets Operator integration example`

### Batch 3.1: DNS Configuration (P2)
**Files:** deployment.yaml, values.yaml, values.schema.json, tests/*
**Commit Message:** `feat(helm): add dnsPolicy and dnsConfig support`

### Batch 3.2: Monitoring Improvements (P2)
**Files:** service.yaml, values.yaml, tests/service_test.yaml
**Commit Message:** `feat(helm): add prometheus annotations to Service`

### Batch 3.3: NetworkPolicy Ingress (P2)
**Files:** networkpolicy.yaml, values.yaml, tests/networkpolicy_test.yaml
**Commit Message:** `feat(helm): add namespace selectors for NetworkPolicy ingress`

### Batch 3.4: Test Coverage (P2)
**Files:** tests/edge_cases_test.yaml
**Commit Message:** `test(helm): add comprehensive edge case tests`

### Batch 3.5: Documentation (P2)
**Files:** README.md, production-values.yaml, .helmignore
**Commit Message:** `docs(helm): add troubleshooting section and improve examples`

### Batch 4.1: Finalization (P1)
**Files:** Chart.yaml, CHANGELOG.md, README.md.gotmpl
**Commit Message:** `docs(helm): update changelog and finalize documentation`

---

## Estimated Effort

| Phase | Tasks | Estimated Time |
|-------|-------|----------------|
| Phase 1: Critical Fixes | Issues #1-4 | 2-3 hours |
| Phase 2: Serious Fixes | Issues #5-10 | 2-3 hours |
| Phase 3: Recommendations | Issues #11-20 | 2-3 hours |
| Phase 4: Finalization | Documentation | 1 hour |
| Testing & Validation | All phases | 2 hours |
| **TOTAL** | **20 issues** | **10-12 hours** |

---

## Progress Tracking

**Overall Progress:** 2/20 issues fixed (10%)

**By Priority:**
- P0 (Critical): 2/4 fixed (50%) - Issues #1, #2 ✅
- P1 (Serious): 0/6 fixed (0%)
- P2 (Recommendations): 0/10 fixed (0%)

**Last Updated:** 2025-11-25

---

## Notes

- All fixes follow semantic commit message format
- Each batch has corresponding tests
- Schema validation enforced where possible
- Security hardening prioritized in all changes
- Production-readiness is the goal
- Breaking changes are avoided where possible
