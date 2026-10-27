# uv `build-system.requires` + locked runtime dep — repro

Minimal reproduction for a problem in [renovatebot/renovate#47482](https://github.com/renovatebot/renovate/pull/47482), which surfaces `uv.lock` transitive dependencies for security updates.

## The problem

A package can be both a build requirement and a locked runtime dependency. Here `setuptools` is in `build-system.requires`, and `torch` depends on it at runtime, so `uv.lock` holds it too:

```toml
[project]
dependencies = ["torch>=2.0"]

[build-system]
requires = ["setuptools>=61"]
```

The lock pins `setuptools` 69.5.1, which several advisories affect, including [GHSA-cx63-2mw6-8hw5](https://github.com/advisories/GHSA-cx63-2mw6-8hw5) and [GHSA-5rjg-fvgr-3xxf](https://github.com/advisories/GHSA-5rjg-fvgr-3xxf).

Renovate matches the advisory against the `build-system.requires` entry, since that one carries the locked version:

```jsonc
{
  "packageName": "setuptools",
  "depType": "build-system.requires",
  "currentValue": ">=61",
  "lockedVersion": "69.5.1",
  "updates": [{ "newVersion": "84.1.0", "isLockfileUpdate": true }]
}
```

The same package is not surfaced as a `uv.lock` dependency, because the build requirement already declares it. But `generateCMD()` skipped `build-system.requires` entirely, on the grounds that build requirements aren't locked, so the branch ran:

```
"command": "uv lock "
DEBUG: No files to commit (branch=renovate/pypi-setuptools-vulnerability)
```

No `--upgrade-package` argument at all, nothing to commit, no PR, and the vulnerable version stays in the lock for good.

## After the fix

A build requirement which carries a `lockedVersion` is now upgraded like any other locked dependency:

```
"command": "uv lock --upgrade-package setuptools==84.1.0"
```

which produced [#2](https://github.com/khvn26/renovate-uv-build-system-repro/pull/2), taking `setuptools` 69.5.1 to 84.1.0 in `uv.lock`.

## Reproduce

```bash
git clone https://github.com/khvn26/renovate-uv-build-system-repro
cd renovate-uv-build-system-repro
LOG_LEVEL=debug node /path/to/renovate/lib/renovate.ts --platform=local | grep setuptools
```
