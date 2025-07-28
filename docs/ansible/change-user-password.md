#  Change User Password with Ansible

This Ansible playbook generates a random password for a specified user, sets it on the target Linux hosts, and saves the details to a local `passwords.csv` file. The password is marked as **expired**, forcing the user to change it at the first login.

---

##  Features

- Generates a random, secure password using `openssl`
- Applies the password to a specified user account on remote servers
- Forces password change at first login (password expiry).
- Saves the password and metadata (date, hostname, IP) to a local `passwords.csv` file

---


##  File Structure

```bash
.
├── change-user-password.yml
├── ansible.cfg
├── inventory
└── passwords.csv         # (created automatically after execution)
```

##  How to Use

Run the playbook with the required `user` variable:

```bash
git clone https://github.com/emrahuludag/change-user-password.git

ansible-playbook change-user-password.yml -e user=myuser
```

---
##  CSV Output
A file named passwords.csv will be created (or overwritten) on the local Ansible control machine with the following format:

```bash
Date,Hostname,IP,user,Password
2025-07-28,host01,192.168.1.10,myuser,XyZ1234secure
```

---
## Security Warning

- Plaintext passwords are saved in passwords.csv. Ensure this file is stored securely and deleted when no longer needed.

- Consider using Ansible Vault for secure storage in production environments.

- For production, avoid writing passwords to disk unless absolutely necessary.