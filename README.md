# un-🦭

`un-seal` is a confined Snap utility designed to automate the initialization, unsealing, and authorization of Vault applications deployed via Juju.

## Overview

Deploying a charmed Vault with Juju requires a specific sequence of operations to bootstrap the cluster: initializing the Vault operator, securely managing the generated unseal keys and root token, unsealing individual units, and authorizing the charm to interact with the Vault API.

`un-seal` streamlines this workflow into a single interactive command. It is designed for security-conscious environments, supporting split-file credential storage and GPG encryption (compatible with hardware tokens such as YubiKey) to facilitate separation of duties.

The tool automatically detects the state of the cluster and performs the necessary actions:

1.  **Cluster Initialization:** Generates key shares and the root token, encrypts them individually to a specified directory, and unseals the cluster.
2.  **Cluster Unsealing:** Detects sealed units (e.g., following a restart), decrypts the necessary keys from storage, and brings the cluster online.

## Features

* **Automated Leader Detection:** Queries the Juju model to identify and target the application leader for initialization.
* **Multi-File Credential Storage:** Stores the root token and each unseal key in separate GPG-encrypted files (e.g., `vault_token.gpg`, `vault_key1.gpg`). This architecture supports the separation of duties by allowing keys to be distributed among different operators.
* **Hardware-Backed Security:** fully integrates with GPG, enabling the use of smart cards and YubiKeys for encryption and decryption operations.
* **Resilient Credential Loading:** Implements a three-tier logic to retrieve keys:
    1.  **Auto-Load:** Scans the credentials directory for encrypted files.
    2.  **File Prompt:** Prompts for specific file paths if automatic detection is incomplete.
    3.  **Manual Entry:** Accepts raw input for keys and tokens as a final fallback mechanism.
* **Smart Threshold Detection:** Queries the Vault status to determine the exact number of unseal keys required before prompting the user.
* **Charm Authorization:** Automates the Juju secret lifecycle (`add-secret` -> `grant-secret` -> `authorize-charm`) to provision the charm with the root token securely.
* **Secure Cleanup:** Utilizes `shred` to overwrite and remove sensitive temporary files (such as the CA certificate) immediately after use.

## Dependencies

### Host Requirements
This snap is built with `classic` confinement to interact with your system's tools. Ensure these are installed:

* **`juju`:** Required to interface with your controller/model.
    ```bash
    sudo snap install juju
    ```

### Bundled Dependencies

The snap comes pre-packaged with the following binaries; no manual installation is required:
* `vault` (CLI client)
* `jq` (JSON processor)
* `gpg` (GnuPG)

## Installation

### Snap Store
Install the snap with classic confinement:

```bash
sudo snap install un-seal --classic
```

### Build from Source
To build and install locally:

1.  Clone the repository:
    ```bash
    git clone https://github.com/Ankow99/un-seal.git
    cd un-seal
    ```
2.  Pack the snap:
    ```bash
    snapcraft pack
    ```
3.  Install the generated snap:
    ```bash
    sudo snap install ./un-seal_3.0.0_amd64.snap --classic --dangerous
    ```

## Usage

Execute the command from a terminal. The tool will attempt to discover a `vault` application in the currently active Juju model.

```bash
un-seal [options]
```

### Options

| Flag | Long Flag | Description | Default |
| :--- | :--- | :--- | :--- |
| `-a` | `--charm-name` | The name of the Juju Vault application. | `vault` |
| `-s` | `--key-shares` | Number of key shares to generate (Init only). | `3` |
| `-t` | `--key-threshold` | Number of keys required to unseal (Init only). | `2` |
| `-T` | `--timeout` | Minutes to wait for units to initialize. | `10` |
| `-c` | `--creds-dir` | Directory to save/load GPG-encrypted files. | `$HOME/[charm-name]_creds/` |
| `-g` | `--gpg-id` | GPG Key ID/Email for encryption (Init only). | *(Prompts if empty during init)* |
|      | `--skip-auth` | Force skip the charm authorization steps (5-8). | `false` |
| `-S` | `--seal` | (Easter Egg) Invoke the seal after success. | `false` |
| `-h` | `--help` | Show help message. | |

## Examples

**Initialize a High-Availability Cluster**
Initialize an application named `vault-ha` with 5 key shares and a threshold of 3, encrypting credentials for a specific GPG recipient:

```bash
un-seal --charm-name vault-ha -s 5 -t 3 --gpg-id "admin@example.com"
```

**Unseal using a custom credential path**
Unseal a cluster using keys stored on external media:

```bash
un-seal --creds-dir /media/secure-usb/vault_keys/
```

## Technical Workflow

1.  **Environment Validation:** Checks for the presence of required dependencies (`juju`, `gpg`, `vault`) and confirms the target application exists in the current Juju model.
2.  **Leader Identification:** Identifies the Juju leader unit, which is required for the initialization step.
3.  **CA Retrieval:** Retrieves the `self-signed-vault-ca-certificate` secret from Juju to establish a secure TLS connection with the Vault units.
4.  **State Management:**
    * **Initialization:** Executes `operator init`. The output is parsed, and the root token and unseal keys are encrypted into individual files within the specified credentials directory.
    * **Unsealing:** Queries the seal status. If sealed, the tool attempts to decrypt sufficient keys from the directory. If keys are missing, it requests file paths or raw input.
5.  **Unseal Operations:** Unseals the **Leader** unit first to ensure cluster consensus, then iterates through all follower units.
6.  **Charm Authorization:** Passes the Root Token to the Juju charm via a short-lived Juju secret, executing the `authorize-charm` action to complete the charm configuration.
7.  **Cleanup:** Securely shreds temporary files created during execution. The encrypted credential files are preserved for backup purposes.

## License

This project is licensed under the GPL-3.0-only license. See the `LICENSE` file for details.
