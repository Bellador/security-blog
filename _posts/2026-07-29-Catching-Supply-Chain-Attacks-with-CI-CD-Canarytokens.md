---
title: "Catching Supply-Chain Attacks with CI/CD Canarytokens"
date: 2026-07-29 14:00:00 +0200
categories: [Active Defense, Security Monitoring]
tags: [GitHub, Security Monitoring, CI/CD, Active Defense]
---

Over the past years, supply-chain compromises through malicious or hijacked open-source packages have gone from a niche concern to a recurring headline. The 2024 compromise of the internet-pillar `xz-utils`, in hindsight, was the harbinger of attacks on foundational parts of modern code. This year we observed threat actors pivoting from OS-level dependencies towards code-sharing ecosystems such as GitHub Marketplace, which hosts GitHub Actions, the npm registry, and the Python Package Index (PyPI). This led to incidents such as when, on March 19, 2026, the popular container/filesystem vulnerability scanner [Trivy](https://www.crowdstrike.com/en-us/blog/from-scanner-to-stealer-inside-the-trivy-action-supply-chain-compromise/) was compromised — namely the package `aquasecurity/trivy-action@0.24.0`, which spread credential stealer malware. Similarly, on April 30, 2026, the popular Machine Learning Python package [PyTorch](https://www.bleepingcomputer.com/news/security/backdoored-pytorch-lightning-package-drops-credential-stealer/) `v.2.6.3` was compromised by the "Shai Hulud" worm, which used harvested credentials to spread autonomously to other code repositories. To list one more example among many, in March 2026 the [GlassWorm malware](https://www.aikido.dev/blog/glassworm-returns-unicode-attack-github-npm-vscode) started infecting and spreading to over 433 compromised components (including GitHub Python and JS repos, VSCode extensions, and npm packages). All of these incidents had a similar root cause: a dependency somewhere in the tree gets compromised, and the payload runs with whatever privileges the build process has. On GitHub Actions runners, that's often broad access to secrets, source code, and sometimes deployment credentials — which is why credential stealer malware is so prolific in these attacks.

## Why detection at the runner level is hard

As a Cyber Security Analyst I can say visibility is hard. Modern enterprises have such a diverse tech stack — a mix of on-prem and SaaS deployments — that it creates fragmented enclaves that are hard to integrate into existing security monitoring. For instance, EDR monitoring might seem a solid bet for on-prem servers, but if you try to apply this same boilerplate to the ephemeral nature of GitHub workers, you'll have less success. Runners spin up, execute a job, and disappear within minutes — there is rarely a persistent agent watching process memory, and even if there were, distinguishing malicious credential access from legitimate build tooling reading environment variables is a losing signature game. Static analysis of the dependency tree helps, but it can't catch a payload that's dynamically fetched at install time or hidden behind an obfuscation layer that only unpacks in a CI context.

What credential-stealer malware *does* reliably do, though, is try to confirm that what it stole is actually usable. Tools like [TruffleHog](https://github.com/trufflesecurity/trufflehog) are commonly bundled into these stealers specifically to scan process memory and the filesystem for secret-shaped strings, then automatically call the relevant provider API to check if the credential is live. That validation step is deterministic; it's automated — and, critically, it's an action a legitimate build process never performs on its own AWS credentials mid-pipeline.

> That is the premise on which I built [`canary-in-the-pipeline`](https://github.com/Bellador/canary-in-the-pipeline): a GitHub composite action that plants decoy AWS credentials into the runner and turns any attempt to *validate* them into an alert to your SOC.

## How it works

The action injects a **decoy AWS credential pair** — a canarytoken — into two places before any other job step runs:

- The runner's process environment (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, etc.)
- An on-disk credentials file, matching where real tooling like the AWS CLI would normally look

Both are dynamically generated per run via the [Thinkst Canary](https://canary.tools/) API, so each token can be attributed back to a specific repository and runner ID. Nothing about the token grants real AWS access — it's a live-looking bait object with a Thinkst listener behind it. If any process, human or malicious, calls `sts:GetCallerIdentity` or similar against it, Thinkst fires an alert straight to your SOC.

The important design constraint here: **this only fires on validation, not on exposure.** A stealer that merely dumps environment variables or reads the on-disk file won't trigger anything on its own, and neither will a curious developer who happens to run `env | grep AWS` during debugging (see the [test repo README](https://github.com/Bellador/canary-in-the-pipeline/blob/master/canarytoken-test-repo/README.md) for details). It's tuned specifically for the automated-validation behavior that current credential-stealer families exhibit.

## Repo layout

The project ships as two pieces:

- **[`securitytoken_inject`](https://github.com/Bellador/canary-in-the-pipeline/blob/master/securitytoken_inject/README.md)** — the composite action itself. Pull the API domain and provider-specific call logic into your fork if you're not using Thinkst Canary; the action is written to be adaptable to other suppliers that expose a mass-deployment API.
- **[`canarytoken-test-repo`](https://github.com/Bellador/canary-in-the-pipeline/blob/master/canarytoken-test-repo/README.md)** — a standalone test harness with a `Canarytoken Effectiveness Check` workflow. It runs the injection action, times how long it adds to a build, confirms the env vars and the on-disk file are actually present, and dumps a summary to the GitHub Actions job UI. Useful both for validating a fresh deployment and for keeping an eye on the performance overhead as you roll it out org-wide.

Rollout itself is meant to be boring: deploy the composite action to wherever your org keeps shared Actions, drop the API key in an org-level secret named `SECURITYTOKEN_APIKEY`, expose it to the repos you care about, and reference the action as the first step in each job of your workflows. The cost — both in pipeline seconds and in maintenance — was tested to be low (~4.6s), making this something that scales well and can be deployed across hundreds of repositories.

## Not the "silver bullet"

This form of canarytoken deployment should not be seen as a replacement for anything in a mature AppSec pipeline — it's closer to a cheap, high-signal addition sitting next to SBOM generation, dependency review, and OIDC-based short-lived credentials (which, frankly, remove a lot of the underlying risk this addresses if you can migrate to them). But OIDC migration takes time, legacy pipelines with long-lived static keys aren't going away overnight, and in the meantime a canarytoken costs almost nothing to deploy and produces a near-zero false-positive alert when it does fire. For teams that can't instrument every runner with real telemetry, that trade-off is worth making.

## Further reading

Two writeups pushed me toward this approach in the first place:

- [Tracebit — Detecting CI/CD supply-chain attacks with canary credentials](https://tracebit.com/blog/detecting-cicd-supply-chain-attacks-with-canary-credentials)
- [Grafana Labs — Canary tokens: the unsung heroes of security at Grafana Labs](https://grafana.com/blog/canary-tokens-learn-all-about-the-unsung-heroes-of-security-at-grafana-labs/)

> Code, setup instructions, and the test harness are all in the repo: **[github.com/Bellador/canary-in-the-pipeline](https://github.com/Bellador/canary-in-the-pipeline)**.