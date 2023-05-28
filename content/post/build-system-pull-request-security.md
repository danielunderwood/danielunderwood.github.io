---
title: "Security Concerns of Build Systems and Pull Requests"
date: 2023-05-28T13:59:25-04:00
tags: [cybersecurity, build-systems]
draft: true
mermaid: true
---

tldr: Understand who has permissions to change the code your build system runs
and don't assume read-only access to git repositories is truly read-only if
someone less privileged than the build system can submit a PR.

---

## Introduction

This past week, I ran into a couple things that made me aware of a simple yet
dangerous method of attacking build systems: an external contributor modifying
build pipelines. It's common (for good reason) to run builds and tests to any
change on a repository, including pull requests from external contributors. It's
also common (again, for good reason) to include build configurations within the
repository itself and most build systems will happily build according to the
configuration in the branch being built.

This combination allows a fairly easy attack: an external contributor can open a
PR that modifies code run by a build system, resulting in the ability to execute
arbitrary code on the build system. Given the purpose of build systems, an
attractive target for this code execution is exfiltration of credentials,
especially in environments where continuous deployment is in use.

Open source projects are particularly susceptible to this type of attack, but
private repositories can also have implications here: a user with otherwise
limited permissions may be able to elevate permissions to that of a build
system. While users may be "read-only" to the core repository, they will still
likely have permissions to fork the repository and submit a PR.


## External Configuration is not Safe

It may seem that some of these attacks are limited to the situation where build
configuration lives inside of the code, but this situation can also apply to
manually configured pipelines (and you'll lose the many benefits of
source-controlled configuration).

The exact attack vector will differ from project to project, but it is typical
for a CI system to call code within the repository either as a script
(`build.sh`, `bin/test`) or as an otherwise external command (`pytest`, `make`).
If _any_ of these are called by CI and execute code in the repository, then you
are potentially susceptible to an attack of this variety.

## CI Systems with Protections

At an initial look: this can be quite a difficult attack to mitigate. Source
code protection mechanisms (e.g., CODEOWNERS files) won't work since the attack
may be triggered by opening a PR, so the protection must be implemented by the
build system.

Luckily, it seems that a number of CI systems have protections built in:

- Azure DevOps Pipelines doesn't pass secrets to fork builds by default and
optionally allows requiring approval from project members[^azure-devops-fork-protection] 
- CircleCI does not pass secrets to forked builds by default[^circleci-forked-builds]
- GitHub Actions can require approval to run workflows from forks and defaults
to this option for first-time contributors[^github-actions-fork-workflow-approval]

Note that this isn't an exhaustive review of CI systems with protections; I just
investigated the ones I directly use. You should investigate the ones you use to
determine your exposure!

## A More General Solution: Build Gating

For a general CI system without specific GitHub integrations, builds are
normally triggered by webhooks on the PR event:

<pre class="mermaid">
graph LR
Repo -- Webhook --> CI;
</pre>

This opens up a fairly simple solution: introduce a build gating system to be
the new target of these webhooks. The gating system may then determine if a
build should be allowed based on authors, files changed, approval comments, etc.
and then trigger the CI system to build the code:

<pre class="mermaid">
graph LR
Repo -- Webhook --> CI-gate
CI-gate -- Webhook if Approved --> CI
</pre>

This approach has the added benefit of being capable of additional logic beyond
what a given CI system supports, but may be difficult to implement for build
systems that integrate directly with GitHub rather than via webhooks.

## Conclusion

This attack is something any team managing code, particularly open source code,
should consider. But it's only one step in securing your build pipelines.

As I delved deeper into this issue, I found even more potential attacks on
pipelines[^github-actions-script-injection], and of course we can't ignore that
everyone's build pipeline is probably littered with third-party dependencies (or
entire workflow steps[^github-actions-third-party]) that could be compromised at
any time. This also doesn't even approach the situation where the build system
infrastructure itself is compromised[^cloudflare-pages-assetnote]. So while this
attack in particular has reasonable mitigations, it's just the beginning of the
story of securing your build pipelines.

[^azure-devops-fork-protection]: https://learn.microsoft.com/en-us/azure/devops/pipelines/security/repos?view=azure-devops
[^circleci-forked-builds]: https://circleci.com/docs/oss/#pass-secrets-to-builds-from-forked-pull-requests
[^github-actions-fork-workflow-approval]: https://docs.github.com/en/actions/managing-workflow-runs/approving-workflow-runs-from-public-forks
[^github-actions-script-injection]: https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#understanding-the-risk-of-script-injections
[^github-actions-third-party]: https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#using-third-party-actions
[^cloudflare-pages-assetnote]: https://blog.assetnote.io/2022/05/06/cloudflare-pages-pt1/


