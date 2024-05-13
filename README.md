# rotate-root-password
These set of playbooks can be used to:
- Rotate root password on A large number of servers using Ansible.
  - First, it'll setup the localhost "control node" by installed MySQL server, add DB and tables needed to store all changed passwords and exection logs.
  - Then, it'll filter the hosts (ignore unreachable hosts and hosts with no SUDO privileges).
  - At last, it'll generate random passwords (a different password for each host and apply the changes).
  - It'll also save or update the DB entries to always ensure that latest password changes incorporated (Passwords are encrypted using AES256 then encoded as base64 before saving it into the DB).
  - The script ensure that a newly created CSV file (encrypted using AES256) is added everytime the script is executed and any exisiting files will be backedup.
  - Save another entry with the exection results (failed servers, unreachable servers, servers with no SUDO privileges for the useraccount exectuing it).
  - Send email notification to the user (ideally admin) with the exection summary.
  - Clear any sensitive files (un encrypted files).

 - Print the passwords of all server in json format.
 - Print the passwords of a single server (it'll prompt the user to enter the server name or IP and search the DB).
 - Print the exection logs (including the history) - this is useful if you'd like to share the exection summary with the audit team.

# What you need to do to make it working?
- Create vars directory like below:
    ~~~
    [root@server change-root-password-playbooks]# tree vars
    vars
    ├── my_variables.yml
    └── secret_vars.yml

    0 directories, 2 files
    ~~~
- Define the following variables as per your choice (use strong passwords):
    ~~~
    [root@server change-root-password-playbooks]# cat vars/my_variables.yml
    # Store all configuration variables here
    base_file: "~/root-password-rotation/.tmp/password_changes.csv"
    encrypted_csv_file: "~/root-password-rotation/encrypted_passwords/encrypted_passwords.enc"
    execution_logs_file: "~/root-password-rotation/execution_logs/execution_logs.txt"
    [root@server change-root-password-playbooks]# ansible-vault view vars/secret_vars.yml
    Vault password:
    vault_mysql_root_password: 'redhat'
    vault_db_name: 'server_passwords'
    vault_db_root_user: 'root'
    vault_db_root_password: 'redhat'
    vault_db_user: 'dbadmin'
    vault_db_user_password: 'redhat'
    vault_encryption_key: 'redhat'
    vault_relayhost: '10.8.109.238'
    vault_password: 'redhat'
    ~~~
- Define hosts and ansible.cfg files like below:
    ~~~
    [root@server change-root-password-playbooks]# cat hosts
    10.8.15.238
    10.10.10.201
    10.0.0.1
    10.0.0.2
    10.0.0.3
    10.0.0.4
    10.0.0.5

    [root@server change-root-password-playbooks]# cat ansible.cfg
    [defaults]
    forks = 20
    #interpreter_python  = /usr/bin/python3
    action_warnings = false
    inventory = ./hosts
    ~~~

- At last, you may need to adjust the main playbook to define the user who will be used for the exection:
    ~~~
    remote_user: mostafa
    ~~~

- On the managed hosts (on which the root password needs to be changed) - make sure that this user has sudo privileges:
  ~~~
  mostafa ALL=(ALL) NOPASSWD: ALL
  ~~~

- Execute the playbooks:
  ~~~
  # ansible-playbook --ask-vault-pass --ask-pass rotate-passwords.yaml
  # ansible-playbook --ask-vault-pass print-all-passwords.yaml
  ...
  ~~~
# Below is an example run for my test setup:
- I intentionally created the hosts file with a list of servers who are unreachable, a host that has no sudo privileges for the connection user and a host that is reachable and has enough SUDO privileges:
    ~~~
    [root@server change-root-password-playbooks]# tree
    .
    ├── ansible.cfg
    ├── rotate-passwords.yaml-undocumented.yaml
    ├── rotate-passwords.yaml
    ├── hosts
    ├── print-all-passwords.yaml
    ├── print-logs.yaml
    ├── print-server-password.yaml
    └── vars
        ├── my_variables.yml
        └── secret_vars.yml

    1 directory, 9 files


    [root@idm-1 change-root-password-playbooks]# ansible-playbook --ask-vault-pass --ask-pass rotate-passwords.yaml
    SSH password:
    Vault password:

    PLAY [Preparing the control node] ***********************************************************************************************************************************************************

    TASK [Gathering Facts] **********************************************************************************************************************************************************************
    ok: [localhost]

    TASK [Ensure directories exist] *************************************************************************************************************************************************************
    ok: [localhost] => (item=None)
    ok: [localhost] => (item=None)
    ok: [localhost] => (item=None)
    ok: [localhost]

    TASK [Check if files exist and gather stats] ************************************************************************************************************************************************
    ok: [localhost] => (item=None)
    ok: [localhost] => (item=None)
    ok: [localhost]

    TASK [Backup old encrypted CSV and Unreachable hosts files if they exist] *******************************************************************************************************************
    changed: [localhost] => (item=None)
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Securely delete old encrypted CSV and Unreachable hosts files if they exist] **********************************************************************************************************
    changed: [localhost] => (item=None)
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Gather the package facts] *************************************************************************************************************************************************************
    ok: [localhost]

    TASK [Install MySQL server (Red Hat-based systems)] *****************************************************************************************************************************************
    skipping: [localhost]

    TASK [Install other required packages (Red Hat-based systems)] ******************************************************************************************************************************
    skipping: [localhost] => (item=None)
    skipping: [localhost] => (item=None)
    skipping: [localhost] => (item=None)
    skipping: [localhost] => (item=None)
    skipping: [localhost] => (item=None)
    skipping: [localhost] => (item=None)
    skipping: [localhost] => (item=None)
    skipping: [localhost] => (item=None)
    skipping: [localhost]

    TASK [Configure main.cf for Postfix] ********************************************************************************************************************************************************
    ok: [localhost] => (item={'line': 'relayhost = [10.8.109.238]'})

    TASK [Ensure MySQL root user can login with password] ***************************************************************************************************************************************
    ok: [localhost]

    TASK [Create database user with all privileges] *********************************************************************************************************************************************
    ok: [localhost]

    TASK [Check if DB tables exist and create them if not exist] ********************************************************************************************************************************
    ok: [localhost] => (item=None)
    ok: [localhost] => (item=None)
    ok: [localhost]

    PLAY [Classify hosts based on reachability] *************************************************************************************************************************************************

    TASK [Gathering Facts] **********************************************************************************************************************************************************************
    ok: [10.8.109.238]
    fatal: [10.10.74.201]: FAILED! => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result"}
    ...ignoring
    fatal: [10.0.0.1]: UNREACHABLE! => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}
    fatal: [10.0.0.2]: UNREACHABLE! => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}
    fatal: [10.0.0.3]: UNREACHABLE! => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}
    fatal: [10.0.0.4]: UNREACHABLE! => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}
    fatal: [10.0.0.5]: UNREACHABLE! => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}

    TASK [Check reachability via ping] **********************************************************************************************************************************************************
    ok: [10.8.109.238]
    fatal: [10.10.74.201]: FAILED! => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result"}
    ...ignoring

    TASK [Group by reachability] ****************************************************************************************************************************************************************
    changed: [10.8.109.238]
    fatal: [10.10.74.201]: FAILED! => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result"}
    ...ignoring

    TASK [Setting a list of unreachable hosts] **************************************************************************************************************************************************
    ok: [10.8.109.238]
    ok: [10.10.74.201]

    TASK [Setting a list of other facts to log into the DB] *************************************************************************************************************************************
    ok: [10.8.109.238 -> localhost]

    PLAY [Classify hosts based on Sudo Access] **************************************************************************************************************************************************

    TASK [Check sudo access] ********************************************************************************************************************************************************************
    changed: [10.8.109.238]
    fatal: [10.10.74.201]: FAILED! => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result"}
    ...ignoring

    TASK [Add hosts without sudo access to sudo_failed_hosts] ***********************************************************************************************************************************
    ok: [10.10.74.201 -> localhost]

    PLAY [Generating and changing root password for reachable hosts] ****************************************************************************************************************************

    TASK [Gathering facts] **********************************************************************************************************************************************************************
    ok: [10.8.109.238]

    TASK [Generate random password for each reachable host] *************************************************************************************************************************************
    ok: [10.8.109.238 -> localhost]

    TASK [Generate hashed password using mkpasswd] **********************************************************************************************************************************************
    ok: [10.8.109.238 -> localhost] => (item=None)
    ok: [10.8.109.238 -> localhost]

    TASK [Apply password change using chpasswd] *************************************************************************************************************************************************
    changed: [10.8.109.238] => (item=None)
    changed: [10.8.109.238]

    TASK [Add hosts where password change failed to failed_hosts group] *************************************************************************************************************************
    skipping: [10.8.109.238]

    TASK [Encrypt password using OpenSSL and encode in base64 with PBKDF2] **********************************************************************************************************************
    changed: [10.8.109.238 -> localhost]

    TASK [Store encrypted password and timestamp in database] ***********************************************************************************************************************************
    changed: [10.8.109.238 -> localhost] => (item=None)
    changed: [10.8.109.238 -> localhost]

    TASK [Create CSV file with username, password changes, and timestamp] ***********************************************************************************************************************
    changed: [10.8.109.238 -> localhost]

    TASK [Setting refined facts] ****************************************************************************************************************************************************************
    ok: [10.8.109.238 -> localhost]

    TASK [Insert execution data into the database] **********************************************************************************************************************************************
    changed: [10.8.109.238 -> localhost]

    TASK [Fetch execution log values from the database] *****************************************************************************************************************************************
    ok: [10.8.109.238 -> localhost]

    TASK [Convert failed hosts to list] *********************************************************************************************************************************************************
    ok: [10.8.109.238 -> localhost]

    TASK [Convert sudo failed hosts to list] ****************************************************************************************************************************************************
    ok: [10.8.109.238 -> localhost]

    TASK [Send email with playbook execution recap and decryption instructions] *****************************************************************************************************************
    ok: [10.8.109.238 -> localhost]

    TASK [Encrypt CSV file using OpenSSL] *******************************************************************************************************************************************************
    changed: [10.8.109.238 -> localhost]

    TASK [Check if unencrypted files exist] *****************************************************************************************************************************************************
    ok: [10.8.109.238 -> localhost] => (item=None)
    ok: [10.8.109.238 -> localhost]

    TASK [Securely delete Plaintext unencrypted files if exist] *********************************************************************************************************************************
    changed: [10.8.109.238 -> localhost] => (item=None)
    changed: [10.8.109.238 -> localhost]

    PLAY RECAP **********************************************************************************************************************************************************************************
    10.0.0.1                   : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
    10.0.0.2                   : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
    10.0.0.3                   : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
    10.0.0.4                   : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
    10.0.0.5                   : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
    10.10.74.201               : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=4
    10.8.109.238               : ok=22   changed=9    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
    localhost                  : ok=10   changed=2    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0

    [root@server change-root-password-playbooks]#
    ~~~

- Printing the passwords for all servers - I fixed the sudo privileges on the second host. Now, it's listed in the DB:
    ~~~
    [root@server change-root-password-playbooks]# ansible-playbook --ask-vault-pass print-all-passwords.yaml
    Vault password:

    PLAY [Fetch and Decrypt All Server Password Information] ************************************************************************************************************************************

    TASK [Query all encrypted passwords and timestamps from server_details] *********************************************************************************************************************
    ok: [localhost]

    TASK [Decrypt passwords and collect details] ************************************************************************************************************************************************
    changed: [localhost] => (item=None)
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [set_fact] *****************************************************************************************************************************************************************************
    ok: [localhost] => (item=None)
    ok: [localhost] => (item=None)
    ok: [localhost]

    TASK [Display server details] ***************************************************************************************************************************************************************
    ok: [localhost] => {
        "msg": [
            {
                "changed_at": "2024-05-08T11:22:42",
                "hostname": "server-2",
                "ip": "10.10.74.201",
                "password": "NOzsgR9pxKhYNUkpblg5M3Su"
            },
            {
                "changed_at": "2024-05-13T11:59:30",
                "hostname": "server-1",
                "ip": "10.8.109.238",
                "password": "onw75iUxcXJOdg2kKu9ZKJd7"
            }
        ]
    }

    PLAY RECAP **********************************************************************************************************************************************************************************
    localhost                  : ok=4    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    ~~~
- Got an email like below:

![image](https://github.com/bu3ny/rotate-root-password/assets/10407203/f27e0861-2303-4ba0-812e-7ef28b982db5)

    
