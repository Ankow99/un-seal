# un-🦭

A simple, interactive, and secure snap for initializing and unsealing Juju-deployed Vault clusters.

## Overview

`un-seal` is a command-line tool that solves the tedious initialization and unsealing problem for [Vault](https://charmhub.io/vault) clusters deployed with [Juju](https://juju.is/). It automates the entire process of initialization, key management, and unsealing, handling the two primary scenarios you'll face:

1.  **Brand-New Cluster:** If you've just deployed Vault, `un-seal` will initialize the cluster, **encrypt** the new keys and root token using GPG (supporting YubiKey), save them to a secure file, and then unseal all units.

2.  **Existing/Restarted Cluster:** If your cluster is already initialized but sealed (e.g., after a restart), `un-seal` will detect this, decrypt your credentials file (prompting for YubiKey touch/PIN), and automatically unseal the cluster.

## Features

* **Auto-Detects Leader:** Automatically finds the Juju leader unit to target for initialization.

* **Hardware-Backed Security:** Full support for **YubiKey** (via GPG) to encrypt and decrypt Vault credentials.

* **Secure Key Storage:** On first-time init, saves the *new* unseal keys and root token to a GPG-encrypted file (`$HOME/[charm-name]_creds.gpg`) with `600` permissions.

* **Auto-Unseal:** No need to manually type keys. The tool loads them securely from your encrypted file to unseal the cluster.

* **Fail-Secure:** Implements `shred` to securely overwrite and wipe sensitive temporary files and secrets from disk/memory immediately after use.

* **Unseals All Units:** Intelligently unseals the leader *first*, then polls and unseals all follower units as they come online.

* **Automatic Charm Authorization:** Handles the full `juju add-secret`, `grant-secret`, and `authorize-charm` flow to provide the charm with the root token.

* **Self-Contained:** The snap bundles all required dependencies (`jq`, `vault`, `gpg`).

## Dependencies

### Host Requirements
This snap is built with `classic` confinement and expects the following to be installed on your host machine:
* **`juju`:** The script must find and use the `juju` command from your host system to access your Juju credentials.
    ```bash
    sudo snap install juju
    ```
* **`snapcraft` (to build from source):**
    ```bash
    sudo snap install snapcraft --classic
    ```

### Bundled Dependencies
The `un-seal` snap bundles the following tools, so you don't need to install them:
* `vault`
* `jq`
* `gpg` (and related smart card tools)

## Installation

### Snap Store - Preferred:
1.  Install the Snap:
    This snap must be installed with `--classic` confinement to access your host `juju` command.
    ```bash
    sudo snap install un-seal --classic
    ```

### From Source:
1.  Clone the Repository:
    ```bash
    git clone https://github.com/Ankow99/un-seal.git
    cd un-seal
    ```
2.  Build the Snap:
    ```bash
    snapcraft pack
    ```
3.  Install the Snap:
    ```bash
    # The version/arch may differ
    sudo snap install ./un-seal_2.1.0_amd64.snap --classic
    ```
    
## Usage
Run the command from your terminal. It will automatically find the `vault` charm in your current Juju model.
```bash
un-seal
```

### Options
* `-a, --charm-name <name>`: The name of the Juju Vault charm (default: "vault"). Must exist in Juju.
* `-s, --key-shares <num>`: The number of key shares *to generate* on first-time init (default: 3). Must be >= 1.
* `-t, --key-threshold <num>`: The number of keys *required* to unseal (default: 2). Must be >= 1 and <= key-shares.
* `-T, --timeout <mins>`: The timeout in minutes to wait for follower units to initialize (default: 10). Must be >= 1.
* `-c, --creds-file <path>`: Path to the GPG-encrypted credentials file (default: `$HOME/[charm-name]_creds.gpg`).
* `-g, --gpg-id <id>`: The GPG Key ID or email to use for encryption. Required during initialization (will prompt if omitted).
* `-h, --help`: Show the help message.

### Examples

**Initialize** a cluster named `vault-ha`, encrypting for a specific GPG email:
\```bash
un-seal --charm-name vault-ha --gpg-id "security@example.com"
\```

**Unseal** a cluster using an existing encrypted file (prompts for YubiKey):
\```bash
un-seal --charm-name vault-ha
\```

## How It Works

The script follows this logical flow:

1.  **Dependency Check:** Verifies that `juju`, `jq`, `vault`, and `gpg` are all functional.
2.  **Find Leader:** Parses `juju status --format=json` to find the leader unit of the specified application.
3.  **Fetch CA:** Grabs the `self-signed-vault-ca-certificate` from Juju secrets.
4.  **Initialize or Decrypt:**
    * **New Vault:** Runs `vault operator init`. It **encrypts** the output to `.gpg` using your provided GPG ID and saves it to disk.
    * **Existing Vault:** Detects initialization, prompts for YubiKey/GPG validation, decrypts the file in memory, and verifies the keys match the Vault threshold.
5.  **Unseal Leader:** Unseals the leader unit *first* using the decrypted keys.
6.  **Unseal Followers:** Loops through all other units, polling for initialization, and unseals them.
7.  **Authorize Charm:** Creates a Juju secret with the root token, grants it to the application, and runs `authorize-charm`.
8.  **Secure Cleanup:** Uses `shred` to wipe the temporary CA file and secrets. If initialization was successful, it deletes the credentials file.

## License
This project is licensed under the GPL-3.0-only license - see the `LICENSE` file for details.
