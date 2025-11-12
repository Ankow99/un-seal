# un-🦭

A simple, interactive snap for initializing and unsealing Juju-deployed Vault clusters.

## Overview

`un-seal` is a command-line tool that solves the "Day 0" problem for [Vault](https://www.vaultproject.io/) clusters deployed with [Juju](https://juju.is/). It automates the entire process of initialization, key management, and unsealing, handling the two primary scenarios you'll face:

1.  **Brand-New Cluster:** If you've just deployed Vault, `un-seal` will initialize the cluster, securely save the new keys and root token to your home directory, and then unseal all units.

2.  **Existing/Restarted Cluster:** If your cluster is already initialized but sealed (e.g., after a restart), `un-seal` will detect this and interactively prompt you for the unseal keys and root token to bring the cluster online.

## Features

* **Auto-Detects Leader:** Automatically finds the Juju leader unit to target for initialization.

* **Interactive:** Prompts for keys if Vault is already initialized.

* **Secure Key Storage:** On first-time init, saves the *new* unseal keys and root token to `$HOME/[app-name]_creds.txt` with `600` permissions.

* **Unseals All Units:** Intelligently unseals the leader *first*, then polls and unseals all follower units as they come online.

* **Automatic Charm Authorization:** Handles the full `juju add-secret`, `grant-secret`, and `authorize-charm` flow to provide the charm with the root token.

* **Self-Contained:** The snap bundles all required dependencies (`jq`, `vault`) and cleans up after itself.

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
    This snap must be installed with `--classic` confinement to access your host `juju` command.
    ```bash
    # The version/arch may differ
    sudo snap install ./vault-unseal_8.1_amd64.snap --classic
    ```
    
## Usage
Run the command from your terminal. It will automatically find the `vault` application in your current Juju model.
```bash
un-seal
```

### Options
* `-a, --app-name <name>`: The name of the Juju Vault application (default: "vault").
* `-s, --key-shares <num>`: The number of key shares *to generate* on first-time init (default: 5).
* `-t, --key-threshold <num>`: The number of keys *required* to unseal (default: 3).
* `-T, --timeout <mins>`: The timeout in minutes to wait for follower units to initialize (default: 10).
* `-h, --help`: Show the help message.

### Example
Initialize a cluster named `vault-ha` using a 3-of-5 key-share setup:
```bash
un-seal --app-name vault-ha --key-shares 5 --key-threshold 3
```

## How It Works

The script follows this logical flow:

1.  **Dependency Check:** Verifies that `juju`, `jq`, and `vault` are all installed and functional.
2.  **Find Leader:** Parses `juju status --format=json` to find the leader unit of the specified application.
3.  **Fetch CA:** Grabs the `self-signed-vault-ca-certificate` from Juju secrets and saves it to `$HOME/[app-name]_ca.pem` for the `vault` client to use.
4.  **Initialize or Prompt:**
    * **New Vault:** It runs `vault operator init`. If successful, it saves the keys/token to `$HOME/[app-name]_creds.txt`.
    * **Existing Vault:** If it detects Vault is initialized, it prompts you for the unseal keys and root token.
5.  **Unseal Leader:** The script unseals the leader unit *first*. This is critical, as follower units will not initialize until the leader is unsealed.
6.  **Unseal Followers:** The script loops through all other units, polling each one until it reports `initialized: true`, and then submits the unseal keys.
7.  **Authorize Charm:** Finally, it creates a Juju secret with the root token, grants it to the application, runs the `authorize-charm` action, and cleans up the secret.

## License
This project is licensed under the GPL-3.0-only license - see the `LICENSE` file for details.
