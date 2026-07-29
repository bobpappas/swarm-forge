# Local Customizations Decision Record

Status: archived customization record; there is no current fork or
reimplementation plan.

Recorded: 2026-07-29

## Current Decision

The user has decided not to use this fork or reimplement the local features
described below for now because `unclebob/swarm-forge` already provides
everything needed for the intended workflow.

Future projects can install the desired upstream `two-pack`, `four-pack`, or
`six-pack` branch and start it with the project's local `./swarm` wrapper. To
work entirely from VS Code without opening external terminal windows, start it
from a VS Code integrated terminal with:

```sh
SWARMFORGE_TERMINAL=none ./swarm
```

Upstream creates all configured tmux sessions, starts the agents and handoff
daemon, and attaches the invoking VS Code terminal to the first role. The user
can keep that attachment or detach with tmux's `Ctrl-b`, then `d`, and attach
other sessions from additional VS Code terminals.

The old local `--no-terminal` implementation differed only by returning to the
invoking shell without making the initial attachment. That mechanical
difference is not required for the user's actual goal. Upstream's existing
`SWARMFORGE_TERMINAL=none` behavior therefore satisfies the feature need.

No local runtime feature currently needs to be reimplemented. This document
preserves the old changes and analysis only in case the decision is revisited.

The older `swarm-forge-local-tool-handoff.html` file remains useful as historical
context about why a reusable local SwarmForge checkout was created. It is not the
authoritative specification of these customizations: it describes an older
repository layout and contains assumptions that upstream has since changed.

## Repository Baseline

Remotes at the time of this record:

- `origin`: `https://github.com/bobpappas/swarm-forge.git`
- `upstream`: `https://github.com/unclebob/swarm-forge.git`

Local branch state:

- Branch: `main`
- Local and `origin/main`: `e2170c1784568aeb390dda87604b4efecbe6db14`
- Working tree: clean before this documentation was added

The local work starts from:

- Commit: `afa322bee192f7bc8f24041a39780d45fc29488b`
- Subject: `Prefer Java test runners over Maven tests`
- Date: 2026-06-03

The two local-only commits are:

1. `5d20c27bd61f0606f2fe4543d91f9c64579b2a15`
   (`added my modifications`)
2. `e2170c1784568aeb390dda87604b4efecbe6db14`
   (`fixing other issues running swarm from a $PATH`)

Together they changed four files relative to the common base:

- Modified `README.md`
- Modified `swarm`
- Modified `swarmforge.sh`
- Added `swarm-forge-local-tool-handoff.html`

The aggregate local delta is 269 insertions and 6 deletions.

The upstream comparison used during this record was:

- `upstream/main`: `9acd54d2239fef7e41ddacd8fd30dfb0e69672fe`
- Subject: `Add top-level close-swarm script to stop a running swarm.`
- Date: 2026-07-10
- Divergence at inspection time: 2 local-only commits and 166
  upstream-only commits

The remote refs were fetched again on 2026-07-29 and remained at the revisions
recorded here. Upstream may advance later and should be checked before installing
a pack or revisiting any fork work.

## Selected Upstream Workflow

The earlier local changes were designed around one globally installed
SwarmForge checkout. That is no longer the desired workflow.

The selected workflow is:

1. Use `unclebob/swarm-forge` directly rather than maintaining the fork.
2. Treat each upstream pack branch as an independently installable project
   template:
   - `two-pack`: one swarm with two roles;
   - `four-pack`: one swarm with four roles; and
   - `six-pack`: one swarm with six roles.
3. For each future project, download or extract the desired pack branch into
   the project rather than cloning a nested SwarmForge repository.
4. Start that project's swarm with its project-local `./swarm` wrapper.
5. Use `SWARMFORGE_TERMINAL=none` when working through VS Code.
6. Allow different projects to choose different pack sizes.

Supporting all three packs means making all three templates available for
independent project use. It does not mean running multiple pack configurations
simultaneously in one project's shared `.swarmforge` runtime state.

## Historical Fork Distribution Analysis

At the inspected upstream revisions, every pack wrapper defaults to downloading
shared scripts from:

```text
https://github.com/unclebob/swarm-forge/archive/refs/heads/main.tar.gz
```

The wrappers allow `SWARMFORGE_SCRIPTS_URL` to override that address. If the fork
plan were resumed, its pack wrappers would need to obtain shared scripts from
the fork's `main` by default. This is no longer an active requirement because
the selected workflow uses Uncle Bob's branches and shared scripts directly.

## Customization 1: PATH- and Symlink-Safe Launcher

Decision: historical only; do not reimplement.

### Motivation

The original launcher located `swarmforge.sh` relative to `$0`. When the
`swarm` command was reached through a symlink on `PATH`, that could resolve the
tool directory incorrectly and look for `swarmforge.sh` beside the symlink
instead of inside the SwarmForge checkout.

### Implemented behavior

The local `swarm` script:

1. Starts with the path in `$0`.
2. Follows every symlink in the chain.
3. Supports both absolute and relative symlink targets.
4. Resolves relative targets relative to the directory containing the current
   symlink.
5. Computes the tool directory from the final, non-symlink target.
6. Executes `swarmforge.sh` from that resolved tool directory.
7. Forwards all original arguments unchanged.

### Reimplementation decision

The agreed project-local `./swarm` workflow removes the need for a global
launcher, PATH installation, or user-level symlink. The symlink-resolution code
must remain recoverable from the archival branch but should not be ported into
the new upstream structure.

If a global launcher is requested again in the future, it should be evaluated as
a new feature rather than assumed to be part of this synchronization.

## Customization 2: Project Discovery From the Current Directory

Decision: historical only; do not reimplement.

### Motivation

When `swarm` is installed globally, the launcher script's directory is the tool
checkout, while the current directory is the project being operated on. The
user also needs to invoke the command from a subdirectory of a configured
project.

### Implemented behavior

When no working-directory argument is supplied, the local launcher:

1. Converts the current directory to an absolute path.
2. Checks that directory and then each parent directory.
3. Selects the first directory containing
   `swarmforge/swarmforge.conf`.
4. If no configured ancestor is found, falls back to the original current
   directory.

When an explicit working-directory argument is supplied, it is converted to an
absolute path and used directly. Upward discovery is not performed for an
explicit argument.

### Reimplementation decision

Automatic ancestor discovery existed to support the global command. With a
project-local wrapper, the user starts the swarm from the project using
`./swarm`; the wrapper already knows its project location.

Do not port upward discovery or the old implicit fallback. An explicit
working-directory argument may continue to be supported if upstream supports
it, but no fork-specific discovery behavior is required.

## Customization 3: Detached or No-Terminal Mode

Decision: do not reimplement; upstream satisfies the user goal.

### Motivation

The user wants SwarmForge to create and start all tmux sessions without:

- opening Terminal.app, Ghostty, Windows Terminal, iTerm2, or another terminal
  surface; and
- attaching the shell that invoked `swarm` to one of the tmux sessions.

The sessions are intentionally left running so the user can attach to them from
VS Code or another tmux-capable interface.

### Implemented interfaces

The local implementation accepts:

```text
swarm --no-terminal [working-directory]
```

It also accepts:

```text
SWARMFORGE_NO_TERMINAL=1 swarm [working-directory]
```

The command-line flag sets the same effective mode as the environment variable.

The old implementation treats the environment variable as enabled only when
its value is exactly `1`.

### Implemented behavior

In no-terminal mode, SwarmForge still:

- validates dependencies and configuration;
- prepares the repository and worktrees;
- creates the tmux sessions;
- starts every configured agent;
- writes normal runtime/session state;
- prints the session names and a manual tmux attachment command; and
- retains the normal cleanup behavior associated with the cleanup-owner agent.

It does not:

- open terminal windows, tabs, or panes through a terminal adapter;
- start the terminal-window watchdog; or
- attach the invoking shell to the cleanup session.

The command returns control to the invoking shell after startup.

### Distinction from terminal backend `none`

This distinction is essential.

In both the old local implementation and the inspected upstream implementation,
the `none` terminal adapter reports that it cannot open sessions. The standard
fallback for a backend that cannot open sessions is to attach the current shell
to the cleanup session.

Therefore:

```text
SWARMFORGE_TERMINAL=none
```

is not equivalent to:

```text
swarm --no-terminal
```

The former attaches the invoking shell. The latter deliberately returns without
opening or attaching any terminal surface.

### Final assessment

The original motivation was to use SwarmForge's tmux sessions from VS Code
without opening external terminal windows. Upstream already supports that goal:

```sh
SWARMFORGE_TERMINAL=none ./swarm
```

When this is run in a VS Code integrated terminal, upstream attaches that
terminal to the first role after startup. Detaching with tmux's `Ctrl-b`, then
`d`, returns to the shell without stopping any agent or tmux session. Additional
VS Code terminals can attach to the remaining role sessions.

Returning without an initial attachment was an implementation detail of the
fork's `--no-terminal` option, not a required user outcome. No replacement
option or launcher change is needed.

## Customization 4: Command-Line Validation and Help

Decision: do not reimplement.

The local `swarmforge.sh` gained a small command-line interface as part of the
PATH work.

Supported syntax:

```text
swarm [--no-terminal] [working-directory]
```

Supported options:

- `--no-terminal`
- `-h`
- `--help`
- `--` to end option parsing

The local implementation:

- rejects unknown options;
- rejects more than one positional working-directory argument;
- prints usage on help and argument errors; and
- only parses options before the first positional argument.

### Reimplementation decision

The PATH-oriented CLI, working-directory discovery, and `--no-terminal` parser
do not need to be preserved. The selected upstream workflow uses the
project-local wrapper and the existing `SWARMFORGE_TERMINAL=none` environment
setting.

## Documentation Changes in the Local Commits

The local README was changed to:

- describe adding the `swarm` launcher, rather than `swarmforge.sh`, to
  `PATH`;
- show `swarm <working-directory>` and project-local `./swarm` usage; and
- document `SWARMFORGE_NO_TERMINAL=1` and `--no-terminal`.

Those paragraphs target the old repository structure and should not be copied
into the current upstream README. PATH installation, automatic project
discovery, and the custom `--no-terminal` option are all retired. Current usage
should follow upstream's pack-installation and
`SWARMFORGE_TERMINAL=none` documentation.

## Historical HTML Handoff

`swarm-forge-local-tool-handoff.html` was added in the first local commit. It
records the earlier plan to move enhanced SwarmForge behavior out of a
project-installed copy and into this reusable fork.

Useful historical intent in that document includes:

- maintaining one reusable local SwarmForge tool checkout;
- avoiding duplicated vendored tool copies across projects;
- keeping project-specific prompts and configuration in their project repos;
- keeping generated runtime state out of Git.

Known stale details include:

- the recommendation to place a stable launcher on `PATH`;
- old role-prompt locations;
- old constitution layout;
- references to `swarmtools`, `logs`, and `agent_context`;
- assumptions that the old `main` branch is directly runnable; and
- an open question about whether the fork exists, which this repository now
  answers.

The HTML file should be preserved with the archival branch. It may later be
updated or retired, but it should not be used as the sole reimplementation
guide.

## Current Upstream Architecture Assessment

The inspected `upstream/main` has substantially diverged from the local base.

Observed changes relevant to these customizations:

- `main` now carries shared documentation, scripts, constitution articles, and
  tests rather than a complete runnable project configuration.
- Runnable configurations are maintained on `two-pack`, `four-pack`, and
  `six-pack` branches.
- Shared orchestration moved under `swarmforge/scripts/`.
- The main launcher logic moved from a large Zsh script to
  `swarmforge/scripts/swarmforge.bb`, with a small shell wrapper.
- Role prompts moved under `swarmforge/roles/`.
- Runtime handoffs are daemon-backed.
- Pack-level `./swarm` wrappers bootstrap shared scripts from `main` when
  needed.
- Terminal adapters still use the same basic capability boundary: when the
  selected adapter cannot open sessions, the launcher attaches the current
  shell.

At the inspected upstream revision:

- No `--no-terminal` option was found in the shared launcher.
- No `SWARMFORGE_NO_TERMINAL` behavior was found.
- `SWARMFORGE_TERMINAL=none` still leads to current-shell attachment.
- The Babashka entry point treats its first ordinary argument as the working
  directory; it does not provide the local option parser.
- No upward search for `swarmforge/swarmforge.conf` was found.
- Upstream `main` no longer contains the old top-level `swarm` launcher.
- Each inspected pack branch contains its own project-local `./swarm`
  bootstrap wrapper.

These findings are provisional and must be repeated against the newest upstream
revision if the fork plan is ever resumed.

Current-shell attachment is acceptable and useful when the invoking shell is a
VS Code integrated terminal. The absence of a fully detached option is
therefore not a missing requirement.

## Remaining Feature Assessment

The complete diff of the two local commits was reviewed, not just their commit
messages. No additional runtime customization was found beyond:

1. PATH and symlink-safe launcher resolution;
2. automatic project-root discovery;
3. detached/no-terminal startup;
4. command-line handling added to expose those behaviors; and
5. related README and HTML documentation.

Under the agreed project-local pack workflow:

- Items 1 and 2 are obsolete.
- Item 4 is obsolete.
- Item 5 should be preserved as history and replaced with current instructions,
  not treated as runtime functionality.
- Item 3 is satisfied at the user-goal level by upstream's existing
  `SWARMFORGE_TERMINAL=none` behavior.

Upstream already provides the major lifecycle features needed around detached
startup:

- project-local pack wrappers;
- tmux session creation and recorded session state;
- manual attachment information;
- daemon-backed handoffs; and
- a top-level `close-swarm` command that can stop sessions and the handoff
  daemon from recorded project state.

Consequently, none of the original local features currently needs to be
reimplemented, and the fork does not need to be maintained for this workflow.

## Feature Disposition

| Behavior | Decision | Reason |
| --- | --- | --- |
| Preserve the old Zsh launcher code | Do not port | Upstream replaced its architecture. Preserve behavior, not implementation. |
| Detached/no-terminal startup | Use upstream | `SWARMFORGE_TERMINAL=none` attaches the VS Code terminal to tmux and satisfies the actual use case. |
| Global `swarm` command on `PATH` | Retire | Each project will use the selected pack's local `./swarm`. |
| Symlink-chain resolution | Retire | It existed only for the global PATH launcher. |
| Upward project-root discovery | Retire | The project-local wrapper identifies the project without ancestor discovery. |
| Support `two-pack` | Use upstream branch | Future projects can install the upstream two-role template directly. |
| Support `four-pack` | Use upstream branch | Future projects can install the upstream four-role template directly. |
| Support `six-pack` | Use upstream branch | Future projects can install the upstream six-role template directly. |
| Point pack bootstrapping at fork `main` | No longer needed | Projects will use Uncle Bob's pack branches and shared `main`. |
| Old README wording | Do not port verbatim | It describes paths and structure that no longer exist upstream. |
| Historical HTML handoff | Preserve/archive | Useful provenance, but stale as current documentation. |

## Upstream Workflow Check

Before discarding the fork as a working copy, perform one practical check with
a fresh upstream pack:

1. Install the desired upstream pack in a test project.
2. From a VS Code integrated terminal, run
   `SWARMFORGE_TERMINAL=none ./swarm`.
3. Confirm that all expected role sessions start.
4. Confirm that the VS Code terminal attaches to the first role.
5. Detach with `Ctrl-b`, then `d`.
6. Use the recorded `.swarmforge/tmux-socket` to list and attach to other role
   sessions from additional VS Code terminals.
7. Confirm that upstream `close-swarm` stops the sessions and handoff daemon.

This is validation of the selected upstream workflow, not a prerequisite for
porting fork code.

## Resolved Decisions

1. Future projects may use `two-pack`, `four-pack`, or `six-pack`.
2. These are used directly from Uncle Bob's upstream repository and are not
   maintained in the fork.
3. A future project chooses one pack when installing SwarmForge.
4. Projects use their local `./swarm` wrapper.
5. The global PATH launcher is retired.
6. Symlink resolution and upward project discovery are not reimplemented.
7. `SWARMFORGE_TERMINAL=none` provides the required VS Code/tmux workflow.
8. No local runtime feature is scheduled for reimplementation.

## If the Decision Is Revisited

If a future workflow specifically requires swarm startup to return without even
the initial tmux attachment, reassess upstream first. Only then consider a
detached/headless option. Preserve the old commits and this record as historical
input, but do not merge the obsolete Zsh implementation into the current
Babashka launcher.
