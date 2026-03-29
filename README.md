# safe-yolo

> **DISCLAIMER**: Follow the instructions below at your own risk! You are responsible for properly securing your agents. This is for informational purposes only: make your own judgment whether these protections are sufficient for your use case. Feel free to suggest any improvements.

How to somewhat safely run coding agents in YOLO mode, i.e.

* `claude --dangerously-skip-permissions`
* `copilot --allow-all`

> **Note**: The goal here is not to set up a fully hardened sandbox that can protect against any adversarial attack. The point is to make it possible to run a coding agent in YOLO mode and have some reasonable assurance that it's not going to erase your devbox or send something embarrassing to your boss.

## Setup WSL

On Windows, the basic principle is to use WSL with minimal interop to the host OS, so that an agent running inside WSL cannot use Windows credentials or resources (except for explicit, opt-in file system interop), and to protect any highly-privileged secrets that need to be inside WSL (e.g to allow git push to remote) to only be accessible via sudo (which the agent can't do without user intervention). That way, the agent can do pretty much whatever it wants inside its WSL sandbox without harming the rest of the system.

Obviously, these steps imply that the agent will be running in a Linux environment, which it will tend to prefer anyway, but it means that Windows-specific development is out of scope for this strategy.

> **Note:** These steps isolate the agent from Windows host resources but do not restrict outbound network access, by design; unlike our production agents, we want our dev agents to have full access to the internet. An agent running inside WSL can still make arbitrary HTTP requests, access internal network services, and reach external APIs. We protect against this with a combination of the "hard" protection of only giving the agent least-privileged access tokens, and the "soft" protection of agent instructions to ask the user before doing anything with external side effects even when in YOLO mode.

### Disable Interop

Edit `/etc/wsl.conf` inside the WSL distro:

```ini
[boot]
systemd = true # Required for SSH server auto-start (default for recent WSL versions)

[interop]
enabled = false # Prevent WSL from launching Windows executables
appendWindowsPath = false # Do not append Windows PATH (no access to cmd.exe, powershell.exe, etc.)

[automount]
enabled = false # Do not auto-mount Windows drives (C:\, D:\, etc.)
mountFsTab = true # Use /etc/fstab for any explicit mounts
```

After editing, restart the distro from PowerShell:

```powershell
wsl --shutdown; wsl
```

### Opt-in to Specific Windows Folder Mounts

If you need access to specific Windows folders (e.g., a shared documents folder), add entries to `/etc/fstab` inside WSL. For example:

Read-only mount:

```
C:/folderA /mnt/folderA drvfs ro,noatime,uid=1000,gid=1000 0 0
```

Read-write mount (use sparingly):

```
C:/folderB /mnt/folderB drvfs rw,noatime,uid=1000,gid=1000 0 0
```

After editing `/etc/fstab`, either restart WSL (`wsl --shutdown; wsl` from PowerShell) or mount manually:

```bash
sudo mkdir -p /mnt/folderA
sudo mount -a
```

> **Note**: File access inside Windows folder mounts is about 20x slower than the native WSL filesystem (even native Windows is slower than native WSL).

## Git Setup inside WSL

### Credential Isolation

With interop disabled, WSL cannot invoke Windows Credential Manager, and Windows-side credentials (browser sessions, SSH keys, etc.) are not accessible from WSL. Git credentials need to be available in WSL, though, if we want to do anything with repos. We'll follow this strategy:

- Clone repos with a **read-only** PAT embedded in the remote URL to limit blast radius if an agent goes rogue
- Use a separate push URL with a **read/write** PAT only when needed

A helper script, `git_clone` (in this repo), handles cloning and optionally setting up write access. It works for both ADO and GitHub repos.

### Bootstrap

Create `~/dev` and copy `git_clone` into it from Windows:

```powershell
wsl mkdir -p ~/dev
Get-Content -Raw C:\path\to\git_clone | wsl bash -c 'cat > ~/dev/git_clone'
```

Or paste the contents of `git_clone` directly into `~/dev/git_clone` inside WSL. Then make it executable:

```bash
chmod +x ~/dev/git_clone
```

### Cloning Repos

Pass the plain clone URL as shown in the ADO or GitHub clone dialog:

```bash
# ADO repo
~/dev/git_clone "https://<org>.visualstudio.com/<project>/_git/<repo>" --user <user@domain.com> --ro <RO_PAT> --rw <RW_PAT>

# GitHub repo
~/dev/git_clone "https://github.com/<org>/<repo>.git" --ro <RO_PAT> --rw <RW_PAT>

# No PAT (plain clone, e.g. public repo)
~/dev/git_clone "https://github.com/<org>/<repo>.git"
```

`--user` is the ADO username email (user@domain.com). It is required for ADO repos when a PAT is provided. Omit `--rw` to skip creating the `gitx_<repo>` write script.

The script clones into `~/dev/<reponame>` and is idempotent: if the repo already exists, it pulls instead.

### Pushing Changes

When `--rw` is provided, `git_clone` creates `~/dev/gitx_<reponame>`: a root-owned script that agents cannot read or execute without `sudo` (which requires a password). When the PAT expires, update it:

```bash
sudo nano ~/dev/gitx_<reponame>
```

Run the script as follows from within the relevant repo to push changes:

```bash
sudo ~/dev/gitx_myrepo push                        # push current branch
sudo ~/dev/gitx_myrepo push HEAD:main              # push to a specific branch
sudo ~/dev/gitx_myrepo push-delete feature-branch  # delete a remote branch
sudo ~/dev/gitx_myrepo push-tags                   # push all tags
```

## Snapshots

Before letting an agent run unsupervised, you can export the distro so you can restore it if things go wrong:

```powershell
wsl --export <distro> backup.tar
```

To restore:

```powershell
wsl --unregister <distro>
wsl --import <distro> <install-path> backup.tar
```

## VS Code via SSH

> **Why not the WSL extension?** The VS Code [Remote - WSL](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-wsl) extension requires WSL interop for Windows path translation. With interop enabled, an agent could create and execute a Windows `.exe` inside WSL, which would run in the Windows user context with full access to Windows credentials. Using Remote - SSH avoids this and allows interop to stay disabled.

### One-time setup (inside WSL)

Install the SSH server and enable it to start automatically:

```bash
sudo apt install openssh-server
sudo systemctl enable ssh
```

> **Note:** This requires `systemd=true` under `[boot]` in `/etc/wsl.conf` (the default for recent WSL versions).

### One-time setup (on Windows)

Generate an SSH key and copy it into WSL for passwordless login:

```powershell
ssh-keygen -t ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub | wsl sh -c "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Add a host entry to `C:\Users\<user>\.ssh\config`:

```
Host WSL
    HostName 127.0.0.1
    User <wsl-username>
```

Install the [Remote - SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh) extension in VS Code.

### Connecting VS Code

From PowerShell:

```powershell
code --remote ssh-remote+WSL /home/<user>/dev/<project>
```

Or from within VS Code, open the Command Palette and select **Remote-SSH: Connect to Host** → **WSL**.

VS Code will automatically install its server component inside WSL on first connect. Extensions that need to run against the project (linters, language servers, etc.) should be installed in the SSH context — VS Code will prompt for this.

## Agent Instructions

The agent instructions should contain a rule that agents must confirm with the user any step that might have side effects outside of the WSL sandbox. This is for defense-in-depth only: the primary protection is the fact that the WSL environment has no access to the Windows host or any credentials with which it could produce side effects in the first place. But you never know, maybe you messed up and gave it an overly powerful PAT.

Here is a suggested rule to put in your global `~/.claude/CLAUDE.md` and `~/.copilot/copilot-instructions.md` files (tip: use @-import syntax and symlink, respectively, to point these to the same shared file, or one to the other):

```
## YOLO Mode Rules

Even when running in YOLO mode (no user permission needed to make tool calls, modify files, execute bash commands, etc.), you MUST ask the user for confirmation of any action you want to take that modifies resources OUTSIDE of the local computer (e.g., an MCP tool call that updates an ADO work item, git push to remote repo, Kusto command execution, etc.). You also MUST ask the user for permission to install things globally (such as CLIs, or anything via apt/winget/etc.).
```

## Warnings and Gotchas!

Windows clipboard can introduce invisible Unicode characters — in particular the UTF-8 BOM (`U+FEFF`, bytes `ef bb bf`) — when copying text between Windows and WSL. Linux will silently accept these as part of filenames, which can cause baffling "No such file or directory" errors when a filename looks correct but isn't.

To scan for affected files:

```
find ~ -name $'\xef\xbb\xbf*'
```

You can also use `xxd <filename> | head` to inspect raw bytes if a file or command behaves unexpectedly but looks correct.

To fix an affected file:

```
mv $'\xef\xbb\xbf'<filename> <filename>
```
