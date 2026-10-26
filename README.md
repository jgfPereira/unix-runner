# Unix Cross-Platform Interactive Runners

A robust GitHub Actions toolkit designed to spin up secure, fully-configured interactive SSH sessions (powered by **Upterm**) across multiple Unix-based operating systems and hardware architectures.

This repository enables cross-platform build pipelines, remote debugging, packaging (AppImage, macOS bundles), and interactive execution environments on **macOS** and **Linux** runners across both **x64** and **ARM64** architectures — all while leveraging GitHub Actions free runner minutes efficiently.

## 🚀 Key Features

* **Multi-Architecture & OS Support:** Dedicated workflows for:
  * **macOS ARM64** (`macos-26`, Apple Silicon host)
  * **macOS x64** (`macos-26-intel`, Intel host)
  * **Linux ARM64** (`ubuntu-24.04-arm` host running an `ubuntu:20.04` Docker container)
  * **Linux x64** (`ubuntu-24.04` host running an `ubuntu:20.04` Docker container)

* **Performance-Optimized Package Managers:**
  * **macOS Homebrew Optimization:** Disables slow Homebrew auto-updates and cleanup tasks (`HOMEBREW_NO_AUTO_UPDATE=1`, `HOMEBREW_NO_INSTALL_CLEANUP=1`), reducing macOS dependency setup from 10+ minutes to under 60 seconds.

* **Packaging & Desktop Tooling:**
  * **Linux AppImage Support:** Installs `appimagetool` (continuous release binaries for both `x86_64` and `aarch64`) with container-compatible extraction mode (`APPIMAGE_EXTRACT_AND_RUN=1`) and desktop integration utilities (`desktop-file-utils`, `file`).

* **Interactive SSH Sessions:** Direct terminal access via Upterm, restricted securely to authorized GitHub users (`limit-access-to-users: 'jgfPereira'`).

* **Container Terminal Ergonomics:** Full support for terminal colors, aliases (`ll`, `la`, colored `grep`), auto-completion (`bash-completion`), and interactive Bash shell startup inside Docker containers.

* **Dynamic Software Toolchains:** Automatically provisions modern developer stacks dynamically at runtime:
  * **Oracle JDK 25** (downloaded, extracted, and registered via `update-alternatives` on Linux or system paths on macOS)
  * **Latest Go (Golang)** (dynamically fetched via `go.dev/VERSION?m=text` for Linux and Homebrew for macOS)
  * **Maven**, **Python 3**, **ImageMagick** (`convert`/`magick`), **Binutils** (`objcopy` for `jpackage`), and **OpenSSL**

* **Secure Encrypted Assets:** Decrypts Git LFS tracked payload archives (`payload.zip.enc`) using PBKDF2 key derivation and AES-256-CBC encryption backed by GitHub Secrets.

## 📁 Workflow Files & Environments

| Workflow File | Target OS | Arch | Runner Host | Environment | Default Shell |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `linux-arm64.yml` | Linux | ARM64 (`aarch64`) | `ubuntu-24.04-arm` | `ubuntu:20.04` Container | `/bin/bash` |
| `linux-x64.yml` | Linux | x64 (`x86_64`) | `ubuntu-24.04` | `ubuntu:20.04` Container | `/bin/bash` |
| `macos-arm64.yml` | macOS | ARM64 (`aarch64`) | `macos-26` | Host OS | `/bin/zsh` / `bash` |
| `macos-x64.yml` | macOS | x64 (`x86_64`) | `macos-26-intel` | Host OS | `/bin/zsh` / `bash` |

## 🔄 Workflow Execution Flow

### Linux Workflows (`linux-arm64.yml` & `linux-x64.yml`)
1. **Container Initialization:** Boots an `ubuntu:20.04` Docker container with `SHELL: /bin/bash` exported for Upterm.
2. **Pre-Checkout Bootstrap:** Installs `git`, `git-lfs`, `unzip`, and `openssl` via `apt-get` inside the container.
3. **Checkout Code (`actions/checkout@v4`):** Clones the repository and downloads Git LFS assets.
4. **Decrypt & Extract Payload:** Decrypts `payload.zip.enc` with `ZIP_PASSWORD` secret and extracts files into `./extracted_files`.
5. **Install Dependencies & Packaging Tools:**
   * Installs `maven`, `python3`, `zip`, `imagemagick`, `curl`, `ca-certificates`, `binutils`, `bash-completion`, `file`, and `desktop-file-utils`.
   * Downloads architecture-specific continuous binary for `appimagetool` (`aarch64` or `x86_64`) to `/usr/local/bin/appimagetool`.
   * Configures `APPIMAGE_EXTRACT_AND_RUN=1` in `$GITHUB_ENV` to enable AppImage execution inside unprivileged Docker containers.
6. **Dynamic Go Installation:** Resolves the latest stable Go version from `go.dev`, downloads arch-specific tarball (`amd64` or `arm64`), installs to `/usr/local/go`, and updates `$GITHUB_PATH`.
7. **Oracle Java 25 Setup:** Downloads JDK 25 tarball for Linux (`x64` or `aarch64`), configures `update-alternatives` (`java`, `javac`), and exports `JAVA_HOME`.
8. **Terminal Ergonomics Configuration:** Writes a custom colorized `.bashrc` prompt (`PS1`), enables bash-completion, and links `.profile` to ensure login SSH sessions load interactive configurations.
9. **SSH Keepalive & Upterm Start:** Configures `ServerAliveInterval 30` in `~/.ssh/config` and launches Upterm SSH.

### macOS Workflows (`macos-arm64.yml` & `macos-x64.yml`)
1. **Checkout Code:** Clones repository with Git LFS assets (`actions/checkout@v4`).
2. **Decrypt & Extract Payload:** Decrypts `payload.zip.enc` and extracts into `./extracted_files`.
3. **Fast Homebrew Dependency Setup:**
   * Skips index updates and cleanup (`HOMEBREW_NO_AUTO_UPDATE=1`, `HOMEBREW_NO_INSTALL_CLEANUP=1`).
   * Installs `maven`, `go`, `python3`, `imagemagick`, and `openssl`.
4. **Oracle Java 25 Setup:** Downloads macOS JDK 25 tarball (`x64` or `aarch64`), extracts to `/Library/Java/JavaVirtualMachines/`, and exports `JAVA_HOME`.
5. **SSH Keepalive & Upterm Start:** Configures SSH client options and launches the Upterm session.

## 🧰 Installed Toolchain Summary

| Tool / Dependency | Version / Source | Supported Platforms |
| :--- | :--- | :--- |
| **Java JDK** | Oracle JDK 25 (Official Builds) | All Workflows |
| **Go (Golang)** | Latest Stable (Dynamically fetched from `go.dev`) | All Workflows |
| **AppImage Tool** | Continuous Release (`appimagetool`) | Linux (x64 & ARM64) |
| **Build Tools** | Maven, Binutils (`objcopy`), `file`, `desktop-file-utils` | All Workflows |
| **Scripting / Utility** | Python 3, ImageMagick, Zip / Unzip, OpenSSL | All Workflows |
| **SSH Interactive** | Upterm Action (`owenthereal/action-upterm@v2`) | All Workflows |

## 🛠️ Useful Commands & Cheat Sheet

### 1. Encrypting Payloads Locally (Before Pushing)

Since this repository is public, sensitive artifacts or private dependencies must be encrypted before being pushed to Git LFS:

```bash
openssl enc -aes-256-cbc -salt -pbkdf2 -in payload.zip -out payload.zip.enc
```

*(Ensure the decryption key matches the `ZIP_PASSWORD` GitHub secret).*

---

### 2. File Transfer via SCP (Active SSH Session)

When connecting via SSH/SCP, the default login working directory is `/root`. Because GitHub Actions extracts files inside the container workspace (`/__w/unix-runner/unix-runner/`), you must use the full absolute path when copying files locally:

#### Fetching Executables to Local Machine (Windows / macOS / Linux):

```bash
# Container Absolute Path (Required)
scp -o ServerAliveInterval=30 -l 8192 <user>@uptermd.upterm.dev:/__w/unix-runner/unix-runner/extracted_files/isign/target/launcher_template-linux .
```

* **`-o ServerAliveInterval=30`**: Prevents SSH idle timeouts.
* **`-l 8192`**: Caps transfer bandwidth at 8 Mbps (~1 MB/s) to maintain SSH session stability over public relays.

*Note: If working directly inside the interactive terminal session, navigate to the workspace directory first by running `cd /__w/unix-runner/unix-runner`.*