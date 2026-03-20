---
name: edgedelta-docker-image
version: 1.1.0
last_updated: 2026-03-20
description: Build custom EdgeDelta agent Docker images on any base OS with SELinux support. Use when customers need to run the agent on specific Linux distributions (Ubuntu, RHEL, Oracle Linux, SUSE, Alpine, etc.), need custom packages or tools installed alongside the agent, require custom CA certificates for corporate proxies, encounter the v0.0.0 version issue, or need SELinux guidance for enforcing systems. Recognizes phrases like "custom agent image", "build Docker image", "agent on RHEL", "agent on Ubuntu", "custom base image", "agent version shows 0.0.0", "add curl to agent container", "SELinux blocking agent", "agent can't read logs", and "permission denied in container".
---

# EdgeDelta Custom Docker Image Builder

Generates Dockerfiles for running the EdgeDelta agent on any base Linux distribution. Handles package manager differences, certificate installation, SELinux contexts, and platform-specific agent tags.

## When to Use This Skill

Activate this skill when the user:
- Needs the EdgeDelta agent running on a specific OS (Ubuntu, RHEL, OEL, SUSE, Alpine, etc.)
- Wants additional packages or tools installed alongside the agent
- Needs custom CA certificates for corporate proxy/firewall environments
- Sees the agent reporting version `v0.0.0`
- Asks about building a custom agent Docker image
- Needs to push a custom agent image to a private registry
- Mentions SELinux, FIPS, or compliance requirements for the agent container
- Gets `permission denied` errors when the agent tries to read host logs
- Needs to run the agent on SELinux-enforcing hosts (RHEL, OEL, CentOS, Rocky, Fedora)

## Critical Knowledge

### Agent Image Tags

The official agent images are at `gcr.io/edgedelta/agent`. Tags are **platform-suffixed** — there is no bare `:latest` manifest.

| Tag Format | Example | Use |
|------------|---------|-----|
| `latest-linux-amd64` | `gcr.io/edgedelta/agent:latest-linux-amd64` | Latest stable, x86_64 |
| `latest-linux-arm64` | `gcr.io/edgedelta/agent:latest-linux-arm64` | Latest stable, ARM64 |
| `v{X.Y.Z}-linux-amd64` | `gcr.io/edgedelta/agent:v2.13.0-linux-amd64` | Pinned version, x86_64 |
| `v{X.Y.Z}-linux-arm64` | `gcr.io/edgedelta/agent:v2.13.0-linux-arm64` | Pinned version, ARM64 |

**Common mistake**: Using `:latest`, `:v`, or an unsuffixed tag results in pull failures or `v0.0.0` version reporting.

To discover available tags:
```bash
gcloud container images list-tags gcr.io/edgedelta/agent --limit=10 --sort-by=~timestamp
```

### The v0.0.0 Problem

The agent version is compiled into the binary via Go ldflags. It defaults to `v0.0.0` if not set at build time. This happens when:
1. Building from the source Dockerfile without passing `--build-arg BUILD_VERSION=vX.Y.Z`
2. Using an invalid or placeholder image tag (`:v`, `:latest` without platform suffix)

**Fix**: Always use the multi-stage copy approach from an official image with a valid platform-specific tag. The version is already baked into the official binary.

### Package Managers by Distro

| Distro Family | Package Manager | Install Command | Update Command |
|---------------|----------------|-----------------|----------------|
| Ubuntu/Debian | apt | `apt install -y` | `apt update -y` |
| RHEL/CentOS/OEL/Rocky/Alma | dnf (or yum) | `dnf install -y` | `dnf update -y` |
| SUSE/SLES/openSUSE | zypper | `zypper install -y` | `zypper refresh` |
| Alpine | apk | `apk add --no-cache` | `apk update` |

### Certificate Handling

For corporate environments with internal CAs or TLS-intercepting proxies:

| Distro | Cert Directory | Update Command |
|--------|---------------|----------------|
| Ubuntu/Debian | `/usr/local/share/ca-certificates/` | `update-ca-certificates` |
| RHEL/OEL/CentOS | `/etc/pki/ca-trust/source/anchors/` | `update-ca-trust` |
| SUSE/SLES | `/usr/share/pki/trust/anchors/` | `update-ca-certificates` |
| Alpine | `/usr/local/share/ca-certificates/` | `update-ca-certificates` |

---

## SELinux Guide

This section is critical for customers running RHEL, Oracle Linux, CentOS, Rocky Linux, or Fedora with SELinux in enforcing mode.

### How SELinux Affects the Agent

When a container runtime launches a container, the process gets an SELinux label — a "type" that determines what the process can access. By default, containers run as `container_t`, which **cannot** read host files like `/var/log` or `/var/lib/docker/containers`, even when Unix file permissions allow it. The denial is silent from the container's perspective — the agent simply receives no log data.

### SELinux Container Types

| Type | Confined? | What It Can Do |
|------|-----------|---------------|
| `container_t` | Yes | Default container. No host file access. |
| `container_logreader_t` | Yes (limited) | Read-only access to `/var/log`. **Best for log collectors.** |
| `spc_t` | No (unconfined) | Full host access. Super Privileged Container. |
| `container_init_t` | Yes | For containers running systemd as PID 1. |

### Decision Tree: Which SELinux Type to Use

```
Agent needs to READ /var/log and /var/lib/docker?
  |
  +-- Yes --> Does agent also need /proc, cgroups, or Docker socket?
  |             |
  |             +-- No  --> Use container_logreader_t (preferred, least privilege)
  |             +-- Yes --> Use spc_t
  |
  +-- Security team prohibits spc_t?
        |
        +-- Yes --> Build a custom policy with udica or manual .te file
```

### Volume Mounts and SELinux: `:z` vs `:Z`

**CRITICAL: Do NOT use `:z` or `:Z` on `/var/log` or `/var/lib/docker/containers`.**

| Suffix | What It Does | Risk |
|--------|-------------|------|
| `:z` (lowercase) | Relabels host directory to `container_file_t` — all containers can access | Breaks host log tools (`journald`, `rsyslog`, `logrotate`) |
| `:Z` (uppercase) | Private relabel — only this container instance can access | Breaks on restart (new container gets different MCS label) |
| none | No relabeling — use with `container_logreader_t` or `spc_t` | Safe for system directories |

Relabeling `/var/log` changes the `var_log_t` label that system log tools depend on. If this has already happened, restore with:
```bash
restorecon -Rv /var/log
```

**For the agent's own writable data directory**, use persistent labeling instead of `:z`/`:Z`:
```bash
mkdir -p /opt/edgedelta/data
semanage fcontext -a -t container_file_t "/opt/edgedelta/data(/.*)?"
restorecon -Rv /opt/edgedelta/data
```

Or use a named Docker volume (automatically labeled `container_file_t`):
```bash
docker volume create edgedelta-data
docker run -v edgedelta-data:/var/lib/edgedelta ...
```

### Docker / Podman Run with SELinux

**Recommended: Use `container_logreader_t`** for read-only log access without relabeling:

```bash
docker run --rm --name edgedelta-agent \
    --security-opt label=type:container_logreader_t \
    -e ED_API_KEY=<API_KEY> \
    -v /var/log:/host/var/log:ro \
    -v /var/lib/docker/containers:/host/var/lib/docker/containers:ro \
    -v edgedelta-data:/var/lib/edgedelta \
    {IMAGE_NAME}:{IMAGE_TAG}
```

**If agent also needs /proc, cgroups, or Docker socket** (e.g., for container metrics):

```bash
docker run --rm --name edgedelta-agent \
    --security-opt label=type:spc_t \
    -e ED_API_KEY=<API_KEY> \
    -v /var/log:/host/var/log:ro \
    -v /var/lib/docker/containers:/host/var/lib/docker/containers:ro \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    -v edgedelta-data:/var/lib/edgedelta \
    {IMAGE_NAME}:{IMAGE_TAG}
```

Verify the type exists on your system:
```bash
seinfo -t container_logreader_t  # Should return the type
# If missing, install/update: dnf install container-selinux
```

### Kubernetes with SELinux

#### DaemonSet with `spc_t` (full access)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: edgedelta-agent
spec:
  selector:
    matchLabels:
      app: edgedelta-agent
  template:
    metadata:
      labels:
        app: edgedelta-agent
    spec:
      serviceAccountName: edgedelta-agent
      securityContext:
        seLinuxOptions:
          user: "system_u"
          role: "system_r"
          type: "spc_t"
          level: "s0"
      containers:
        - name: agent
          image: {REGISTRY}/{IMAGE_NAME}:{IMAGE_TAG}
          securityContext:
            runAsUser: 0
          env:
            - name: ED_API_KEY
              valueFrom:
                secretKeyRef:
                  name: edgedelta-secret
                  key: api-key
          volumeMounts:
            - name: varlog
              mountPath: /host/var/log
              readOnly: true
            - name: docker-containers
              mountPath: /host/var/lib/docker/containers
              readOnly: true
            - name: agent-data
              mountPath: /var/lib/edgedelta
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
            type: Directory
        - name: docker-containers
          hostPath:
            path: /var/lib/docker/containers
            type: DirectoryOrCreate
        - name: agent-data
          hostPath:
            path: /opt/edgedelta/data
            type: DirectoryOrCreate
```

#### DaemonSet with `container_logreader_t` (least privilege)

Replace the `securityContext` block:
```yaml
      securityContext:
        seLinuxOptions:
          user: "system_u"
          role: "system_r"
          type: "container_logreader_t"
          level: "s0"
```

This gives read-only access to host logs but no `/proc` or cgroup access. Use when the security team disallows `spc_t`.

#### OpenShift: SecurityContextConstraints

OpenShift requires an SCC to allow hostPath volumes:

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: edgedelta-agent-scc
allowPrivilegedContainer: false
allowHostDirVolumePlugin: true
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: MustRunAs
  seLinuxOptions:
    user: "system_u"
    role: "system_r"
    type: "spc_t"
    level: "s0"
volumes:
  - hostPath
  - configMap
  - secret
  - emptyDir
```

Apply and bind:
```bash
oc apply -f edgedelta-scc.yaml
oc adm policy add-scc-to-user edgedelta-agent-scc -z edgedelta-agent -n edgedelta
```

### Custom SELinux Policy (When `spc_t` Is Prohibited)

#### Method A: Automatic with `udica` (recommended)

`udica` inspects a running container and generates a tailored policy:

```bash
# Install udica
dnf install udica

# Run the agent container
podman run -d --name edgedelta-agent \
    -v /var/log:/host/var/log:ro \
    -v /var/lib/docker/containers:/host/var/lib/docker/containers:ro \
    -v /opt/edgedelta/data:/var/lib/edgedelta \
    edgedelta/agent:latest

# Export inspection and generate policy
podman inspect edgedelta-agent > edgedelta-agent.json
udica -j edgedelta-agent.json edgedelta_agent

# Load the policy
semodule -i edgedelta_agent.cil \
    /usr/share/udica/templates/{base_container.cil,net_container.cil,log_container.cil}

# Re-run with the custom type
podman run --security-opt label=type:edgedelta_agent.process \
    -v /var/log:/host/var/log:ro \
    edgedelta/agent:latest
```

#### Method B: Manual `.te` File (full control)

```te
policy_module(edgedelta_agent, 1.0)

# Define the agent process type
type edgedelta_agent_t;
type edgedelta_agent_exec_t;
application_domain(edgedelta_agent_t, edgedelta_agent_exec_t)

# Network: allow TCP/UDP egress for API calls
allow edgedelta_agent_t self:tcp_socket create_stream_socket_perms;
allow edgedelta_agent_t self:udp_socket create_socket_perms;
corenet_tcp_connect_all_ports(edgedelta_agent_t)
corenet_tcp_bind_generic_node(edgedelta_agent_t)

# Read host log files (var_log_t)
allow edgedelta_agent_t var_t:dir { getattr search open read };
allow edgedelta_agent_t var_log_t:dir { getattr search open read lock ioctl };
allow edgedelta_agent_t var_log_t:file { open getattr read ioctl lock };

# Read Docker/containerd container logs (container_var_lib_t)
allow edgedelta_agent_t container_var_lib_t:dir { getattr search open read };
allow edgedelta_agent_t container_var_lib_t:file { open getattr read ioctl };

# Read /proc for host metrics (optional)
kernel_read_system_state(edgedelta_agent_t)
kernel_read_kernel_sysctls(edgedelta_agent_t)

# Agent data directory (writable)
type edgedelta_agent_data_t;
files_type(edgedelta_agent_data_t)
allow edgedelta_agent_t edgedelta_agent_data_t:dir manage_dir_perms;
allow edgedelta_agent_t edgedelta_agent_data_t:file manage_file_perms;

# DNS resolution and /etc/resolv.conf
sysnet_dns_name_resolve(edgedelta_agent_t)
files_read_etc_files(edgedelta_agent_t)
```

Build and load:
```bash
dnf install selinux-policy-devel
make -f /usr/share/selinux/devel/Makefile edgedelta_agent.pp
semodule -i edgedelta_agent.pp

# Label the data directory
semanage fcontext -a -t edgedelta_agent_data_t "/opt/edgedelta/data(/.*)?"
restorecon -Rv /opt/edgedelta/data

# Run with custom type
docker run --security-opt label=type:edgedelta_agent_t \
    -v /var/log:/host/var/log:ro \
    -v /opt/edgedelta/data:/var/lib/edgedelta \
    edgedelta/agent:latest
```

Remove the module:
```bash
semodule -r edgedelta_agent
```

### SELinux Booleans

These on/off switches adjust SELinux policy without custom modules:

| Boolean | Default | When to Enable |
|---------|---------|----------------|
| `container_manage_cgroup` | off | Agent needs cgroup metrics (CPU/memory per container) |
| `container_read_certs` | off | Agent uses mTLS with custom cert store |
| `container_use_cephfs` | off | Collecting logs from CephFS mounts |

```bash
# Enable permanently
setsebool -P container_manage_cgroup 1

# Verify
getsebool container_manage_cgroup
```

If `setsebool` fails with "boolean does not exist", install `container-selinux`:
```bash
dnf install container-selinux
systemctl restart docker
```

### Troubleshooting SELinux Denials

#### Quick Diagnostic Sequence

```bash
# 1. Is SELinux active?
getenforce

# 2. Temporarily disable to confirm SELinux is the cause
setenforce 0          # Agent works now? SELinux was blocking.
setenforce 1          # Re-enable immediately

# 3. Find the denial
ausearch -m AVC -c edgedelta -ts recent

# 4. Human-readable explanation
ausearch -m AVC -ts recent | audit2why

# 5. Generate a fix
ausearch -m AVC -ts recent | audit2allow -M edgedelta_fix
cat edgedelta_fix.te  # REVIEW BEFORE LOADING
semodule -i edgedelta_fix.pp

# 6. Check contexts
ps -eZ | grep edgedelta      # Process context
ls -Z /var/log/               # File contexts
ls -Z /opt/edgedelta/         # Data dir context
```

#### Reading a Denial

```
type=AVC msg=audit(...): avc: denied { read } for
  pid=12345 comm="edgedelta"
  scontext=system_u:system_r:container_t:s0:c1,c2     <-- process label
  tcontext=system_u:object_r:var_log_t:s0              <-- target file label
  tclass=file permissive=0
```

This says: process running as `container_t` tried to read a file labeled `var_log_t` and was denied.

#### Common Failure Modes

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| Agent starts, no errors, but receives no log data | `container_t` can't read `var_log_t` | Use `container_logreader_t` or `spc_t` |
| Agent crashes with `permission denied` on startup | Data directory labeled `default_t` | `semanage fcontext` + `restorecon` on data dir |
| Agent worked once, fails on restart | `:Z` volume flag — new MCS label per container | Use `semanage fcontext` instead of `:Z` |
| Host `journalctl`/`rsyslog` broke after running agent | `:z` relabeled `/var/log` to `container_file_t` | `restorecon -Rv /var/log` |
| Agent can't connect to EdgeDelta API | Usually NOT SELinux (`container_t` allows TCP) | Check firewall/proxy; verify with `ausearch` |
| `setsebool` says boolean doesn't exist | `container-selinux` package missing | `dnf install container-selinux` |
| Agent worked, broke after system relabel/reboot | `chcon` changes don't survive relabel | Use `semanage fcontext` for permanent rules |

#### Required Host Packages for SELinux Troubleshooting

```bash
dnf install \
    container-selinux \
    selinux-policy-devel \
    udica \
    setroubleshoot-server \
    policycoreutils-python-utils
```

---

## Workflow

### Step 1: Gather Requirements

Ask the user:
1. **Base OS**: Which Linux distribution and version? (Ubuntu 22.04, RHEL 9, OEL 8, Alpine, etc.)
2. **Architecture**: amd64 or arm64? (default: amd64)
3. **Agent version**: Latest stable or pinned version? (default: latest)
4. **Additional packages**: What tools do they need? (curl, bash, debugging tools, etc.)
5. **Custom certificates**: Do they need to add internal CA certs?
6. **SELinux**: Is SELinux enforcing on their hosts? (check with `getenforce`)
7. **Registry**: Where will they push the image? (optional)

### Step 2: Select the Agent Source Tag

Based on architecture and version preference:
```
Architecture: amd64, Version: latest  --> gcr.io/edgedelta/agent:latest-linux-amd64
Architecture: arm64, Version: latest  --> gcr.io/edgedelta/agent:latest-linux-arm64
Architecture: amd64, Version: v2.13.0 --> gcr.io/edgedelta/agent:v2.13.0-linux-amd64
```

If the user wants to verify available tags:
```bash
gcloud container images list-tags gcr.io/edgedelta/agent --filter="tags:latest OR tags:v2.13" --limit=10 --format='table(tags,timestamp)'
```

### Step 3: Generate the Dockerfile

Use the appropriate template below based on the distro family. Always use multi-stage build to copy the pre-built agent binary — never rebuild from source.

#### Template: Debian/Ubuntu

```dockerfile
FROM --platform=linux/{ARCH} gcr.io/edgedelta/agent:{TAG} AS edgedelta_image
FROM ubuntu:{VERSION}
COPY --from=edgedelta_image /edgedelta/edgedelta /edgedelta/edgedelta
COPY --from=edgedelta_image /etc/ssl/certs /etc/ssl/certs/
RUN apt update -y && apt install -y --no-install-recommends \
    ca-certificates \
    {EXTRA_PACKAGES} \
    && rm -rf /var/lib/apt/lists/*
# CUSTOM_CERTS: Uncomment if adding internal CA certificates
# COPY my-corp-ca.crt /usr/local/share/ca-certificates/
# RUN update-ca-certificates
ENV EDGEDELTA_AS_DOCKER=1
ENTRYPOINT ["/edgedelta/edgedelta", "-c", "/edgedelta/config.yml"]
```

#### Template: RHEL / Oracle Linux / Rocky / AlmaLinux

```dockerfile
FROM --platform=linux/{ARCH} gcr.io/edgedelta/agent:{TAG} AS edgedelta_image
FROM {DISTRO_IMAGE}:{VERSION}
COPY --from=edgedelta_image /edgedelta/edgedelta /edgedelta/edgedelta
COPY --from=edgedelta_image /etc/ssl/certs /etc/ssl/certs/
RUN dnf update -y && dnf install -y \
    ca-certificates \
    {EXTRA_PACKAGES} \
    && dnf clean all
# CUSTOM_CERTS: Uncomment if adding internal CA certificates
# COPY my-corp-ca.crt /etc/pki/ca-trust/source/anchors/
# RUN update-ca-trust
ENV EDGEDELTA_AS_DOCKER=1
ENTRYPOINT ["/edgedelta/edgedelta", "-c", "/edgedelta/config.yml"]
```

Distro image references:
- RHEL UBI: `redhat/ubi9:latest`, `redhat/ubi8:latest`
- Oracle Linux: `oraclelinux:9`, `oraclelinux:8`
- Rocky Linux: `rockylinux:9`, `rockylinux:8`
- AlmaLinux: `almalinux:9`, `almalinux:8`

#### Template: SUSE / SLES / openSUSE

```dockerfile
FROM --platform=linux/{ARCH} gcr.io/edgedelta/agent:{TAG} AS edgedelta_image
FROM opensuse/leap:{VERSION}
COPY --from=edgedelta_image /edgedelta/edgedelta /edgedelta/edgedelta
COPY --from=edgedelta_image /etc/ssl/certs /etc/ssl/certs/
RUN zypper refresh && zypper install -y \
    ca-certificates \
    {EXTRA_PACKAGES} \
    && zypper clean --all
# CUSTOM_CERTS: Uncomment if adding internal CA certificates
# COPY my-corp-ca.crt /usr/share/pki/trust/anchors/
# RUN update-ca-certificates
ENV EDGEDELTA_AS_DOCKER=1
ENTRYPOINT ["/edgedelta/edgedelta", "-c", "/edgedelta/config.yml"]
```

#### Template: Alpine (minimal)

```dockerfile
FROM --platform=linux/{ARCH} gcr.io/edgedelta/agent:{TAG} AS edgedelta_image
FROM alpine:{VERSION}
COPY --from=edgedelta_image /edgedelta/edgedelta /edgedelta/edgedelta
COPY --from=edgedelta_image /etc/ssl/certs /etc/ssl/certs/
RUN apk update && apk add --no-cache \
    ca-certificates \
    {EXTRA_PACKAGES}
# CUSTOM_CERTS: Uncomment if adding internal CA certificates
# COPY my-corp-ca.crt /usr/local/share/ca-certificates/
# RUN update-ca-certificates
ENV EDGEDELTA_AS_DOCKER=1
ENTRYPOINT ["/edgedelta/edgedelta", "-c", "/edgedelta/config.yml"]
```

### Step 4: Build Instructions

Provide the build command:
```bash
docker build --platform=linux/{ARCH} -t {IMAGE_NAME}:{IMAGE_TAG} .
```

If pushing to a private registry:
```bash
docker tag {IMAGE_NAME}:{IMAGE_TAG} {REGISTRY}/{IMAGE_NAME}:{IMAGE_TAG}
docker push {REGISTRY}/{IMAGE_NAME}:{IMAGE_TAG}
```

### Step 5: Run Instructions

#### Docker Run (non-SELinux)
```bash
docker run --rm --name edgedelta-agent \
    -e ED_API_KEY=<API_KEY> \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    -v /var/lib/docker/containers/:/var/lib/docker/containers/:ro \
    {IMAGE_NAME}:{IMAGE_TAG}
```

#### Docker Run (SELinux enforcing)
```bash
docker run --rm --name edgedelta-agent \
    --security-opt label=type:container_logreader_t \
    -e ED_API_KEY=<API_KEY> \
    -v /var/log:/host/var/log:ro \
    -v /var/lib/docker/containers:/host/var/lib/docker/containers:ro \
    -v edgedelta-data:/var/lib/edgedelta \
    {IMAGE_NAME}:{IMAGE_TAG}
```

See the **SELinux Guide** section above for `spc_t`, custom policies, and K8s configuration.

#### Kubernetes DaemonSet (snippet)
```yaml
spec:
  containers:
    - name: edgedelta-agent
      image: {REGISTRY}/{IMAGE_NAME}:{IMAGE_TAG}
      env:
        - name: ED_API_KEY
          valueFrom:
            secretKeyRef:
              name: edgedelta-secret
              key: api-key
      volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: docker-containers
          mountPath: /var/lib/docker/containers
          readOnly: true
  volumes:
    - name: varlog
      hostPath:
        path: /var/log
    - name: docker-containers
      hostPath:
        path: /var/lib/docker/containers
```

For SELinux K8s environments, add to the pod spec (see SELinux Guide for full DaemonSet):
```yaml
spec:
  securityContext:
    seLinuxOptions:
      type: spc_t
```

### Step 6: Verify

After the container starts, check the logs for the correct version:
```bash
docker logs <container_id> 2>&1 | grep "Build version"
```

Expected output:
```
INFO  edgedelta/main.go:156  Build version: "v2.13.0", build time: "..."
```

If it shows `v0.0.0`, the image tag was invalid. Verify the tag exists:
```bash
gcloud container images list-tags gcr.io/edgedelta/agent --filter="tags:{TAG}" --limit=5
```

---

## Example Interactions

**User**: "I need the EdgeDelta agent running on Oracle Linux 8 with curl"
**Assistant**: Generates OEL 8 Dockerfile using `oraclelinux:8` base, copies from `gcr.io/edgedelta/agent:latest-linux-amd64`, installs curl via dnf.

**User**: "We're running RHEL 9 with SELinux enforcing and need custom CA certs"
**Assistant**: Generates RHEL UBI9 Dockerfile with cert installation. For Docker: provides `--security-opt label=type:container_logreader_t`. For K8s: provides DaemonSet with `seLinuxOptions`. Warns about `:z`/`:Z` on `/var/log`.

**User**: "My agent shows v0.0.0"
**Assistant**: Identifies the tag issue, shows how to find valid tags, regenerates Dockerfile with correct platform-specific tag.

**User**: "Build me an Alpine agent image with bash and jq for debugging"
**Assistant**: Generates minimal Alpine Dockerfile with bash and jq via apk.

**User**: "Agent gets permission denied reading /var/log on our RHEL boxes"
**Assistant**: Walks through SELinux diagnosis: `getenforce`, `ausearch -m AVC -c edgedelta`, recommends `container_logreader_t` or `spc_t`, provides `docker run` and K8s fixes.

**User**: "Our security team won't allow spc_t, how do we run the agent?"
**Assistant**: Generates custom SELinux policy using `udica` or manual `.te` file scoped to `var_log_t` read + `container_var_lib_t` read + network egress.

## When NOT to Use This Skill

- Building the agent from source (internal dev) -- use `edgedelta-dev-practices` skill
- Pipeline configuration -- use `edgedelta-pipelines` skill
- Agent runtime configuration (YAML) -- use `edgedelta-pipelines` or `edgedelta-reference` skills
- Windows agent containers -- not covered (different binary and base image)

## Reference

- [EdgeDelta Documentation](https://docs.edgedelta.com)
- [EdgeDelta Agent Installation](https://docs.edgedelta.com/installation)
- [Red Hat: SELinux Container Labeling Advice](https://developers.redhat.com/articles/2025/04/11/my-advice-selinux-container-labeling)
- [Red Hat: Creating SELinux Policies for Containers](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/using_selinux/creating-selinux-policies-for-containers_using-selinux)
- [GitHub: containers/udica](https://github.com/containers/udica) -- automatic SELinux policy generator
- [GitHub: containers/container-selinux](https://github.com/containers/container-selinux) -- container_logreader_t and other types
- [Kubernetes: Configure Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- Agent images: `gcr.io/edgedelta/agent`
- Source Dockerfile: `build/Dockerfiles/agent/Dockerfile` in the edgedelta repo
- Version variable: `pkg/build/build.go` -- injected via `-ldflags -X`
