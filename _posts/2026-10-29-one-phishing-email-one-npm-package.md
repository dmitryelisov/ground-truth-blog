---
layout: post
title: "One phishing email, one npm package, every machine"
description: >-
  How the Shai-Hulud worm spreads through npm, what the 2026 defaults in npm 12 and
  pnpm now block, what they still don't, and the two config changes that cover most of the gap.
author: "Teramind Team"
pillar: Security engineering
image: /assets/img/one-phishing-email-cover.png
tags: [npm, supply-chain, security, pnpm, vscode]
faq:
  - q: "What is the Shai-Hulud worm?"
    a: >-
      A self-propagating npm supply-chain worm. It enters through a maintainer whose npm
      credentials were phished. Malicious versions of their packages run code at install time,
      harvest secrets (npm tokens, GitHub tokens, cloud keys) and use the stolen npm token to
      publish infected versions of every other package that maintainer owns. The first wave
      (September 2025) used a postinstall script. The second (November 2025) moved to
      preinstall and added a fallback that destroys the victim's home directory.
  - q: "Does package-lock.json protect me from a compromised npm package?"
    a: >-
      No. A lockfile gives you reproducible builds, not trust. Anything that resolves versions
      fresh pulls the malicious release in: adding or updating a dependency, regenerating the
      lockfile, an automated version bump, `npx`, or a global install.
  - q: "What minimum release age should I set?"
    a: >-
      One to three days is a common choice. The gate only works if someone else spots the
      malicious version before your window ends. Exclude specific packages when you need an
      urgent security fix sooner: `min-release-age-exclude` in npm,
      `minimumReleaseAgeExclude` in pnpm.
  - q: "Does npm 12 make supply-chain worms a solved problem?"
    a: >-
      No. npm 12 (July 2026) stops dependency install scripts from running unless you approve
      them, which closes the install-time path this worm used. It does nothing about malicious
      code that runs when the package is imported, built or tested, about editor extensions,
      or about older npm versions still running in CI images.
  - q: "Should I turn off auto-update for VS Code and Cursor extensions?"
    a: >-
      On developer machines, yes. An extension is a package with full access to your
      workspace, and auto-update installs whatever its publisher ships with no review. Recent
      VS Code builds wait 12 hours before auto-installing an update
      (`extensions.autoUpdateDelay`). That helps, but it is a shorter window than most teams
      give npm packages. Either set `"extensions.autoUpdate": false` and update deliberately,
      or raise the delay to a few days.
---

The entry point is boring. One maintainer gets a convincing phishing email. Their npm credentials go to someone else. A patch release appears, then another, and every machine that resolves a fresh version pulls it in without anyone deciding to.

The defence is just as boring, which is why almost nobody had switched it on. That has started to change: in 2026 the package managers began turning parts of it on by default. That is worth knowing, and it is exactly why it's worth checking what the new defaults still leave open.

We spent one of our internal engineering AMAs taking this attack class apart. This is the version that holds outside the room.

## How the worm actually moves

Shai-Hulud is the name for a family of npm worms. The first wave hit in September 2025, and a second, louder one in November. [Unit 42's write-up](https://unit42.paloaltonetworks.com/npm-supply-chain-attack/) is the best public account. The mechanics:

1. **Steal a maintainer.** Phish the npm credentials of someone who publishes packages people depend on.
2. **Publish a poisoned release.** Add a lifecycle script to a new patch version. The first wave used `postinstall`; the November wave moved to `preinstall`, so the code ran before the install had even finished.
3. **Harvest.** On the victim's machine or CI runner, read `.npmrc` for npm tokens, scan environment variables and config files for GitHub tokens and AWS, GCP and Azure keys.
4. **Exfiltrate in public.** Create a new public GitHub repository under the victim's own account and commit the stolen secrets to it. A repo you didn't create, appearing on your account, is the alarm.
5. **Spread.** Log in to npm with the stolen token, list every other package the victim maintains, inject the same payload, publish. Use stolen GitHub tokens to push workflows into the organisation's repositories, so the compromise moves into CI.
6. **Burn the house down if it fails.** In the November wave, if the worm could not steal credentials or set up exfiltration, it tried to overwrite and delete every writable file in the user's home directory.

<figure>
  <img src="{{ '/assets/img/shai-hulud-flow.svg' | relative_url }}" alt="The Shai-Hulud loop: phish a maintainer, publish a poisoned patch, a fresh install runs it, harvest secrets, exfiltrate them to a public repo, spread through the victim's other packages and CI, and back to step two. A release-age gate sits before the install; npm 12 and pnpm turn install scripts off by default. If nothing can be stolen, the November 2025 wave wipes the home directory.">
  <figcaption>The loop. Each stolen token is a new entry point. The green markers are the two places the 2026 defaults now cut in.</figcaption>
</figure>

None of it is clever. Every step uses a capability the developer's machine already grants to any package it installs.

And it is not over. Trackers counted further waves through 2026 — SAP-ecosystem packages in April, TanStack in May, the "ChainDrop" variant across hundreds of packages in August — and new compromised releases as recently as [8 October](https://brco.dev/insights/updates/npm-supply-chain-attacks-2026/).

## Why your lockfile doesn't save you

The first thing someone says in this conversation is "we have a `package-lock.json`". We had the same debate.

A lockfile pins what was resolved the last time you resolved. It gives you **predictable builds, not safety**. The poisoned version gets in the moment anything resolves fresh:

- someone runs `npm install some-new-thing` and the range admits the new patch;
- the lockfile is regenerated to fix a conflict;
- a dependency bot opens a version bump and it gets merged;
- `npx` or a global install fetches "latest";
- a transitive dependency of a tool you run once pulls in the infected package.

On a team of any size, at least one of those happens every day.

## Change one: make brand-new versions wait

Every malicious release has the same shape: it is *new*. It was published minutes or hours ago. The security vendors, the registry and the rest of the ecosystem need some time to spot it and pull it.

So don't install anything that young. Both npm and pnpm now support a release-age gate.

**npm** (recent 11.x releases and npm 12), in `.npmrc` — the value is in days:

```ini
min-release-age=3
# let a specific package through when you need an urgent fix
min-release-age-exclude=some-package
```

**pnpm**, in `pnpm-workspace.yaml` — the value is in minutes:

```yaml
minimumReleaseAge: 4320   # 3 days
minimumReleaseAgeExclude:
  - some-package
```

This is what it looks like when it works. We set a three-day gate and asked npm for a TypeScript nightly published the day before:

<figure>
  <img src="{{ '/assets/img/npm-min-release-age.png' | relative_url }}" alt="Terminal: with min-release-age=3 in .npmrc, npm install typescript@7.1.0-dev.20261027.1 fails with ETARGET, no matching version with a date before 10/25/2026. npm install typescript@next succeeds and npm ls shows typescript@7.0.2.">
  <figcaption>Real output, npm 11.17, 2026-10-28. One line, the path to the local debug log, is removed.</figcaption>
</figure>

Two things to notice. An exact version that is too young fails loudly, with `ETARGET`. A tag or a range does not fail. npm quietly picks something old enough, so here `@next` came back as `7.0.2` from July. That is the behaviour you want, but it means the gate can silently hold back an update you were expecting.

Current pnpm documents a default of 1440 minutes (one day). Older pnpm versions shipped with the gate off, so set it explicitly rather than trusting whatever version your CI image has. Yarn and Bun have equivalents with their own names and units; check the docs for the version you run.

The trade-off is real: a security fix also waits. That is what the exclude list is for. One to three days is a common window; the right number is "longer than it takes someone else to notice".

## Change two: stop your editor updating itself

This is the one almost nobody has thought about.

An editor extension is a package. It runs with access to your workspace, your terminal and whatever tokens are lying around in it. VS Code and Cursor both **auto-update extensions by default**. That means a compromised extension release becomes code execution on every developer machine, with no lockfile and no review.

VS Code has started to hedge. Recent builds wait 12 hours after an update is published before installing it automatically:

<figure>
  <img src="{{ '/assets/img/vscode-extensions-auto-update.png' | relative_url }}" alt="VS Code settings: Extensions: Auto Update is set to on; Extensions: Auto Update Delay is 12 hours, with the description that it helps avoid installing potentially problematic updates immediately after release.">
  <figcaption>The defaults today: auto-update on, with a 12-hour delay. Screenshot from vscode.dev, 2026-10-28.</figcaption>
</figure>

Twelve hours is a release-age gate, just a short one. That is less than most teams would accept for an npm package. Pick one of two settings in `settings.json`:

```jsonc
{
  // Strictest: nothing updates until you decide it should
  "extensions.autoUpdate": false,

  // Or keep auto-update and give the ecosystem three days to notice
  "extensions.autoUpdateDelay": 72
}
```

Cursor is built on VS Code and reads the same extension settings. Check that your version has the delay setting before you rely on it. If you turn auto-update off, update extensions deliberately, the way you'd bump a dependency. That is the same friction you already accept everywhere else in the supply chain.

## What the 2026 defaults fixed — and what they didn't

The ecosystem did move. **npm 12**, generally available on 8 July 2026, stops `preinstall`, `install` and `postinstall` scripts from dependencies from running unless you approve them, and blocks git and remote-URL dependencies by default ([Socket's summary](https://socket.dev/blog/npm-12)). Current **pnpm** refuses unreviewed build scripts and fails the install by default (`strictDepBuilds`), with an explicit `allowBuilds` list for the ones you trust.

That closes the exact path both Shai-Hulud waves used. It is a genuine improvement. It also leaves four gaps:

1. **Code that runs on import.** Install scripts were the easy path, not the only one. A payload in the module body runs the first time your test suite, dev server or build imports the package. Turning off install scripts moves the moment of execution; it doesn't remove it.
2. **The npm in your CI image.** The default only applies where npm 12 actually runs. Images pinned to npm 10 or 11, old `node` base images and global tools keep the old behaviour. And npm 12 *skips* an unapproved script with a warning; the install still succeeds. If you want CI to fail instead, turn on `strict-allow-scripts`.
3. **Editor extensions.** None of the package-manager changes touch them, and VS Code's own 12-hour delay is short. See change two.
4. **Stolen tokens.** Once a worm has a GitHub token, it doesn't need npm at all — it can push a workflow. Trusted publishing proves *which* workflow published a package, not that the workflow is honest.

The old rule was "assume install-time code runs". The updated one is: **assume the package's code runs, at some point, on a machine with secrets on it.** Then make that machine boring to compromise.

## Make the developer machine a poor target

The worm's whole economy is secrets sitting on developer machines and CI runners. Every one you remove is one fewer it can spread with.

- **Passphrase your SSH keys.** An unprotected key on an infected machine is not just a leaked credential; it is a propagation path into every server and repository it can reach.
- **Clean up tokens in plain text.** `.npmrc`, `.env` files and, increasingly, AI tooling: MCP server configs and markdown notes with API keys pasted in are exactly what a secret scanner finds first.
- **Use a disposable environment for anything untrusted.** Trying out an unfamiliar package? Do it in a cloud dev environment you can throw away, not on the laptop that holds your production credentials.
- **Prefer scoped, short-lived tokens** for publishing and CI over long-lived personal ones.

## When it happens anyway

Prevention reduces the odds; it doesn't make them zero. What decides how bad the day gets is how fast the first hour goes.

- **Write the "I think I'm infected" page before you need it.** Who to tell, what to do first, which tokens to revoke. Nobody reads documentation well while their machine is doing something strange.
- **Suspend and revoke first, investigate second.** Disable the affected accounts, revoke every token the machine could have touched — npm, GitHub, cloud, CI — then work out what happened.
- **Wipe, don't clean.** A machine that ran a worm gets reinstalled. You cannot prove you found everything, so don't try.
- **Make it blameless, and mean it.** The person who clicked the link is the one you most need talking to you in the first ten minutes. A culture that hunts for someone to blame buys silence, and silence costs you hours you don't have.

## The cheat sheet

| Change | Where | Effort |
|---|---|---|
| Release-age gate (1–3 days) | `.npmrc` or `pnpm-workspace.yaml` | 2 minutes |
| Extension auto-update off, or a delay of 72 hours or more | editor `settings.json` | 1 minute |
| npm 12 or current pnpm in CI, fail on unapproved scripts | CI images, `strict-allow-scripts` / `strictDepBuilds` | an afternoon |
| Passphrases on SSH keys; tokens out of plain-text files | every developer machine | an hour per person |
| Untrusted packages tried in a disposable environment | team habit | none, once it's normal |
| An "I think I'm infected" page | team wiki | an hour, once |

The first two rows take three minutes and close the paths this worm family has used. Do them today. The rest is how you make the next variant boring.
