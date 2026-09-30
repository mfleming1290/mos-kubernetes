# MOS Kubernetes

mos-kubernetes provides a **MOS plugin** that integrates the `k3s`
(lightweight Kubernetes) distribution into the MOS ecosystem.

---

## Overview

This repository contains the **MOS plugin implementation**, optional helper
functions, the SysV init script and configuration files (such as
`settings.json`).

The plugin allows MOS to make use of k3s as either a control-plane/server node or an agent/worker node.

### Cluster modes

- **Server** starts `k3s server` and can host the control plane.
- **Agent** starts `k3s agent` and joins an existing server by passing join flags such as `--server` and `--token` through Start Arguments.
- Server and agent runtime state are kept separate (`data` and `agent-data`) so switching modes does not overwrite the other mode's cluster state.


### Binary Source

- k3s: [https://github.com/k3s-io/k3s](https://github.com/k3s-io/k3s)

---

## Build & Automation

This repository includes a **GitHub Actions workflow** used to build and package
the plugin and its associated components for MOS.

The build process is fully automated and produces artifacts that can be
installed through the MOS Hub.

---

## Licensing

The contents of this repository (plugin code, build scripts, configuration,
and automation) are licensed under **GPL-3.0**.

`k3s` itself is licensed under its respective upstream license.

---

## Third-Party Software

This repository builds and packages third-party open-source software.
Packaged components remain licensed under their original upstream licenses.

Refer to `THIRD_PARTY.md` for details.
