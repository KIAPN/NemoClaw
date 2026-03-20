# ADR-001: Local Ollama Inference on DGX Spark

**Status:** Accepted
**Date:** 2026-03-19
**Authors:** flyguyryeguy, Claude Code
**Hardware:** DGX Spark (spark-9371), GB10/ARM64, 128 GB unified memory
**Software:** NemoClaw 0.1.0, OpenShell 0.0.10, OpenClaw 2026.3.11, Ollama 0.18.1

## Context

NemoClaw is NVIDIA's enterprise wrapper around OpenClaw, an open-source AI coding agent. NemoClaw runs OpenClaw inside an OpenShell sandbox with network policy enforcement, binary restrictions, and a proxy-based security model.

We need to run NemoClaw on a DGX Spark with local Ollama inference using models already downloaded to the device (Nemotron 3 Super 120B, Nemotron 3 Nano 30B). This avoids cloud API costs, latency, and data egress for a device that has more than enough compute to run inference locally.

### The Problem

NVIDIA's intended setup path (`nemoclaw onboard` with Ollama selected) does not work on DGX Spark. Local Ollama support is behind `NEMOCLAW_EXPERIMENTAL=1` and has six independent failure points in OpenShell 0.0.10.

NVIDIA's supported local inference path is NIM (NVIDIA Inference Microservices), not Ollama. NIM does not run natively on GB10. DGX Spark owners who want local inference are in an unsupported gap.

## Decision

**Bypass `inference.local` entirely and route OpenClaw directly to Ollama via the Docker bridge IP.**

We chose to:

1. Create a patched Dockerfile (`Dockerfile.spark-ollama`) that hardcodes `baseUrl: http://172.18.0.1:11434/v1` instead of `https://inference.local/v1`
2. Write a custom sandbox network policy (`sandbox-policy-ollama.yaml`) that allows direct Ollama access on Docker bridge IPs
3. Apply UFW rules on the host for all three Docker bridge subnets
4. Use the `ollama/` model ID prefix in OpenClaw config (required by OpenClaw's provider routing)
5. Set request timeout to 120s (Ollama cold start can take 50-74s for large models)

## Alternatives Considered

### A. Wait for NVIDIA to fix `inference.local` (Rejected)

Issues #314, #326, #385 are open but there's no timeline for a fix. `inference.local` is architecturally broken for local providers — the OpenShell proxy hardcodes upstream URLs by provider type and runs DNS resolution before intercepting the virtual hostname. This is not a configuration issue; it requires code changes to OpenShell.

### B. Use NIM instead of Ollama (Rejected)

NIM is NVIDIA's supported local inference backend, but:
- NIM does not run natively on GB10
- The `nim-local` blueprint profile expects NIM at `nim-service.local:8000`
- NIM requires a separate container with GPU passthrough (untested on Spark)
- Ollama already works and has all four Nemotron models loaded

### C. Run inference outside the sandbox (Rejected)

We could skip the sandbox entirely and run OpenClaw directly on the host. This would work but defeats the purpose of NemoClaw's security model — the sandbox provides network policy enforcement, binary restrictions, and filesystem isolation. Our use case (eventually giving NemoClaw access to repos and tools) requires the security boundary.

### D. Patch `inference.local` DNS inside the sandbox (Rejected)

Adding `inference.local` to `/etc/hosts` inside the sandbox was considered, but:
- The sandbox runs as `sandbox` user (no root)
- Even if we added it, the proxy still runs DNS resolution independently
- The proxy's `allowed_ips` check would still fail for the resolved IP

## The Six Layers of Breakage

Each layer is an independent failure that must be addressed. Fixing only some layers produces different error messages but the same result: 403 Forbidden.

### Layer 1: Sandbox network policy missing Ollama entry

**Symptom:** `FORWARD action=deny reason=endpoint inference.local:80 not in policy 'ollama_local'`

The default sandbox policy (`openclaw-sandbox.yaml`) has no entry for local Ollama endpoints. The `nemoclaw onboard` wizard is supposed to write an Ollama policy but hard-exits at step 5 on DGX Spark with:

> "Local Ollama is responding on localhost, but containers cannot reach http://host.openshell.internal:11434"

**Fix:** Custom `sandbox-policy-ollama.yaml` with `ollama_local` network policy entries.

### Layer 2: Policy endpoint format requires `access: full`

**Symptom:** `FORWARD action=deny reason=endpoint has L7 rules; use CONNECT`

Using `protocol: rest` with `rules:` does not work for plain HTTP endpoints. The proxy attempts L7 inspection which fails for non-TLS traffic.

**Fix:** Use `access: full` instead of `protocol: rest` on all Ollama endpoints.

### Layer 3: Private IPs require explicit `allowed_ips`

**Symptom:** `FORWARD blocked: internal IP without allowed_ips dst_host=172.17.0.1`

The proxy blocks connections to RFC1918 addresses by default, even when the endpoint is in the policy.

**Fix:** Add `allowed_ips` to each endpoint entry listing the Docker bridge IPs.

### Layer 4: UFW blocks Docker bridge traffic

**Symptom:** Connection timeout from container to host port 11434

Ubuntu 24.04 on DGX Spark has UFW enabled by default. Docker bridge subnets are not whitelisted.

**Fix:**
```bash
sudo ufw allow from 172.17.0.0/16 to any port 11434 comment "Docker default bridge"
sudo ufw allow from 172.18.0.0/16 to any port 11434 comment "OpenShell containers"
sudo ufw allow from 172.20.0.0/16 to any port 11434 comment "Gateway container"
```

Note: Three subnets exist because OpenShell creates separate Docker networks for the gateway and sandbox clusters.

### Layer 5: `inference.local` proxy interception bug

**Symptom:** `FORWARD blocked: allowed_ips check failed reason=DNS resolution failed for inference.local:80`

Even after adding `inference.local` to the policy with `access: full` and `allowed_ips`, the OpenShell proxy runs DNS resolution on `inference.local` *before* intercepting it as a virtual hostname. Since `inference.local` is not a real DNS name, resolution fails.

Additionally, the gateway hardcodes upstream URLs by provider type (GitHub issue #385):
- `openai` type → routes to `api.openai.com` (ignores `baseUrl` config)
- `nvidia` type → routes to `integrate.api.nvidia.com`
- `generic` type → rejected by `openshell inference set`

**This is a bug in OpenShell, not a configuration issue.**

**Fix:** Bypass `inference.local` entirely. Configure OpenClaw to use the direct Docker bridge IP (`http://172.18.0.1:11434/v1`) in `openclaw.json`.

### Layer 6: OpenClaw model ID requires provider prefix

**Symptom:** `FailoverError: Unknown model: anthropic/nemotron-3-nano:latest`

When the model ID in `openclaw.json` is `nemotron-3-nano:latest` (without a provider prefix), OpenClaw defaults to the `anthropic/` prefix. The model must be specified as `ollama/nemotron-3-nano:latest` to match the provider name in the config.

**Fix:** Use `ollama/nemotron-3-nano:latest` as the primary model in `openclaw.json`.

### Layer 6b: OpenClaw default request timeout too short

**Symptom:** `LLM request timed out.`

OpenClaw's default HTTP request timeout is shorter than Ollama's cold-start time (50-74s for Nemotron 3 Super 120B on GB10). The request completes successfully if given enough time.

**Fix:** Set `timeoutSeconds: 120` and `requestTimeoutMs: 120000` in `openclaw.json`, or pass `--timeout 120` on the command line.

## Working Configuration

### Files

| File | Location | Purpose |
|------|----------|---------|
| `Dockerfile.spark-ollama` | `~/NemoClaw/` | Patched sandbox image — routes to Ollama directly |
| `sandbox-policy-ollama.yaml` | `~/NemoClaw/` | Network policy with Ollama access on all Docker bridge IPs |

### Dockerfile Changes (vs upstream)

The only change from the upstream `Dockerfile` is the `openclaw.json` generation step:

| Setting | Upstream | Ours |
|---------|----------|------|
| `baseUrl` | `https://inference.local/v1` | `http://172.18.0.1:11434/v1` |
| `apiKey` | `openshell-managed` | `local-ollama` |
| Provider name | `nvidia` | `ollama` |
| Primary model | `nvidia/nemotron-3-super-120b-a12b` | `ollama/nemotron-3-nano:latest` |
| `timeoutSeconds` | (not set) | `120` |

### Network Policy Changes (vs upstream)

Added `ollama_local` policy with endpoints for:
- `172.17.0.1:11434` (Docker default bridge)
- `172.18.0.1:11434` (OpenShell cluster bridge)
- `172.20.0.1:11434` (gateway cluster bridge)
- `host.openshell.internal:11434` (OpenShell host alias)
- `inference.local:80` and `:443` (in case OpenShell fixes the proxy bug later)

All with `access: full` and `allowed_ips`. Binaries: `openclaw`, `node`, `curl`.

### Build Procedure

```bash
# Stage build context (workaround for .dockerignore excluding nemoclaw/dist/)
BUILDCTX=$(mktemp -d /tmp/nemoclaw-build-XXXX)
cp ~/NemoClaw/Dockerfile.spark-ollama "$BUILDCTX/Dockerfile"
cp -r ~/NemoClaw/nemoclaw "$BUILDCTX/nemoclaw"
cp -r ~/NemoClaw/nemoclaw-blueprint "$BUILDCTX/nemoclaw-blueprint"
cp -r ~/NemoClaw/scripts "$BUILDCTX/scripts"

# Create sandbox
openshell sandbox create \
  --from "$BUILDCTX/Dockerfile" \
  --name devin \
  --policy ~/NemoClaw/sandbox-policy-ollama.yaml \
  --no-tty

# Clean up
rm -rf "$BUILDCTX"
```

### Test

```bash
# Inside sandbox:
export NVIDIA_API_KEY=local-ollama
export ANTHROPIC_API_KEY=local-ollama
openclaw agent --agent main --local \
  -m "how many rs in strawberry?" \
  --session-id test1 \
  --timeout 120
```

## Consequences

### Positive

- Local inference works end-to-end on DGX Spark with Ollama
- Sandbox security model preserved (network policy enforcement, binary restrictions)
- No cloud API dependency or data egress
- Both Nano (30B) and Super (120B) models available
- Build is reproducible from two files (Dockerfile + policy YAML)

### Negative

- Hardcoded Docker bridge IP (`172.18.0.1`) — will break if Docker networking changes
- Bypasses OpenShell's inference routing entirely — no `inference.local` integration
- NemoClaw plugin still registers against `build.nvidia.com` (cosmetic, doesn't affect function)
- Must rebuild sandbox image to change model or Ollama endpoint
- Not aligned with NVIDIA's supported path — may diverge further as NemoClaw evolves

### Risks

- OpenShell updates could change the proxy behavior and break our policy workarounds
- Docker bridge IP could change after a `docker network prune` or gateway restart
- If NVIDIA fixes `inference.local`, we should migrate back to the standard path to reduce maintenance burden

## Related Issues

| Issue | URL | Relevance |
|-------|-----|-----------|
| #314 | https://github.com/NVIDIA/NemoClaw/issues/314 | Original 403 bug report (our KIAPN comment) |
| #326 | https://github.com/NVIDIA/NemoClaw/issues/326 | inference.local routing not created (macOS) |
| #385 | https://github.com/NVIDIA/NemoClaw/issues/385 | "No path exists in OpenShell 0.0.10" for local routing |
| #391 | https://github.com/NVIDIA/NemoClaw/issues/391 | Missing binary in policy causes silent 403 |
| #307 | https://github.com/NVIDIA/NemoClaw/issues/307 | Egress proxy blocks direct API access |

## Review Triggers

Re-evaluate this ADR when:
- OpenShell releases a version that fixes `inference.local` for local providers
- NIM becomes available on GB10/DGX Spark
- Docker networking changes on the host (check `docker network inspect openshell-cluster-nemoclaw | grep Gateway`)
- NVIDIA moves Ollama support out of `NEMOCLAW_EXPERIMENTAL`
