# Submodule Credentials Buildkite Plugin

Gives git a token for one private GitHub organization, for the whole of a
Buildkite job.

## Why this plugin exists

A repository that has a submodule in a private GitHub organization cannot clone
that submodule during checkout, because the agent has no credential for it. The
usual solution is the agent variable `BUILDKITE_GIT_SUBMODULE_CLONE_CONFIG`,
which adds a `url.<with-token>.insteadOf=<plain-url>` rewrite to the
`git submodule update` command.

Buildkite agent **3.133.0** and later treat that variable as *protected*. A
pipeline that sets it gets this warning, and the variable is dropped:

```
~~~ Detected protected environment variables
# Your pipeline environment has protected environment variables set. These can
# only be set via hooks, plugins or the agent configuration.
⚠️ Warning: Ignored BUILDKITE_GIT_SUBMODULE_CLONE_CONFIG
```

Agent 3.104.0 still accepted it. This plugin restores the behaviour without a
change to the agent configuration, because a plugin `environment` hook runs in
the "Preparing plugins" phase, before "Preparing working directory", and is
permitted to set protected variables.

## What the plugin sets

The plugin builds one rewrite rule and writes it into two variables.

| Variable | Read by | Covers |
|---|---|---|
| `BUILDKITE_GIT_SUBMODULE_CLONE_CONFIG` | the agent | the `git submodule update` of the checkout phase |
| `GIT_CONFIG_PARAMETERS` | git itself | every later git command of the job, and a git command in a container, if the container plugin passes the variable through |

The second variable is necessary because the agent gives the rule to the
checkout only. A command that the job runs later — for example a `git fetch`
inside the submodule, from a script that runs in a container — starts with no
credential, and GitHub answers `403`. `GIT_CONFIG_PARAMETERS` is standard git,
not Buildkite, and git applies it in every process that inherits it.

If the job runs in a container, add `GIT_CONFIG_PARAMETERS` to the environment
that the container plugin passes through:

```yaml
- plugins:
    - docker#v3.5.0:
        environment:
          - GIT_CONFIG_PARAMETERS
```

The rule stays in the environment. It is not written to any file, so it does
not reach `.git/config`, and it does not reach a remote URL.

## Usage

```yaml
steps:
  - command: "make test"
    plugins:
      - MinaProtocol/submodule-credentials#v1.1.0:
          org: "o1-labs"
          token-env: "GH_SUBMODULE_TOKEN"
```

The plugin holds no token. It reads one from the environment variable named by
`token-env`, and builds the rewrite rule from it.

## Configuration

| Option | Default | Meaning |
|---|---|---|
| `org` | `o1-labs` | GitHub organization whose URLs receive the token. |
| `token-env` | `GH_SUBMODULE_TOKEN` | Name of the variable that holds the token. |

The token needs **read** access to the private repositories. If the
organization uses SAML SSO, authorize the token for that organization.

If the named variable is empty, the plugin prints a message and makes no
change. Neither variable is touched. A pipeline that has only public
submodules therefore keeps working, and can carry the plugin with no effect.

An existing value of either variable is kept. The plugin appends its rule:
with a comma for `BUILDKITE_GIT_SUBMODULE_CLONE_CONFIG`, and with a space and
single quotes for `GIT_CONFIG_PARAMETERS`, which is the format git expects.

## Keep the token out of the logs

Name the variable with a `_TOKEN` suffix. The agent redacts the values of
variables that match its `redacted-vars` patterns (`*_TOKEN`, `*_SECRET`,
`*_PASSWORD`, and others) wherever they occur in the log, including inside the
two variables that this plugin sets.

## This repository must be public

The agent clones the plugin before it clones your repository. A private plugin
repository would itself need a credential, which is the problem this plugin
solves.
