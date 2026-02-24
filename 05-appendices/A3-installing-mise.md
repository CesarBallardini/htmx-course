# Appendix C -- Installing mise

**Scope:** Step-by-step instructions for installing
[mise](https://mise.jdx.dev), the tool version manager used in this course to
install Gleam and Erlang/OTP.

---

## 1. What Is mise?

mise (pronounced "meez", from the French *mise en place*) is a polyglot tool
version manager. It installs and manages versions of programming languages,
runtimes, and CLI tools -- including Gleam, Erlang, Node.js, Python, and
hundreds of others.

For this course, mise handles two things:

1. **Installing Gleam and Erlang/OTP** at compatible versions.
2. **Pinning those versions** via a `mise.toml` file in the project root, so
   every developer working on the project uses the same toolchain.

When you run `mise install` inside a directory that contains a `mise.toml`,
mise reads the file and installs exactly the versions listed.

---

## 2. Installing mise

### GNU/Linux

The recommended method is the install script:

```bash
curl https://mise.run | sh
```

This downloads the latest mise binary and places it in `~/.local/bin/`. After
the script finishes, add mise to your shell by running the activation command
for your shell:

```bash
# Bash
echo 'eval "$(~/.local/bin/mise activate bash)"' >> ~/.bashrc
source ~/.bashrc

# Zsh
echo 'eval "$(~/.local/bin/mise activate zsh)"' >> ~/.zshrc
source ~/.zshrc

# Fish
echo '~/.local/bin/mise activate fish | source' >> ~/.config/fish/config.fish
source ~/.config/fish/config.fish
```

**Alternative: system package managers.**

On Ubuntu (26.04+):

```bash
sudo add-apt-repository -y ppa:jdxcode/mise
sudo apt update -y
sudo apt install -y mise
```

On Fedora (41+):

```bash
dnf copr enable jdxcode/mise
dnf install mise
```

On Arch Linux:

```bash
sudo pacman -S mise
```

On Alpine Linux:

```bash
apk add mise
```

### macOS

With Homebrew:

```bash
brew install mise
```

Or the install script (same as Linux):

```bash
curl https://mise.run | sh
```

After installation, activate mise in your shell:

```bash
# Bash
echo 'eval "$(mise activate bash)"' >> ~/.bashrc

# Zsh
echo 'eval "$(mise activate zsh)"' >> ~/.zshrc

# Fish
echo 'mise activate fish | source' >> ~/.config/fish/config.fish
```

### Windows

With Chocolatey:

```powershell
choco install mise
```

With Scoop:

```powershell
scoop install mise
```

With winget:

```powershell
winget install jdx.mise
```

After installation, activate mise by adding this to your shell profile. For
PowerShell, add to your `$PROFILE`:

```powershell
# PowerShell
mise activate pwsh | Out-String | Invoke-Expression
```

For Git Bash / MSYS2, use the Bash activation from the Linux section above.

---

## 3. Verifying the Installation

After installing and activating mise, verify it works:

```bash
mise --version
# mise 2025.x.x
```

---

## 4. Installing the Course Tools

Navigate to the project directory (where `mise.toml` lives) and run:

```bash
mise install
```

mise reads the `mise.toml` file:

```toml
[tools]
erlang = "27"
gleam = "1"
```

This installs:

- **Erlang/OTP 27** (the latest patch of the 27.x line). On first install this
  may take several minutes because mise compiles Erlang from source using
  [kerl](https://github.com/kerl/kerl). Subsequent installs are instant.
- **Gleam 1** (the latest 1.x release). This downloads a prebuilt binary and
  takes only a few seconds.

Verify both tools are available:

```bash
gleam --version
# gleam 1.14.0 (or later)

erl -eval 'erlang:display(erlang:system_info(otp_release)), halt().' -noshell
# "27"
```

> **Note on rebar3.** Gleam's build tool downloads rebar3 (the Erlang build
> tool it uses under the hood) automatically on first run. You do not need to
> install it separately.

---

## 5. How mise Selects Versions

When you run `gleam` or `erl`, mise intercepts the command and uses the version
specified in the nearest `mise.toml` file, searching from the current directory
upward. This means:

- Inside the Teamwork project directory, `gleam --version` returns the version
  pinned in `mise.toml`.
- Outside the project, mise falls back to a global default (if you set one with
  `mise use -g gleam@1`) or the system-installed version.

This per-directory version switching is automatic. You do not need to run any
`activate` or `switch` commands when you move between projects.

---

## 6. Useful mise Commands

| Command | What it does |
|---------|--------------|
| `mise install` | Install all tools listed in `mise.toml` |
| `mise use gleam@1` | Pin Gleam 1.x in the current directory's `mise.toml` |
| `mise use -g erlang@27` | Set a global default Erlang version |
| `mise ls` | Show installed tool versions and which is active |
| `mise ls-remote gleam` | List all available Gleam versions |
| `mise self-update` | Update mise itself to the latest version |
| `mise doctor` | Diagnose common configuration problems |

---

## 7. Official Documentation

| Resource | Link |
|----------|------|
| mise homepage | [https://mise.jdx.dev](https://mise.jdx.dev) |
| Installation guide | [https://mise.jdx.dev/installing-mise.html](https://mise.jdx.dev/installing-mise.html) |
| Getting started | [https://mise.jdx.dev/getting-started.html](https://mise.jdx.dev/getting-started.html) |
| Configuration reference | [https://mise.jdx.dev/configuration.html](https://mise.jdx.dev/configuration.html) |
| Tool registry | [https://mise.jdx.dev/registry.html](https://mise.jdx.dev/registry.html) |
| GitHub repository | [https://github.com/jdx/mise](https://github.com/jdx/mise) |
| Erlang backend docs | [https://mise.jdx.dev/lang/erlang.html](https://mise.jdx.dev/lang/erlang.html) |

---

## 8. Troubleshooting

**`mise: command not found` after installation.**
You need to activate mise in your shell. See the activation commands in
Section 2 above for your shell, or run `mise doctor` to diagnose the issue.

**Erlang compilation fails.**
mise compiles Erlang from source, which requires build dependencies. On
Debian/Ubuntu:

```bash
sudo apt install build-essential autoconf m4 libncurses-dev libssl-dev \
  libwxgtk3.2-dev libgl1-mesa-dev libglu1-mesa-dev libpng-dev libssh-dev \
  unixodbc-dev xsltproc fop
```

On macOS with Homebrew:

```bash
brew install autoconf openssl wxwidgets
```

See the [kerl documentation](https://github.com/kerl/kerl#readme) for the full
list of optional dependencies.

**Gleam install fails with a checksum error.**
Run `mise cache clear` and try again. If the error persists, update mise with
`mise self-update`.

**I do not want to use mise.**
That is fine. Install Gleam and Erlang/OTP manually using the instructions on
their official websites:

- Gleam: [https://gleam.run/getting-started/installing/](https://gleam.run/getting-started/installing/)
- Erlang/OTP: [https://www.erlang.org/downloads](https://www.erlang.org/downloads)

Any Gleam 1.x and OTP 26+ combination will work with this course.
