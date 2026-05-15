# Codex Windows Sandbox Tutorial

This repository is a beginner-friendly explanation of OpenAI's Codex sandbox design for Windows.

Original article: [Building a safe, effective sandbox to enable Codex on Windows](https://openai.com/index/building-codex-windows-sandbox/)

## Infographic

![Codex Windows Sandbox Architecture](codex_windows_sandbox_architecture.png)

## What This Explains

Codex is a coding agent that can run real commands on a developer's computer. That is useful, but it creates a security problem: a process running as the user may be able to read files, edit files, launch child processes, and access the internet.

The Windows sandbox design tries to make Codex useful without giving every command unrestricted access.

The main goals are:

- let Codex read enough context to understand a project
- allow writes only in approved workspace locations
- block internet access by default
- apply the same restrictions to child processes
- preserve a practical developer workflow

## The Core Challenge

Windows did not provide one built-in sandbox primitive that cleanly matched Codex's needs.

Codex is not a single predictable app. It may launch:

- shells
- Git
- Python
- package managers
- test runners
- build tools
- arbitrary project-specific binaries

So OpenAI had to compose several Windows security mechanisms instead of using one simple sandbox switch.

## Key Windows Concepts

| Concept | Plain-English Meaning |
|---|---|
| Process | A running program, such as `git.exe`, `python.exe`, or `codex.exe`. |
| Child process | A program launched by another program. Sandbox rules must follow the whole process tree. |
| SID | A Windows security identity. Users, groups, and sessions are represented by SIDs. |
| ACL | A file or directory permission list. It says who can read, write, execute, or delete. |
| Token | The permission badge attached to a running process. Windows checks this when the process tries to do something. |
| Restricted token | A token with extra limits. Codex uses this to require an additional sandbox-specific permission check for writes. |
| Windows Firewall | The OS feature used in the final design to block outbound network access. |
| UAC / elevation | The Windows admin prompt used when setup needs privileged operations. |
| DPAPI | Windows' local encryption API, used to protect stored sandbox user credentials. |

## Design Evolution

### 1. Existing Windows Tools Were Not Enough

OpenAI evaluated several Windows options:

- **AppContainer**: strong isolation, but too narrow for arbitrary developer tools.
- **Windows Sandbox**: strong VM-like isolation, but too detached from the user's real checkout and unavailable on Windows Home.
- **Mandatory Integrity Control**: promising on paper, but it would change the trust semantics of the user's actual workspace.

### 2. First Prototype: Unelevated Sandbox

The first prototype avoided admin setup.

It used:

- a synthetic sandbox write SID
- ACLs on writable directories
- write-restricted tokens
- environment-variable tricks to discourage network access

This worked reasonably well for file writes, but network blocking was weak. Programs could ignore proxy variables, bypass `PATH`, or open sockets directly.

### 3. Final Design: Elevated Sandbox

The final design accepts an elevated setup step so Windows can enforce stronger boundaries.

It creates two dedicated local users:

- `CodexSandboxOffline`: used when network access should be blocked
- `CodexSandboxOnline`: used when network access is allowed

Windows Firewall can target the offline sandbox user and block outbound traffic.

## Final Architecture

The final architecture has four main executable layers:

1. `codex.exe`
   - normal unelevated Codex harness
   - coordinates the conversation and command execution

2. `codex-windows-sandbox-setup.exe`
   - elevated setup helper
   - creates sandbox users
   - creates firewall rules
   - applies ACLs
   - stores credentials encrypted with DPAPI

3. `codex-command-runner.exe`
   - runs as the sandbox user
   - creates the restricted token
   - launches the actual command

4. Child command
   - the real shell, Git, Python, test runner, or build tool
   - inherits the sandbox restrictions

## Challenge and Solution Summary

| Challenge | How It Was Overcome |
|---|---|
| Windows lacked one Codex-shaped sandbox primitive | Compose SIDs, ACLs, restricted tokens, local users, firewall rules, and helper binaries. |
| Need useful workspace writes without full filesystem access | Use ACLs plus write-restricted tokens and a sandbox write SID. |
| Need to protect sensitive project internals | Deny sandbox writes to paths such as `.git`, `.codex`, and `.agents`. |
| Need strong no-network mode | Use Windows Firewall rules. |
| Firewall could not target a synthetic restricted token SID | Run commands as dedicated sandbox users and target the offline user. |
| Dedicated sandbox users could not read the real user's files by default | Grant read ACLs to important directories during setup. |
| Setup required admin rights | Move privileged work into `codex-windows-sandbox-setup.exe`. |
| Launching a restricted child process across user boundaries was difficult | Add `codex-command-runner.exe`, which runs as the sandbox user and launches the final restricted command. |

## Files

- [`codex_windows_sandbox_summary.md`](codex_windows_sandbox_summary.md): detailed beginner-friendly explanation
- [`codex_windows_sandbox_architecture.png`](codex_windows_sandbox_architecture.png): generated architecture infographic

## Attribution

This tutorial is based on OpenAI's article by David Wiesen: [Building a safe, effective sandbox to enable Codex on Windows](https://openai.com/index/building-codex-windows-sandbox/).
