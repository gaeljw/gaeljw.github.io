---
layout: default
title: Eclipse Temurin container image OS Renovate preset
description: A Renovate preset to suggest OS updates for Eclipse Temurin container images
---

* Do not remove this line (it will not be displayed)
{:toc}

-----

# Purpose

This preset adds the ability for Renovate to suggest upgrades to OS variants for `eclipse-temurin` images which [it doesn't do by default](https://docs.renovatebot.com/modules/versioning/docker/).

See also the [discussion](https://github.com/renovatebot/renovate/discussions/33934) that led to this configuration.

# Example

This preset will suggest changes similar to the following:

**Dockerfile**:

```diff
- FROM eclipse-temurin:21-jre-jammy
+ FROM eclipse-temurin:21-jre-noble
```

# Preset

**preset.json5**:

```json5
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "customManagers": [
    {
      "customType": "regex",
      // In which file to look for something to do
      // Same pattern as standard Docker manager (https://docs.renovatebot.com/modules/manager/dockerfile/)
      "managerFilePatterns": [
        "/(^|/|\\.)([Dd]ocker|[Cc]ontainer)file$/",
        "/(^|/)([Dd]ocker|[Cc]ontainer)file[^/]*$/"
      ],
      // Regex to extract some attributes from the Dockerfile
      // I.e. from "FROM eclipse-temurin:21.0.5_11-jre-jammy", how to extract "21.0.5_11" and "jammy"
      "matchStrings": [
        "FROM eclipse-temurin:(?<regularVersion>.*)-jre-(?<currentValue>.*)"
      ],
      // From the list of available versions (image tags on container registry), how to extract versions
      // i.e. from "21.0.5_11-jre-noble" or "21.0.6_7-jre-noble", how to filter only the same Java version and identify
      // "noble" as the version we care about here
      "extractVersionTemplate": "^{{{regularVersion}}}-jre-(?<version>.*)$",
      // Defining a custom depName so that it doesn't clash with regular updates of Java version
      "depNameTemplate": "eclipse-temurin-os-variant",
      // Name of the package to look for in the datasource
      // Using packageName because depName is overriden
      "packageNameTemplate": "eclipse-temurin",
      // Versioning to use
      // This is where the magic appear: Renovate knows that noble > jammy for instance
      "versioningTemplate": "ubuntu",
      // Where to look for available versions
      "datasourceTemplate": "docker"
    }
  ],
  // Add an extra packageRule to overcome a default preset 'workarounds:javaLTSVersions' that comes with Renovate and
  // define specific versioning and allowedVersions for Java dependencies
  // See: https://docs.renovatebot.com/presets-workarounds/#workaroundsjavaltsversions
  "packageRules": [
    {
      "matchDatasources": [
        "docker"
      ],
      // Matching the depName configured in the custom manager
      "matchDepNames": [
        "eclipse-temurin-os-variant"
      ],
      // Overriding the versioning and allowedVersions from preset
      "versioning": "ubuntu",
      "allowedVersions": "/.*/"
    }
  ]
}
```

_Note that it only handles `-jre` variants of the images but it can be tweaked to support both `-jre` and `-jdk`._

# Other presets

Check out [other useful presets](./index.markdown).
