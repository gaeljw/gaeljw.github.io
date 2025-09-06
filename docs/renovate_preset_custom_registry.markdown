---
layout: default
title: Custom registry Renovate preset
description: A Renovate preset to configure custom registry
---

* Do not remove this line (it will not be displayed)
{:toc}

-----

# Purpose

This Renovate preset is an example for configuring a custom registry for a datasource.

Use-cases:

* a company registry that stores internal artifacts. Renovate would otherwise not be able to detect updates for such artifacts.
* a company registry that acts as a proxy to external registries. This is a way to enforce dependencies are discovered through the proxy.

# Preset

This is an example for Maven/SBT, Docker & Helm. It can be extended to support all [datasources](https://docs.renovatebot.com/modules/datasource/) that Renovate supports.

**preset.json**:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "description": "Add company registries for each datasource type",
  "packageRules": [
    {
      "matchDatasources": [
        "maven",
        "sbt-package",
        "sbt-plugin"
      ],
      "registryUrls": [
        "https://artifactory.mycompany.net/artifactory/internal-releases",
        "https://artifactory.mycompany.net/artifactory/external"
      ]
    },
    {
      "matchDatasources": [
        "docker"
      ],
      "registryUrls": [
        "https://container-registry.mycompany.net/"
      ]
    },
    {
      "matchDatasources": [
        "helm"
      ],
      "registryUrls": [
        "https://artifactory.mycompany.net/artifactory/helm"
      ]
    }
  ]
}
```

# Other presets

Check out [other useful presets](./index.markdown).
