# Ansible ile Linux sunucularda root user password değiştirilmesi

Yönetmekte olduğunuz sunucularda root user gibi kullanıcıların parolasının rutin olarak değiştirilmesi ciddi bir iştir. Bir Pam çözümünüz yoksa veya herhangi bir otomasyon toolu kullanmıyorsanız yönetmiş olduğunuz sistemin büyüklüğüne göre ciddi zaman kaybı ve efor getirmektedir. Aşağıdaki paylaşmış olduğum örnekte  **root** kullanıcısına ait parolayı hedef sunucularda değiştirir, expire düzenlemesi yapar ve switch user yaparak değiştirmiş olduğu parolayı teyit etmesini sağlar. Ayrıca playbook'u çalıştırdığınız sunucunuda bir csv dosyasında parolayı kayıt eder.


> [!Warning|style:flat]
> Ansible'da yeniyseniz production sistemlerde çalıştırmadan önce her zaman test ortamlarınızda test etmeniz önerilir.


> [!Attention|style:flat]
> Parola değiştirmek için hedef sunucularda çalıştıracağınız userı **-u username** ile belirtebilirsiniz. Burada çalıştırılan user ansible.cfg dosyasında remote_user: admin tanımı yapılarak playbok çalıştırılmıştır.

### Playbook Dosyaları
```bash
git clone https://github.com/emrahuludag/ansible-root-pass-change.git
```

### Playbook İçerik

```bash
change-ssh-passwd/
├── ansible.cfg
├── inventory
├── main.yml
├── readme.md
```

### Playbook Dosyası
```bash
cat main.yml

---
- hosts: servers
  vars:
   salt: "{{ lookup('pipe', 'openssl rand -hex 4') }}"
  tasks:

    - name: Change {{ username }} password
      user:
        name: "{{ username }}"
        update_password: always
        password: "{{ password | password_hash('sha512',salt) }}"

    - name: Create CSV File Header
      lineinfile:
        path: "./passwords.csv"
        line: "Date,Hostname,IP,Username,Password"
        create: true
      delegate_to: localhost

    - name: Save {{ username }} new password to local file
      lineinfile:
        path: "./passwords.csv"
        line: "{{ ansible_date_time.date }},{{ ansible_facts.hostname }},{{ inventory_hostname }},{{ username }},{{ password }}"
        create: true
      delegate_to: localhost

    - name: Set never expire password
      shell: "chage -I -1 -m 0 -M 99999 -E -1 {{ username }}"
 
    - name: Check user expire
      shell: chage -l "{{ username }}"
      register: expire
    
    - name: Chage {{ username }} result 
      debug:
        msg: "{{ expire.stdout }}" 
  
    - name: Switch to {{ username }} using shell
      become: no
      shell: echo "{{ password }}" | su -c 'whoami' {{ username }}
      no_log: true
      ignore_errors: yes
      register: su_result
   
    - name: Success Login {{ username }}  
      debug:
        msg: "{{ username }} login success"
      when: su_result.stdout ==  username 

    - name: Failure Login {{ username }}
      debug:
        msg: "{{ username }} login failure"
      when: "'Authentication failure' in su_result.stderr"

```

### Playbook Çalıştırılması
```bash
ansible-playbook main.yml -kK -e username=abc -e password=test124
```

### Playbook Output
```
SSH password: 
BECOME password[defaults to SSH password]: 

PLAY [servers] *************************************************************************************************************************************************************************************
Monday 24 June 2024  02:01:46 +0300 (0:00:00.009)       0:00:00.009 *********** 

TASK [Gathering Facts] *****************************************************************************************************************************************************************************
ok: [10.0.0.1]
Monday 24 June 2024  02:01:51 +0300 (0:00:04.437)       0:00:04.446 *********** 

TASK [Change abc password] *************************************************************************************************************************************************************************
changed: [10.0.0.1]
Monday 24 June 2024  02:01:53 +0300 (0:00:02.767)       0:00:07.214 *********** 

TASK [Create CSV File Header] **********************************************************************************************************************************************************************
ok: [10.0.0.1 -> localhost]
Monday 24 June 2024  02:01:54 +0300 (0:00:00.354)       0:00:07.568 *********** 

TASK [Save abc new password to local file] *********************************************************************************************************************************************************
changed: [10.0.0.1 -> localhost]
Monday 24 June 2024  02:01:54 +0300 (0:00:00.222)       0:00:07.790 *********** 

TASK [never expire password] ***********************************************************************************************************************************************************************
changed: [10.0.0.1]
Monday 24 June 2024  02:01:56 +0300 (0:00:02.338)       0:00:10.129 *********** 

TASK [check user expire] ***************************************************************************************************************************************************************************
changed: [10.0.0.1]
Monday 24 June 2024  02:01:58 +0300 (0:00:02.101)       0:00:12.231 *********** 

TASK [Chage abc result] ****************************************************************************************************************************************************************************
ok: [10.0.0.1] => 
  msg: |-
    Last password change                                    : Jun 23, 2024
    Password expires                                        : never
    Password inactive                                       : never
    Account expires                                         : never
    Minimum number of days between password change          : 0
    Maximum number of days between password change          : 99999
    Number of days of warning before password expires       : 7
Monday 24 June 2024  02:01:58 +0300 (0:00:00.026)       0:00:12.257 *********** 

TASK [Switch to abc using shell] *******************************************************************************************************************************************************************
changed: [10.0.0.1]
Monday 24 June 2024  02:02:01 +0300 (0:00:02.069)       0:00:14.327 *********** 

TASK [Success Login abc] ***************************************************************************************************************************************************************************
ok: [10.0.0.1] => 
  msg: abc login success
Monday 24 June 2024  02:02:01 +0300 (0:00:00.027)       0:00:14.354 *********** 

PLAY RECAP *****************************************************************************************************************************************************************************************
10.0.0.1                : ok=9    changed=5    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   

Monday 24 June 2024  02:02:01 +0300 (0:00:00.047)       0:00:14.402 *********** 
=============================================================================== 
Gathering Facts ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 4.44s
Change abc password ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 2.77s
Create CSV File Header ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.35s
Save abc new password to local file --------------------------------------------------------------------------------------------------------------------------------------------------------- 0.22s
never expire password ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- 2.34s
check user expire --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 2.10s
Chage abc result ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.03s
Switch to abc using shell ------------------------------------------------------------------------------------------------------------------------------------------------------------------- 2.07s
Success Login abc --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.03s
Failure Login abc --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.05s
Playbook run took 0 days, 0 hours, 0 minutes, 14 seconds
```


```
**cat password.csv**
Date,Hostname,IP,Username,Password
2024-06-24,srv01,10.0.0.1,root,test123
2024-06-24,srv02,10.0.0.2,root,test123
```

---
## Ansible Vault Kullanımı
Root parolasını vault ile secret dosyası oluşturarak playbook'un buradan okumasını sağlayabilirsiniz. main.yaml dosyasında aşağıdaki satır eklenir ve vault oluşturduktan sonra playbook çalıştırılır.

**main.yaml dosyanızda;**
```bash
- hosts: servers
  vars_files:   <-- Satırı ekle 
    - ./vault.yml <-- Satırı ekle 
```

**Vault oluşturulması;**

```bash
**ansible-vault create vault.yml**
New Vault password: 
Confirm New Vault password: 
```

```bash
Vault değiştirilmesi;

**ansible-vault edit vault.yml**
```

**Playbook çalıştırılması**
```bash
**echo "vault-pass" > password.txt**
**chmod 600 password.txt**
**ansible-playbook main.yml -kK -e username=root --vault-password-file=password.txt**
```
---