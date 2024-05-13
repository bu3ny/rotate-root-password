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
