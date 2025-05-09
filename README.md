# Ansible Role: Postfix SMTP Relay

This role configures a Postfix server to act as an SMTP relay (smarthost), typically to forward emails through an authenticated upstream mail provider.

## Requirements

*   **Ansible:** Version 2.9 or higher.
*   **Target OS:** Enterprise Linux 8 or 9 (e.g., RHEL, CentOS, Rocky Linux, AlmaLinux). The role uses `dnf` for package management and `firewalld` for firewall configuration.
*   **Postfix and Cyrus SASL:** These packages will be installed by the role.

## Role Variables

The role uses variables to customize the Postfix configuration. Default values are defined in `roles/postfix_relay/defaults/main.yml`. It is highly recommended to override sensitive variables, especially `postfix_smarthost_pass`, using Ansible Vault.

### Key Variables:

*   `postfix_relay_hostname`: (String) The hostname of the relay server.
    *   Default: `"relay01.int.example"`
*   `postfix_relay_mynetworks`: (List of strings) A list of IP addresses or networks (CIDR notation) that are allowed to send email through this relay without SMTP authentication.
    *   Default:
        ```yaml
        - "127.0.0.0/8"  # localhost
        - "10.10.20.15"  # Example internal server
        ```
*   `postfix_smarthost`: (String) The hostname or IP address of the upstream smarthost (e.g., your email provider's SMTP server).
    *   Default: `"smtp.mail.cloud.local"`
*   `postfix_smarthost_port`: (Integer) The port number for the upstream smarthost.
    *   Common values: `25` (SMTP), `587` (Submission with STARTTLS), `465` (SMTPS - implicit TLS).
    *   Default: `25`
*   `postfix_smarthost_user`: (String) The username for authenticating with the upstream smarthost.
    *   Default: `"customer-123"`
*   `postfix_smarthost_pass`: (String) The password for authenticating with the upstream smarthost.
    *   **IMPORTANT:** This is a sensitive value. **ALWAYS encrypt this variable using Ansible Vault.**
    *   Default: `"S3cr3t!"` (Placeholder - **DO NOT USE IN PRODUCTION WITHOUT VAULTING**)
*   `postfix_tls_security_level`: (String) Controls TLS encryption for outgoing mail to the smarthost.
    *   `encrypt`: Require STARTTLS encryption. The connection will fail if TLS cannot be negotiated.
    *   `may`: Opportunistic TLS. Use TLS if available, but send in cleartext if not.
    *   `none`: Disable TLS.
    *   Default: `"encrypt"`
*   `postfix_smarthost_ca_file`: (String) Path to the CA certificate bundle for verifying the smarthost's TLS certificate.
    *   Default: `"/etc/ssl/certs/ca-bundle.crt"` (Common for RHEL-based systems)
    *   *Note: This was added as an improvement. Ensure your `main.cf.j2` template uses this variable: `smtp_tls_CAfile = {{ postfix_smarthost_ca_file }}`.*
*   `postfix_packages`: (List of strings) The list of packages to install for Postfix and SASL authentication.
    *   Default:
        ```yaml
        - postfix
        - cyrus-sasl
        - cyrus-sasl-plain
        ```

## How to Use

1.  **Inventory Setup:**
    Define your relay server(s) in an inventory file (e.g., `inventory/hosts.yml`):
    ```yaml
    # inventory/hosts.yml
    all:
      children:
        relay_servers:
          hosts:
            your-relay-server.example.com:
    ```

2.  **Variable Configuration (Ansible Vault for Secrets):**
    It's best practice to store sensitive information like `postfix_smarthost_pass` and potentially `postfix_smarthost_user` in a vaulted file.

    Create a file, for example, `group_vars/relay_servers.yml` (or `host_vars/your-relay-server.example.com.yml`):
    ```yaml
    # group_vars/relay_servers.yml
    postfix_relay_hostname: "myrelay.mydomain.com"
    postfix_smarthost: "smtp.yourprovider.com"
    postfix_smarthost_port: 587
    postfix_smarthost_user: "user@yourprovider.com"
    postfix_smarthost_pass: "YourActualPassword"
    postfix_tls_security_level: "encrypt"
    postfix_relay_mynetworks:
      - "127.0.0.0/8"
      - "192.168.1.0/24" # Allow your internal network
    ```

    Encrypt this file using Ansible Vault:
    ```bash
    ansible-vault encrypt group_vars/relay_servers.yml
    ```
    You will be prompted to set a vault password.

3.  **Playbook Creation:**
    Create a playbook to apply the `postfix_relay` role to your target hosts (e.g., `playbooks/relay.yml`):
    ```yaml
    # playbooks/relay.yml
    ---
    - name: Configure Postfix SMTP Relay
      hosts: relay_servers
      become: true
      roles:
        - postfix_relay
    ```

4.  **Run the Playbook:**
    Execute the playbook. If you used Ansible Vault, you'll need to provide the vault password.
    ```bash
    ansible-playbook -i inventory/hosts.yml playbooks/relay.yml --ask-vault-pass
    ```
    Or, if your vault password is in a file:
    ```bash
    ansible-playbook -i inventory/hosts.yml playbooks/relay.yml --vault-password-file ~/.vault_pass.txt
    ```

## Example Playbook

The file `playbooks/relay.yml` provided in this repository is a ready-to-use example:
```yaml
# playbooks/relay.yml
---
- name: Configure Postfix SMTP Relay
  hosts: relay # Assumes 'relay' is a group in your inventory
  become: true
  roles:
    - postfix_relay
```

## Testing the Relay

Once the playbook has run successfully, you can test the relay by sending an email from a machine listed in `postfix_relay_mynetworks` (or from the relay server itself).

1.  **Log in to the relay server or an allowed client machine.**
2.  **Use the `mail` command:**
    ```bash
    echo "This is a test email body." | mail -s "Test Email from Ansible Relay" recipient@example.com
    ```
    Replace `recipient@example.com` with a valid email address you can check.

3.  **Check Postfix Logs on the Relay Server:**
    Monitor the mail log for activity. The location of the log file can vary, but it's often `/var/log/maillog` or accessible via `journalctl`.
    ```bash
    sudo tail -f /var/log/maillog
    ```
    Or, for systems using systemd journals:
    ```bash
    sudo journalctl -u postfix -f
    ```
    Look for lines indicating the connection to the smarthost, authentication, and successful sending. Example success message:
    ```
    postfix/smtp[...]: A1B2C3D4E5: to=<recipient@example.com>, relay=smtp.yourprovider.com[1.2.3.4]:587, delay=..., delays=..., dsn=2.0.0, status=sent (250 2.0.0 Ok: queued as ABCDEF12345)
    ```

## Role Tasks Overview

The `roles/postfix_relay/tasks/main.yml` file performs the following actions:

1.  **Installs Packages:** Uses `ansible.builtin.dnf` to install `postfix`, `cyrus-sasl`, and `cyrus-sasl-plain`.
2.  **Ensures Postfix Service:** Enables and starts the `postfix` service.
3.  **Deploys `main.cf`:** Templates the Postfix main configuration file (`/etc/postfix/main.cf`) using `roles/postfix_relay/templates/main.cf.j2`. Notifies a handler to restart Postfix if this file changes.
4.  **Deploys `sasl_passwd`:** Templates the SASL password file (`/etc/postfix/sasl_passwd`) using `roles/postfix_relay/templates/sasl_passwd.j2`. This file contains the credentials for the upstream smarthost. Notifies a handler to run `postmap` on this file and then restart Postfix if it changes. The file permissions are set to `0600` (owner read/write only) for security.
5.  **Opens Firewall Port:** Uses `ansible.posix.firewalld` to open the SMTP port (25/tcp) if the target system is RedHat family.

## Handlers

Defined in `roles/postfix_relay/handlers/main.yml`:

*   **`Postmap sasl_passwd`**: Runs the `postmap /etc/postfix/sasl_passwd` command to create or update the Postfix lookup table from the `sasl_passwd` text file. This is necessary for Postfix to use the credentials. Notifies the `Restart postfix` handler.
*   **`Restart postfix`**: Restarts the `postfix` service to apply configuration changes.

## Meta Information

From `roles/postfix_relay/meta/main.yml`:

*   **Author:** Ricardo Arguello
*   **Description:** Configures a Postfix SMTP relay (smarthost).
*   **License:** MIT (or your preferred license)
*   **Minimum Ansible Version:** 2.9
*   **Supported Platforms:**
    *   EL (Enterprise Linux)
        *   Versions: 8, 9

## Original Commands (from previous README)

These commands are useful for direct interaction and testing:

*   **Run playbook:**
    ```bash
    # cd email-relay-ansible # (If not already in the project root)
    ansible-playbook -i inventory/hosts.yml playbooks/relay.yml --ask-vault-pass
    ```

*   **Encrypt variables (example):**
    ```bash
    ansible-vault encrypt group_vars/relay_servers.yml
    ```
    (Adjust filename as needed, e.g., `group_vars/relay.yml` if that's what you use)

*   **Test mail sending from relay (shell command):**
    ```bash
    ansible -i inventory/hosts.yml relay_servers -m shell -a "echo 'mail test' | mail -s 'ansible-relay-test' you@example.com" --ask-vault-pass
    ```
    (Replace `relay_servers` with your inventory group name if different)

*   **Watch logs on relay:**
    ```bash
    # On the relay server:
    # sudo tail -f /var/log/maillog
    # or
    # sudo journalctl -u postfix -f
    ```
    Look for output similar to:
    ```
    postfix/smtp[...] STARTTLS wrapper started
    ... auth=plain, status=sent (250 2.0.0 Ok: queued)
    ```

This comprehensive README should provide a good understanding of the role and how to use it effectively.
