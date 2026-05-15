# Codex Sandbox on Windows: Beginner-Friendly Explanation

Source: `codex_sandbox_on_windows_openai.mhtml`

## The Problem in Plain English

Codex is a coding agent that runs commands on your computer. It may read files, edit files, run tests, call Git, use Python, install packages, or invoke build tools.

That is useful because Codex can do real developer work. It is also risky because, by default, a process running as you can usually do almost anything you can do:

- read private files
- modify files outside the project
- change Git internals
- call the internet
- run child programs that inherit the same broad permissions

So Codex needs a sandbox: a controlled execution environment where commands can do enough to be useful, but not enough to freely damage or exfiltrate data.

The desired default behavior is:

- broad read access, so Codex can inspect the project and tools
- write access only to approved places, usually the current workspace
- no internet access unless the user approves it
- restrictions that apply not only to the first command, but also to every child process it launches

On macOS and Linux, there are mature OS primitives that roughly fit this job. Windows did not provide one built-in primitive that cleanly matched "run arbitrary coding-agent commands, allow workspace writes, block network by default."

OpenAI therefore had to build a custom Windows sandbox from smaller Windows security parts.

## Background: The Technologies Involved

### Process

A process is a running program. For example:

- `codex.exe`
- `cmd.exe`
- `powershell.exe`
- `git.exe`
- `python.exe`

When Codex asks the operating system to run a command, Windows creates a process for that command. That process may create more processes. Those are called child processes.

The sandbox must apply to the whole process tree, not only the first command. Otherwise Codex could run `python.exe`, and Python could then open files or sockets outside the intended boundary.

### Sandbox

A sandbox is a restricted environment. The important point is that the operating system, not just Codex code, enforces the restriction.

If the model or a tool tries to do something outside the rules, Windows should block it.

### SID

SID means Security Identifier. It is how Windows represents identities.

A Windows user has a SID. A group has a SID. A login session has a SID. Windows access rules are usually expressed in terms of SIDs.

You can think of a SID as a durable internal ID that Windows uses when deciding:

- who is this process running as?
- which groups does it belong to?
- what files or resources can it access?

### Synthetic SID

A synthetic SID is an identity-like marker that does not correspond to a normal human user account.

OpenAI used a synthetic SID for the sandbox write permission. This let the sandbox mark specific places as writable by the sandbox without pretending that the sandbox was a real person.

In the article, this synthetic identity is described as `sandbox-write`.

### ACL

ACL means Access Control List.

An ACL is the permissions list attached to a file or directory. It says which users, groups, or SIDs can read, write, execute, or delete that object.

For example, a directory ACL can say:

- Simon can read and write
- Administrators can read and write
- `sandbox-write` can write here
- `sandbox-write` cannot write to `.git`

ACLs are powerful but can be expensive to apply recursively across a large project tree.

### Token

A token is a Windows security object attached to a process. It describes the identity and privileges of that process.

When Windows checks whether a process can do something, it looks at the process token and compares it against the target resource's ACL or policy.

### Restricted Token

A restricted token is a token with additional limits.

OpenAI used a specific kind called a write-restricted token. With this token, a write is allowed only if two checks pass:

1. The normal process identity is allowed to write.
2. At least one restricted SID in the token is also allowed to write.

This gives a useful shape for a sandbox:

- the process still has enough identity to behave like a developer tool
- writes only succeed where the sandbox-specific SID has also been granted write permission

### Principal

The principal is the main identity a process runs as. Usually this is the current Windows user.

The final Codex Windows design changed this. Instead of sandboxed commands running as the real user, they run as dedicated local users created by Codex:

- `CodexSandboxOffline`
- `CodexSandboxOnline`

That change is what made strong firewall rules possible.

### Windows Firewall

Windows Firewall can block outbound network access. But firewall rules need something concrete to target, such as a user or program path.

The problem was that Windows Firewall could not target "only processes that include this synthetic sandbox SID in their restricted token."

So OpenAI created dedicated sandbox users and targeted firewall rules at the offline sandbox user.

### UAC and Elevation

UAC means User Account Control. It is the Windows prompt that appears when a program needs administrator privileges.

The first Codex prototype tried to avoid elevation. The final design requires elevation during setup because it must create local users and firewall rules.

Regular Codex use still stays unelevated. The elevated work is isolated into a setup binary.

### DPAPI

DPAPI means Data Protection API. It is a Windows facility for encrypting secrets locally.

Codex needs credentials for the sandbox users it creates. The setup process stores those credentials locally and encrypts them with DPAPI, in a place the sandbox users cannot read.

## What Windows Alternatives Were Considered

### AppContainer

AppContainer is a native Windows sandboxing model. It is strong, but it is designed for apps that know their required capabilities in advance.

That does not fit Codex well. Codex is not one predictable app. It may need to launch arbitrary developer tools depending on the project.

Result: rejected because it was too narrow.

### Windows Sandbox

Windows Sandbox is a disposable, lightweight virtual environment. It gives strong isolation.

But Codex needs to operate on the user's real checkout, real tools, and real environment. A separate throwaway desktop would require a lot of host/guest bridging. It also is not available on Windows Home.

Result: rejected because it was too detached from real developer workflows.

### Mandatory Integrity Control

Windows has integrity levels such as low, medium, and high. A low-integrity process cannot write to normal medium-integrity objects unless those objects are explicitly relabeled.

The idea was: run Codex at low integrity and relabel the workspace so Codex can write there.

The problem is that relabeling the workspace changes the trust meaning of the actual project directory. It would not mean only "Codex can write here." It would mean low-integrity processes in general can write there.

Result: rejected because it changed host filesystem semantics too broadly.

## First Prototype: Unelevated Sandbox

The first prototype tried to avoid admin prompts.

It focused on two controls:

1. filesystem writes
2. network access

### Filesystem Write Control

The prototype used:

- a synthetic SID called `sandbox-write`
- ACLs on the workspace and other configured writable roots
- a write-restricted token for sandboxed commands

The write-restricted token included restricted SIDs such as:

- Everyone
- the current login session SID
- the synthetic `sandbox-write` SID

For a write to succeed, Windows had to agree that both the normal identity and the restricted sandbox SID were allowed.

This let Codex write to:

- the current working directory
- configured `writable_roots`

And it explicitly denied writes to sensitive paths such as:

- `<cwd>/.git`
- `<cwd>/.codex`
- `<cwd>/.agents`

This was a good match for workspace-scoped writes.

### Network Control

The harder part was network blocking without admin rights.

Without elevation, Codex could not install strong Windows Firewall rules. So the prototype tried to make normal developer tools fail closed by changing environment variables and command resolution.

Examples:

- set proxy variables like `HTTPS_PROXY=http://127.0.0.1:9`
- set `ALL_PROXY=http://127.0.0.1:9`
- set Git proxy variables
- set `GIT_SSH_COMMAND=cmd /c exit 1`
- prepend a `denybin` directory to `PATH`
- reorder `PATHEXT` so stub scripts could shadow real network tools like SSH and SCP

This caught many normal tool-driven network calls.

But it was not strong enforcement. A program could ignore proxy variables, bypass `PATH`, or open sockets directly.

Result: good enough to prove direction, not good enough for the final sandbox.

## Why the Prototype Was Not Enough

The unelevated prototype had real advantages:

- no admin setup prompt
- granular file write control
- standard Windows primitives
- useful developer workflow compatibility

But it had serious weaknesses:

- applying ACLs across large workspaces could be slow
- it changed real ACLs on the user's filesystem
- changing sandbox semantics later could require expensive ACL changes
- network blocking was advisory, not strongly enforced

The network weakness was the decisive issue. A sandbox that blocks writes well but lets arbitrary code open sockets directly is not enough for a coding agent that may inspect sensitive local files.

## Final Design: Elevated Sandbox

The final design relaxes the "no elevation ever" goal.

Instead, it uses elevation during setup to create stronger operating-system-level boundaries.

The key change:

Sandboxed commands no longer run as the real Windows user. They run as one of two dedicated local users:

- `CodexSandboxOffline`
- `CodexSandboxOnline`

The offline user is targeted by Windows Firewall rules that block outbound network access.

The online user is used when network access is allowed.

This solved the firewall-targeting problem because Windows Firewall can target a real Windows user account.

## Setup Responsibilities

The elevated setup process does several things:

- creates the synthetic SID if needed
- creates `CodexSandboxOffline` and `CodexSandboxOnline` if needed
- stores their credentials locally, encrypted with DPAPI
- creates or validates firewall rules for `CodexSandboxOffline`
- grants needed read ACLs to the sandbox users
- applies write ACLs for approved writable locations

The read ACL work matters because a new Windows user cannot automatically read the real user's profile directory. Without extra ACLs, Codex would fail on ordinary reads.

To reduce setup delay, some expensive ACL work runs asynchronously.

## Why There Are Multiple Binaries

The final architecture uses separate executables because each one has a different job.

### `codex.exe`

This is the normal Codex harness.

It remains unelevated during normal use. It coordinates the model, the user conversation, and command execution.

### `codex-windows-sandbox-setup.exe`

This handles elevated Windows setup.

It is separate so `codex.exe` does not have to carry all the Windows-only admin setup machinery, and so Codex only crosses the UAC boundary when setup work is actually needed.

### `codex-command-runner.exe`

This runs as the sandbox user.

Its job is to create the final restricted token and launch the actual command.

This binary exists because `codex.exe` could not reliably perform the full sequence itself across the Windows user boundary.

The attempted direct flow was:

1. `codex.exe` calls `LogonUserW(...)` for the sandbox user.
2. `codex.exe` calls `CreateRestrictedToken(...)`.
3. `codex.exe` calls `CreateProcessAsUserW(...)` to launch the child command.

That hit a Windows privilege wall.

The working flow is split:

1. `codex.exe` calls `CreateProcessWithLogonW(...)` to launch `codex-command-runner.exe` as the sandbox user.
2. The runner opens its own token with `OpenProcessToken(...)`.
3. The runner extracts the sandbox login SID with `GetTokenInformation(...)`.
4. The runner creates the restricted token with `CreateRestrictedToken(...)`.
5. The runner launches the actual child process with `CreateProcessAsUserW(...)`.

Because the runner is already running as the sandbox user, it can do the final token restriction and process creation from the correct side of the user boundary.

## Final Architecture

The runtime stack looks like this:

1. `codex.exe`
   - normal unelevated harness
   - decides what command should run

2. `codex-windows-sandbox-setup.exe`
   - elevated setup helper
   - creates users, firewall rules, SIDs, ACLs, and encrypted credentials

3. `codex-command-runner.exe`
   - launched as `CodexSandboxOffline` or `CodexSandboxOnline`
   - creates a restricted token
   - starts the real command

4. Child process
   - actual command such as Git, Python, shell, test runner, or build tool
   - inherits the sandbox boundaries

5. Windows enforcement layer
   - ACLs enforce filesystem write boundaries
   - restricted tokens enforce the extra write check
   - firewall rules enforce offline network blocking

## Challenge-to-Solution Map

| Challenge | Why It Was Hard | How It Was Overcome |
|---|---|---|
| No single Windows sandbox primitive matched Codex | Codex launches arbitrary developer tools and child processes. | OpenAI composed Windows SIDs, tokens, ACLs, local users, firewall rules, and helper binaries. |
| AppContainer was too narrow | It expects a fixed app capability model. Codex needs open-ended developer workflows. | Rejected in favor of a custom sandbox. |
| Windows Sandbox was too detached | Codex needs the real checkout and host environment, not a separate disposable VM. | Rejected because it would hurt practical developer workflow and is unavailable on Windows Home. |
| Mandatory Integrity Control changed workspace trust semantics | Low-integrity relabeling would make the workspace writable by low-integrity processes generally. | Rejected because the security meaning was too broad. |
| Need workspace writes but not arbitrary writes | Full access is unsafe; blocking all writes is unusable. | Used ACLs plus write-restricted tokens and a synthetic sandbox write SID. |
| Need to protect sensitive paths inside writable workspaces | `.git`, `.codex`, and `.agents` may contain important state. | Explicitly denied sandbox write access to those directories. |
| Need no-network mode | Network access can exfiltrate local data. | Final design uses Windows Firewall rules. |
| Firewall could not target a restricted token's synthetic SID | Firewall rules cannot mean "block only this token containing this restricted SID." | Commands run as dedicated sandbox users so firewall rules can target the user. |
| Need both offline and online execution | Some commands should be blocked from the internet; approved commands may need internet. | Created `CodexSandboxOffline` and `CodexSandboxOnline`. |
| New sandbox users could not read the real user's files | Windows users do not automatically read each other's profile directories. | Setup grants read ACLs to sandbox users for common directories. |
| Setup needed admin rights | Creating users and firewall rules requires elevation. | Setup moved into `codex-windows-sandbox-setup.exe`. |
| Process launching crossed a Windows privilege boundary | `codex.exe` could not reliably create a restricted sandbox-user token and launch the child directly. | Added `codex-command-runner.exe`, which runs as the sandbox user and launches the restricted child from there. |

## The Core Design Lesson

The Codex Windows sandbox is complex because the problem is not simply "block a program."

The real requirement is:

- let an AI coding agent behave like a developer
- support arbitrary project tools
- allow useful file reads
- allow controlled workspace writes
- protect sensitive local state
- block internet access by default
- enforce everything through Windows, not through convention
- preserve a reasonable user experience

Windows did not provide one abstraction for that. OpenAI's final design is a layered system where each piece solves one part of the safety/usefulness tradeoff.

