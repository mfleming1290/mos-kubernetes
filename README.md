# MOS Kubernetes

mos-kubernetes provides a **MOS plugin** that integrates the `k3s`
(lightweight Kubernetes) distribution into the MOS ecosystem.

---

## Overview

This repository contains the **MOS plugin implementation**, optional helper
functions, the SysV init script and configuration files (such as
`settings.json`).

The plugin allows MOS to make use of k3s.

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
