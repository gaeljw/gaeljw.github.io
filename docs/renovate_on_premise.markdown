---
layout: default
title: Running Renovate on premise
description: How to self-host and run Renovate against your private Gitlab repositories
---

* Do not remove this line (it will not be displayed)
{:toc}

-----

_This article is not meant as a duplicate of the official documentation but rather as a walkthrough guide for self-hosting Renovate._
 
> _I wrote this article because the [official documentation](https://github.com/renovatebot/renovate/blob/main/docs/usage/getting-started/running.md) felt messy to me and I spent several hours figuring out the correct combination of parameters to use to run properly._

_Last update: 2023-08 with Renovate v36.49.0_

-----

# 🛠️ What is Renovate?

[Renovate](https://github.com/renovatebot/renovate) is a tool to **automate your dependencies updates**.

It supports:
- many languages and dependency managers. From Maven `pom.xml` to Kubernetes YAML manifests.
- all major source code management platforms. Gitlab, Github, Bitbucket... cloud or on-premise.

Renovate will look at your project source code, detect dependencies that are being used and open a Merge/Pull Request whenever one of the dependency has a newer version available.

It's similar to Github's own [Dependabot](https://docs.github.com/en/code-security/dependabot) or specific tools available in some languages (like [Scala Steward](https://github.com/scala-steward-org/scala-steward) for Scala) but way more flexible and customizable.

# 💡 Why use it?

**Having dependencies out-of-date is a major security risk** as vulnerabilities are being discovered and exploited on a daily basis. On the other hand, manually checking for new versions of dependencies is excessively time consuming.

In addition to the security matters, **technical debt** is another important thing to take care of: it's way easier to upgrade dependencies gradually when a new version comes out rather than waiting to be few versions late and spending several days upgrading.

Many projects exist with the same goal but they are most of the time limited to a specific scope of usage like only one language or one build tool. **Renovate supports almost all languages and tools in a unified way.**

Moreover, the community is very reactive when you have questions or potential issues.

IMHO, this makes Renovate a must-have in every organisation.

# 🧰 Running on-premise

Renovate can run in different ways. This article will focus on the steps needed to run it on-premise.

The infrastructure we'll use as a base is the following:
- a private on-premise **Gitlab instance hosting the repositories** to renovate
- a **task scheduler with ability to run containers**, this can be for instance:
  - a private on-premise Jenkins server
  - a Gitlab runner used via Gitlab CI (shell executor or Docker/Kubernetes executor directly)

![OnPremise architecture](./assets/renovate_on_premise/RenovateOnPrem.drawio.png)

The configurations we'll see below are specific to Gitlab as the project hosting platform but should be quite similar for other platforms.

Regarding the task scheduler, you can use any tool of your choice. We'll provide a very generic example as well as a Gitlab CI pipeline example.

We'll run Renovate by using the official container image (using Podman or Docker) but note that you could also run it using NPM directly. Using the container image has the advantage of being isolated, autonomous and avoiding conflicts with other tools.

-----

# ✍️ Setup

In order to run, Renovate need some configurations.

These configurations can be done in multiple ways:
- config file
- environment variables
- CLI flags

We chose to split our configurations in two subsets:
1. non secret configurations, managed by anyone, stored in a config file in Git in a dedicated repository
2. secret configurations (like a Gitlab token) and/or configurations managed by a subset of people (like a Platform Engineering team), stored in the task scheduler credentials and given as environment variables

![Configuration structure](./assets/renovate_on_premise/RenovateConfigArchi.drawio.png)

> _In a different context, this configuration structure choice could be totally different. It's up to you to define what's the more appropriate given your usage._
>
> _There can be one configuration repository per team for instance and thus one Renovate job per team._

Note that you can find all the configurations available in the official [self-hosted configuration options](https://docs.renovatebot.com/self-hosted-configuration/) documentation.

## Configuration file

Let's create a Gitlab repository _renovate-config_ in order to store a `config.js` file for the first subset of configurations.

The file must have the following minimum content:

```js
module.exports = {
    // Here the configurations we'll add (see below)
};
```

### Repositories list

Renovate needs to know which repositories to work on. This is the list of Gitlab repositories you want Renovate to look at to find dependencies upgrades and generate merge requests.

This has to be configured with the `repositories` key as a list of `string` with the name of the repositories.

```js
module.exports = {
    
    // ...

    repositories: [
        'myGitlabGroup/myProject',
        // List all your projects here
    ]
};
```

Renovate can also be setup to “autodiscover” projects, see more below.

### Other configurations

As a start, this is all we need in the configuration file. We may add other configurations later (see more in the ["Going Further"](#going-further) section).

## Environment variables

Given our configuration structure choice, the following items need to be stored in the task scheduler to be available to the Renovate job as environment variables.

### Gitlab details

- `RENOVATE_PLATFORM`: set value to `gitlab`
- `RENOVATE_ENDPOINT`: set value to your private Gitlab server URL, something like `http://gitlab.mycompany.net/api/v4`
  - If you're using Gitlab CI, you can use the variable `$CI_API_V4_URL` provided by Gitlab with exactly the desired value
- `RENOVATE_TOKEN`: an access token to Gitlab API, it has to be generated in Gitlab with the following scopes: `read_user`, `api`, `write_repository`. Store it as a secret value.

You can find more details in the [Renovate Gitlab Platform](https://docs.renovatebot.com/modules/platform/gitlab/) documentation.

### Github token

This one is not mandatory but strongly recommended if you want Renovate to be able to pull the changelogs of dependencies from their Github repository.

- `GITHUB_COM_TOKEN`: a token to Github API, it has to be generated in GitHub with read-only scopes. Store it as a secret value.

## Renovate job

Now that the minimum configurations are done, let's create the Renovate job.

The job is pretty straightforward to write. It needs to:
1. clone the Renovate configuration repository _renovate-config_ we created previously
2. inject previously created keys as environment variables
3. run the Renovate container

### Generic job

As (1) and (2) will depend on your task scheduler, we won't give more details in this article.

And regarding (3), it might depend on your task scheduler but the goal is to run something like the following command:
```shell
podman run --rm \
    -v "$(pwd)/config.js:/usr/src/app/config.js" \
    -e "GITHUB_COM_TOKEN" \
    -e "RENOVATE_TOKEN" \
    -e "RENOVATE_PLATFORM" \
    -e "RENOVATE_ENDPOINT" \
    renovate/renovate:36.49.0
```

Note that:
- it assumes the `config.js` file from _renovate-config_ repository is available in the current directory
- it assumes the environment variables are defined with the same name expected by Renovate, you could change the name and thus use the syntax `-e "GITHUB_COM_TOKEN=$MY_CUSTOM_GITHUB_TOKEN_VAR_NAME"`
- the Renovate image version (tag) is fixed, but it could be made an environment variable as well and/or use `latest` if you don't mind potential breaking changes in Renovate

### Gitlab CI job

There are multiple ways of running it with a Gitlab CI pipeline, we'll cover the most straightforward usage which is to add a pipeline in the _renovate-config_ repository and use a Gitlab runner that supports containerized jobs (using the `image` attribute).

Step (1) will happen automatically. Then (2) and (3) can be achieved with the following job:

```yaml
variables:
  # It's up to you to choose if you define these variables in your Gitlab instance or here directly in the file
  RENOVATE_PLATFORM: 'gitlab'
  RENOVATE_ENDPOINT: $CI_API_V4_URL
  # These two variables however must be kept secret and defined at project or instance level, not directly in the file.
  # You may want to have them defined globally with another name as we did here:
  RENOVATE_TOKEN: $CI_RENOVATE_GITLAB_TOKEN
  GITHUB_COM_TOKEN: $CI_RENOVATE_GITHUB_TOKEN

stages:
  - renovate

renovate:
  stage: renovate
  image: renovate/renovate:36.49.0
  script:
    - renovate
```

Note that:
- we didn't define any `tags` but this is likely required in your environment to target an appropriate Gitlab runner
- we didn't define any `cache` for now but that may greatly speed up runs, see more below in ["Going Further"](#going-further) section
- the Renovate image version (tag) is fixed, but it could be made an environment variable as well and/or use `latest` if you don't mind potential breaking changes in Renovate


# 🚀 Let it run!

Run your job once manually to see the results.

Output will look something like the following:
```
 INFO: Repository started (repository=myGitlabGroup/myProject)
       "renovateVersion": "36.27.2"

 INFO: Dependency extraction complete (repository=myGitlabGroup/myProject, baseBranch=master)
       "stats": {
         "managers": {
           "gitlabci-include": {"fileCount": 1, "depCount": 1},
           "npm": {"fileCount": 1, "depCount": 26}
         },
         "total": {"fileCount": 2, "depCount": 27}
       }

...
```

A first Merge Request should be created in the project(s) to be renovated explaining several things and what will be the next MRs that Renovate will open with actual dependency updates. You have to merge this first MR in order for Renovate to do anything more.

Renovate will actually create a file `renovate.json` at the root of the project in this first MR, in order to store its configuration. For now, the file should be almost empty. We'll see more about this below in ["Going Further"](#going-further) section.

Then, schedule the job according to your preferences: once a day can be a good start.

Bear in mind that if your project has many dependencies not up-to-date, it will take some initial effort to review and merge all MRs opened by Renovate. But then, the rate of opened MR should lower significantly.

-----

# ⏭️ Going further

## Autodiscover projects

So far, we've seen a setup where the list of projects to be renovated is manually managed. This can quickly become hard to keep up to date.

That's why Renovate also has a "autodiscover" feature which will find all projects it has access to (using the Gitlab token) and run Renovate on it.

To enable it, in the `config.js` file, use the [`autodiscover`](https://docs.renovatebot.com/self-hosted-configuration/#autodiscover) and optionally the [`autodiscoverFilter`](https://docs.renovatebot.com/self-hosted-configuration/#autodiscoverfilter) options:

```js
{
    autodiscover: true,
    autodiscoverFilter: ["some-group/*"],
    // Remove the existing repositories configuration
    // repositories: []
}
```

## Define configurations in a central place (Config Presets)

Each renovated project can define additional configurations in the `renovate.json` file to customize Renovate behaviour.

The list of possible customizations is large: adding labels to MRs, changing max number of opened MR, adding exclusion rules... You can find the exhaustive list in [Renovate Configuration Options](https://docs.renovatebot.com/configuration-options/) documentation.

It's likely that you'll want to share some configurations across multiple projects. This is possible with a feature called **[Config Presets](https://docs.renovatebot.com/config-presets/)** in Renovate.

With the setup we've seen previously, the _renovate-config_ repo can naturally serves as a place to store this preset.

To do so:
- create a `default.json` file in the repo to store the preset configurations
- reference this preset in your projects to be renovated

### Create the preset

The `default.json` should look something like this:
```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  // All your shared configurations
  // For instance:
  "dependencyDashboard": true,
  "packageRules": [
    {
      "matchDepTypes": ["test"],
      "automerge": true
    }
  ]
}
```

### Reference the preset

In your projects to be renovated, update the `renovate.json` file with following `extends` attribute:
```json
{
  "extends": [
    "local>renovate-config"
  ]
}
```

In this case, `local` means "the same Gitlab instance" as the one where the project to be renovated is stored.

Other options, like referencing a specific version, are possible. Check out [Config Presets Hosting](https://docs.renovatebot.com/config-presets/#gitlab).

### Automatically extend the preset for new projects

If you want the preset to be automatically configured for new projects, you can use the [`onboardingConfig`](https://docs.renovatebot.com/self-hosted-configuration/#onboardingconfig) attribute in the _renovate-config_ `config.js` file we defined at the beginning:

```javascript
module.exports = {
    
    // ...

    onboardingConfig: {
        extends: ['local>renovate-config']
    }
}
```

## Speed up runs with cache

You can significantly reduce duration of Renovate job runs by using a cache.

By default, Renovate stores various files in `/tmp/renovate` and `/tmp/renovate/cache`. You can change these directories with the [`baseDir`](https://docs.renovatebot.com/self-hosted-configuration/#basedir) and/or [`cacheDir`](https://docs.renovatebot.com/self-hosted-configuration/#cachedir) options if `/tmp` is not appropriate for you.

Depending on your task scheduler, you should look at implementing a way to reuse the content of these directories in upcoming runs.

### Generic job

A simple approach can be to mount the `/tmp/renovate` directory as a volume when running the container:

```
podman run --rm \
    -v "/tmp/renovate:/tmp/renovate" \
    ...
```

### Gitlab CI job

If using Gitlab CI, you can do this easily by adding following options in the job:

```yaml
renovate:
  # ...
  cache:
    paths:
      - renovate/
    when: 'always'
  variables:
    RENOVATE_BASE_DIR: './renovate'
```

It's necessary to override the `RENOVATE_BASE_DIR` as Gitlab CI can only cache items of the working directory.

# ℹ️ Tips

## Troubleshooting issues

If you need to troubleshoot any behaviour, the easiest way is to change the log level of Renovate.

To do so, you can set the environment variable `LOG_LEVEL=debug` in the same way as other variables.

This will give you a lot more details at what Renovate is doing.

## Using private registries

If you're using private registries, like a private Artifactory instance, you can configure Renovate to target them rather than the default ones with a setup like following:

```json
{
  "packageRules": [
    {
        "matchDatasources": ["maven"],
        "registryUrls": [
            "https://artifactory.mycompany.net/artifactory/maven-repo"
        ]
    }
  ]
}
```

## Using registries with a self-signed certificate

If you're using registries with a self-signed certificate, like a private Artifactory instance, you'll need to give the CA certificate to Renovate so HTTPS connections to the registries are possible.

To do so, you need to mount or inject somehow the certificate file in the Renovate container and set the `SSL_CERT_FILE` environment variable to the path of the file.

The actual setup will vary depending how you are able to retrieve the file.
