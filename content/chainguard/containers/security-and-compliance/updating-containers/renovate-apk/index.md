---
title: "Using Renovate to update packages"
linktitle: "Using Renovate to update packages"
type: "article"
description: "How to configure Renovate's APK datasource to keep pinned package versions in your Dockerfiles up to date."
date: 2026-09-20T00:00:00+00:00
lastmod: 2026-09-20T00:00:00+00:00
draft: false
tags: ["Chainguard Containers"]
images: []
weight: 035
toc: true
---

Pinned package versions accumulate CVEs over time and may become unavailable as Chainguard [removes older versions from its repositories](/chainguard/containers/building-and-modifying/packages/package-model/#package-retention-in-public-repositories). This guide describes how to configure Renovate to update `apk add pkg=version` pins in Dockerfiles through its [APK datasource](https://docs.renovatebot.com/modules/datasource/apk/), keeping them current as new builds ship.

For updating references to Chainguard Containers themselves, refer to [Using Renovate with Chainguard Containers](/chainguard/containers/security-and-compliance/updating-containers/renovate/).

> **Note**: **Pin APK packages and images together.** Newer images introduced by mutable tags may include newer packages that conflict with your older pinned versions. If you are pinning package versions then you should also pin the base image to a digest and use Renovate to keep both up to date. This ensures a higher degree of reproducibility and avoids unexpected build failures.

> **Note**: **Renovate only supports exact package names.** It doesn't resolve APK `provides` aliases, so pin the fully-qualified name (e.g `argo-cd-2.14`, not `argo-cd`).

## Prerequisites

To follow this guide, you need:

* Renovate installed and configured. Refer to Renovate's [installation instructions](https://docs.renovatebot.com/getting-started/installing-onboarding/) if you haven't set this up.
* `chainctl`, Chainguard's command-line interface, installed on your local machine. Refer to [How to install `chainctl`](/platform/chainctl-usage/how-to-install-chainctl/) if you haven't set this up.
* A Dockerfile that pins one or more packages to a specific version, for example `RUN apk add --no-cache curl=8.12.1-r0`.

## Default repositories

Chainguard Containers ship with their `/etc/apk/repositories` file already populated with two public, org-scoped mirrors served from `virtualapk.cgr.dev`. Neither requires authentication:

* **`virtualapk.cgr.dev/<org-id>/chainguard`** — open-source packages used in Chainguard's Free container images.
* **`virtualapk.cgr.dev/<org-id>/extra-packages`** — additional packages that aren't fully open source but can still be redistributed by Chainguard.

If you aren't modifying `/etc/apk/repositories` in your build then you should add a `packageRules` entry to your `renovate.json` that matches the `apk` datasource and lists both of the default URLs.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended"
  ],
  "packageRules": [
    {
      "matchDatasources": ["apk"],
      "registryUrls": [
        "https://virtualapk.cgr.dev/<org-id>/chainguard?arch=x86_64",
        "https://virtualapk.cgr.dev/<org-id>/extra-packages?arch=x86_64"
      ]
    }
  ]
}
```

Replace `<org-id>` with your organization's ID, which you can find by running `chainctl iam org list -o table`.

Set `arch=aarch64` if you are building exclusively for that architecture.

In practice, Chainguard publish almost every package with the same versions for each architecture. However, there are some exceptions, so if you are building on both, you may consider having duplicate URLs for each architecture, or having specific `packageRules` for different Dockerfiles depending on target architecture.

## Private repository

Your organization-scoped [private repository](/chainguard/containers/building-and-modifying/packages/private-apk-repos/) (**`apk.cgr.dev/<org-name>`**) serves the packages your organization is entitled to and provides packages that are not available from the public mirrors.

If you are modifying the `/etc/apk/repositories` file in your builds to include it, then you should also include it in your Renovate configuration.

Firstly, create a pull token scoped to the `apk` repository:

```shell
chainctl auth pull-token create --repository=apk --ttl=259200m
```

The command prints a JSON object containing an `identity_id` and a `token`. When authenticating to `apk.cgr.dev`, use `identity_id` as the username and `token` as the password. The `--ttl=259200m` value sets a six-month lifetime; shorten it for tighter rotation or lengthen it as needed.

Then, add the private repository's URL to `registryUrls` and add a `hostRules` entry so Renovate can authenticate with the pull token:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended"
  ],
  "hostRules": [
    {
      "matchHost": "apk.cgr.dev",
      "username": "{{ secrets.PULL_TOKEN_USERNAME }}",
      "password": "{{ secrets.PULL_TOKEN_PASSWORD }}"
    }
  ],
  "packageRules": [
    {
      "matchDatasources": ["apk"],
      "registryUrls": [
        "https://virtualapk.cgr.dev/<org-id>/chainguard?arch=x86_64",
        "https://virtualapk.cgr.dev/<org-id>/extra-packages?arch=x86_64",
        "https://apk.cgr.dev/<org-name>?arch=x86_64"
      ]
    }
  ]
}
```

Remove the public repositories if you are using the private repository exclusively.

Replace `<org-id>` and `<org-name>` with your organization's values, which you can find by running `chainctl iam org list -o table`.

At runtime, inject the pull token into the `PULL_TOKEN_USERNAME` and `PULL_TOKEN_PASSWORD` secrets via the `RENOVATE_SECRETS` environment variable:

```shell
export RENOVATE_SECRETS="{\"PULL_TOKEN_USERNAME\": \"<identity-id>\", \"PULL_TOKEN_PASSWORD\": \"<token>\"}"
```

Alternatively, you can also define `hostRules` in a self-hosted [`config.js`](https://docs.renovatebot.com/self-hosted-configuration/) or supply them entirely through environment variables using [`detectHostRulesFromEnv`](https://docs.renovatebot.com/self-hosted-configuration/#detecthostrulesfromenv).

## Learn more

* [Using Renovate with Chainguard Containers](/chainguard/containers/security-and-compliance/updating-containers/renovate/) covers updating references to the Chainguard Containers themselves.
* [Renovate's APK datasource documentation](https://docs.renovatebot.com/modules/datasource/apk/) covers every `registryUrl` query parameter Renovate supports.
* [Authenticating to the Chainguard registry](/chainguard/containers/registry/authenticating/) documents pull tokens and the other authentication options in full.
