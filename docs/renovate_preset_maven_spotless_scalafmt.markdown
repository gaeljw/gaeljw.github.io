---
layout: default
title: Scalafmt Maven Spotless plugin Renovate preset
description: A Renovate preset to suggest Scalafmt upgrades when used in Maven Spotless plugin
---

* Do not remove this line (it will not be displayed)
{:toc}

-----

# Purpose

This preset adds the ability for Renovate to suggest upgrades to the [Scalafmt](https://scalameta.org/scalafmt) version set in [Maven Spotless plugin](https://github.com/diffplug/spotless).

As Renovate natively [supports upgrading version in the Scalafmt configuration file](https://docs.renovatebot.com/modules/manager/scalafmt/), this preset will include the changes to the `pom.xml` file in the same PR/MR as the `.scalafmt.conf` file.

# Example

This preset will suggest changes similar to the following:

**pom.xml**:

```diff
<plugin>
  <groupId>com.diffplug.spotless</groupId>
  <artifactId>spotless-maven-plugin</artifactId>
  <version>2.46.1</version>
  <configuration>
      <scala>
        <scalafmt>
-            <version>3.9.7</version>
+            <version>3.9.8</version>
            <file>${project.basedir}/.scalafmt.conf</file>
        </scalafmt>
      </scala>
  </configuration>
</plugin>
```

# Preset

**preset.json5**:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "description": "Configure Renovate to suggest upgrades to the Scalafmt version set in Spotless plugin parameters. In same PR as .scalafmt.conf file.",
  "customManagers": [
    {
      "customType": "regex",
      // Where to look for (pom.xml)
      "managerFilePatterns": ["/(^|/)pom\\.xml$/"],
      // What to look for and extract from the pom.xml
      "matchStrings": [
        "<scalafmt>\\s*<version>(?<currentValue>.*?)<\\/version>"
      ],
      // Using same settings as scalafmt manager
      // Especially the depName so that the upgrade is part of the same PR as .scalafmt.conf file
      // See https://github.com/renovatebot/renovate/blob/main/lib/modules/manager/scalafmt/extract.ts#L18 for reference
      "depNameTemplate": "scalafmt",
      "datasourceTemplate": "github-releases",
      "packageNameTemplate": "scalameta/scalafmt",
      "extractVersionTemplate": "^v(?<version>\\S+)",
      "versioningTemplate": "semver"
    }
  ]
}
```

# Other presets

Check out [other useful presets](./index.markdown).
