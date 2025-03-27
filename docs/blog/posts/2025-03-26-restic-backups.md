---
date: 2025-03-26
title: Deploying Restic on OpenStack Flex with User Data
authors:
  - cloudnull
description: >
  Deploying Restic on OpenStack Flex with User Data
categories:
  - Server
  - Backups
---

# Deploying Restic on OpenStack Flex with User Data

[Restic](https://restic.net/) is an open-source backup tool that focuses on providing secure, efficient, and easy-to-use backups. This blog post will show you how to automatically install and configure Restic on a fresh OpenStack VM using user data (cloud-init).

Here’s what you’ll accomplish:

1. Install Restic to `/usr/local/bin`.
2. Configure a systemd service and timer that runs every 12 hours to back the `/home` directory.
3. Store backups in Swift using application credentials.

<!-- more -->

## Prerequisites

!!! note "This blog post was written with the following environment assumptions already existing"

    - Network: `tenant-net`
    - Key Pair: `tenant-key`

    If the project isn't setup completely, checkout the [getting started guide](https://blog.rackspacecloud.com/blog/2024/06/18/getting_started_with_rackspace_openstack_flex).

- **OpenStack project** where you can create and manage new VM instances.

- **Application credentials** to authenticate with Swift. You’ll typically have two pieces of information from your OpenStack environment: `OS_APPLICATION_CREDENTIAL_ID` and `OS_APPLICATION_CREDENTIAL_SECRET`.

- **A container** created in Swift to hold the Restic backups.

## High-Level Workflow Overview

Below is an example **Mermaid** sequence diagram that illustrates the high-level workflow of how Restic operates with Swift as the storage backend. You can embed this code (between triple backticks) in a Markdown file or a platform that supports Mermaid to visualize the diagram.

``` mermaid
sequenceDiagram
    participant U as User / Agent
    participant R as Restic CLI / Service
    participant S as Swift Storage
    participant Repo as Restic Repository

    U->>R: 1) Initiates backup or restore command
    note right of R: Restic reads config/env<br> (credentials, repository URL, etc.)

    alt Backup Workflow
        R->>R: Deduplicate & encrypt data chunks
        R->>S: Send encrypted data to Swift
        S->>Repo: Swift stores data in Restic repository
        Repo->>R: Confirm successful storage
    end

    alt Restore Workflow
        R->>S: Request encrypted data chunks
        S->>Repo: Retrieve data from Restic repository
        Repo->>R: Return encrypted data
        R->>R: Decrypt data chunks
        R->>U: Provide restored files to user
    end

    note over R,S: Communication with Swift uses<br> OpenStack application credentials.
```

### Explanation

**User / Agent**: This is either a person or an automated process that invokes Restic commands (backup, restore, etc.).

**Restic CLI / Service**: The Restic binary or service running on your machine. It manages your backups, handles data deduplication, and encrypts your data before sending it to storage.

**Swift Storage**: Your Swift endpoint in OpenStack, storing the actual backup objects. Restic uses application credentials to authenticate.

**Restic Repository**: The structure Restic maintains within Swift to hold snapshots, indexes, and data packs.

This diagram demonstrates how Restic takes in your backup or restore request, works with Swift to store or retrieve deduplicated data, and confirms a successful backup or returns the restored data back to the user.

## Steps Overview

1. **Write the cloud-init (user data) script** – The script:
   - Downloads/installs Restic.
   - Creates an environment file for Swift credentials.
   - Creates systemd service and timer units for automated backups.
   - Starts the timer.

2. **Launch a new VM** using the provided user data script.

3. **Verify** that Restic is installed and backups run automatically.

### Generate the Application Credentials

Before you can use Restic with Swift, you need to create application credentials in OpenStack. These credentials will allow Restic to authenticate with Swift and store backups in your container.

For the purpose of this example, use the openstack cli to generate the credentials with a minimum set of roles required for Restic to function.

``` shell
openstack --os-cloud default application credential create \
          --restricted \
          --role creator \
          --role reader \
          restic
```

!!! example "Application Credential Output"

    ``` shell
    +--------------+----------------------------------------------------------------------------------------+
    | Field        | Value                                                                                  |
    +--------------+----------------------------------------------------------------------------------------+
    | id           | 12312312312313123131231231231231                                                       |
    | name         | restic                                                                                 |
    | description  | None                                                                                   |
    | project_id   | 12312312312312312313123123123132                                                       |
    | roles        | creator reader                                                                         |
    | unrestricted | False                                                                                  |
    | access_rules | []                                                                                     |
    | expires_at   | None                                                                                   |
    | secret       | 12345678901234567890123456789012345678901234567890123456789012345678901234567890123456 |
    +--------------+----------------------------------------------------------------------------------------+
    ```

### The Cloud-init Script

Below is an example user data script you can provide to your instance in the “User Data” field when launching it in the dashboard (or via the CLI with `--user-data`). the script will install Restic, set up a systemd service and timer to run backups every 12 hours, and store the backups in Swift.

!!! example "`/tmp/restic-user-data.yaml`"

    ``` yaml
    --8<-- "docs/blog/posts/assets/files/2025-03-26/restic-cloud-init.yaml"
    ```

Download the cloud-config script [here](assets/files/2025-03-26/restic-cloud-init.yaml).

!!! note

    To use this user-data file, update it with your own Swift container name, environment variables, and desired backup paths.

    Remember to update placeholder credentials and passwords from the script.

    - `<YOUR_APP_CRED_ID>` and `<YOUR_APP_CRED_SECRET>` placeholders with the application credential ID and secret you generated earlier.

    - `<SUPER_SECURE_PASSWORD>` value with the secret you wish to use to encrypt your backup contents.

#### Explanation of Key Sections

- **`packages:`** Installs any packages needed for setup. We install `curl` to fetch the Restic binary.

- **Download and Install Restic** We retrieve the latest official Restic binary and place it in `/usr/local/bin/restic`, then make it executable.

- **Environment File:** `/etc/restic.env` This file contains your Swift credentials (`OS_APPLICATION_CREDENTIAL_ID`, `OS_APPLICATION_CREDENTIAL_SECRET`, etc.) along with the Restic repository URL and password. We protect this file with `chmod 600`.

- **Systemd Service:** `/etc/systemd/system/restic-backup.service` A simple one-shot service that runs `restic backup` on your specified directory. It sources the environment variables from `/etc/restic.env`. You can add a `forget --prune` step or `check` step as you see fit.

- **Systemd Timer:** `/etc/systemd/system/restic-backup.timer` Triggers `restic-backup.service` every 12 hours. Also sets a delay of 15 minutes after boot before the first run (`OnBootSec=15min`), ensuring the system is somewhat stable before performing the initial backup.

- **Initializing the Repository:** Before using Restic for the first time, you typically run `restic init`.

### Launch the VM with User Data

!!! note "OS Compatibility"

    At the time of this writing, the user-data script provided is compatible with both Debian and RHEL based distributions: tested Debian, Ubuntu, CentOS, and Rocky. If you are using a different distribution, you may need to adjust the package installation commands.

The OpenStack CLI, provide the above YAML in the **User Data** section when creating your new instance.

``` shell
openstack --os-cloud default server create app-0 \
          --flavor gp.0.1.2 \
          --network tenant-net \
          --image "Debian 12" \
          --user-data /tmp/restic-user-data.yaml \
          --key-name tenant-key
```

### Verification

After the VM boots, you can verify that your cloud-init script ran successfully and that the systemd timer is running:

#### Check Restic version

``` shell
restic version
```

!!! example "Restic Version Output"

    ``` shell
    restic 0.17.3 compiled with go1.23.3 on linux/amd64
    ```

#### Check systemd timer

``` shell
systemctl status restic-backup.timer
systemctl status restic-backup.service
```

Ensure the timer is active and set to trigger every 12 hours.

#### Check Swift container

Check the swift object storage environment for the configuration and initialization of the Restic repository. In the above example, the swift container will be named `restic`, with the backup data matching the hostname of the VM.

``` shell
openstack --os-cloud rxt-sjc-mine-free object list restic
```

!!! example "Swift Container Output"

    ``` shell
    +-----------------------------------------------------------------------------+
    | Name                                                                        |
    +-----------------------------------------------------------------------------+
    | app-0/config                                                                |
    | app-0/keys/1234567890123456789012345678901234567890123456789012345678901234 |
    +-----------------------------------------------------------------------------+
    ```

#### Check backups

To trigger a backup manually, run:

``` shell
sudo systemctl start restic-backup.service
```

Then, see if the snapshot was created:

``` shell
export $(grep -v '^#' /etc/restic/restic.env | xargs)
restic snapshots
```

!!! example "Restic Snapshots Output"

    ``` shell
    created new cache in /root/.cache/restic
    ID        Time                 Host        Tags        Paths  Size
    -----------------------------------------------------------------------
    9aa868eb  2025-03-27 06:29:36  app-0                   /home  4.525 KiB
    -----------------------------------------------------------------------
    1 snapshots
    ```

If you see snapshots listed, your backups are working and stored in Swift as intended.

## Next Steps

- **Add** a `restic forget --prune` step to remove old snapshots and manage disk usage in Swift.

- **Customize** the backup paths, schedules, and retention policies to suit your needs.

- **Extend the script**: Add more directories to back up and/or integrate with other services.

- **Monitor logs**: Keep an eye on `journalctl -u restic-backup.service` for issues.

- **Read the Restic documentation**: [Learn more about Restic’s features](https://restic.readthedocs.io) and how to use it effectively.

## Conclusion

By leveraging a simple cloud-init script, you can seamlessly install and configure Restic on a new OpenStack VM to back up any directory to Swift storage. Coupling Restic with Swift application credentials provides a secure, automated backup process that fits right into your OpenStack environment. With systemd timers handling schedules, you’ll have regular backups without manual intervention—ensuring your data is consistently protected.

Feel free to extend this script for pruning old backups, encrypting credentials at rest, or adopting more advanced rotation schemes. Happy backing up!
