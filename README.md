# Submodule Credentials Buildkite Plugin

Gives the Buildkite agent a token for private GitHub submodules.

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

## Usage

```yaml
steps:
  - command: "make test"
    plugins:
      - MinaProtocol/submodule-credentials#v1.0.0:
          org: "o1-labs"
          token-env: "GH_SUBMODULE_TOKEN"
```

The plugin holds no token. It reads one from the environment variable named by
`token-env`, and builds the rewrite rule from it.

## Configuration

| Option | Default | Meaning |
|---|---|---|
| `org` | `o1-labs` | GitHub organization whose submodule URLs receive the token. |
| `token-env` | `GH_SUBMODULE_TOKEN` | Name of the variable that holds the token. |

The token needs **read** access to the private submodule repositories. If the
organization uses SAML SSO, authorize the token for that organization.

If the named variable is empty, the plugin prints a message and makes no change.
Branches that use only public submodules therefore keep working.

An existing `BUILDKITE_GIT_SUBMODULE_CLONE_CONFIG` is kept; this plugin appends
its rule to it, separated by a comma.

## Keep the token out of the logs

Name the variable with a `_TOKEN` suffix. The agent redacts the values of
variables that match its `redacted-vars` patterns (`*_TOKEN`, `*_SECRET`,
`*_PASSWORD`, and others) wherever they occur in the log, including inside the
`git -c url....insteadOf=...` command line that the checkout prints.

## This repository must be public

The agent clones the plugin before it clones your repository. A private plugin
repository would itself need a credential, which is the problem this plugin
solves.
