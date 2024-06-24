
![](./img/rootpass.png? ':size=50%')

# Ansible ile Linux sunucularda root user random password ile değiştirilmesi


[Ansible ile Linux sunucularda root user password değiştirilmesi](https://emrahuludag.github.io/sysknow/#/ansible/change-ssh-password) konusuna sunucularda root user parolasının değiştirilmesini ele almıştık. Burada daha complex haline getirip tüm sunucularda root parolasını **farklı** olacak şekilde değiştirilmesi ve bu parolaların daha sonra safe manager'a eklenmesi için bir CSV file'a export alınacaktır.

> [!Warning|style:flat]
> Ansible'da yeniyseniz production sistemlerde çalıştırmadan önce her zaman test ortamlarınızda test etmeniz önerilir.


> [!Attention|style:flat]
> Root user parolası otomatik olarak generate edilip password.csv dosyasına yazılır.

> [!TIP|style:flat]
> password complex oluşturmak için aşağıdaki satırı kullanabilirsiniz ancak console erişimlerinde special karakterler ciddi zorluk çıkaracaktır.
>
> generate_root_password: "{{ lookup('community.general.random_string', 'length=12 chars=ascii_letters,digits') }}"

### Playbook Dosyaları
```bash
git clone https://github.com/emrahuludag/ansible-change-root-password-randomly.git
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
**main.yml**
```bash
---
- hosts: servers
  become: yes
  vars:
   username: root
  tasks:
    
    - name: Generate root password
      set_fact:
        generate_root_password: "{{ lookup('pipe', 'openssl rand -base64 12 | tr -dc A-Za-z0-9') }}"
        salt: "{{ lookup('pipe', 'openssl rand -hex 4') }}"

    - name: Change {{ username }} password
      user:
        name: "{{ username }}"
        update_password: always
        password: "{{ generate_root_password | password_hash('sha512', salt ) }}"

    - name: Create CSV File Header
      lineinfile:
        path: "./passwords.csv"
        line: "Date,Hostname,IP,Username,Password"
        create: true
      delegate_to: localhost

    - name: Save {{ username }} new password to local file
      lineinfile:
        path: "./passwords.csv"
        line: "{{ ansible_date_time.date }},{{ ansible_facts.hostname }},{{ inventory_hostname }},{{ username }},{{ generate_root_password }}"
        create: true
      delegate_to: localhost

    - name: never expire password
      shell: "chage -I -1 -m 0 -M 99999 -E -1 {{ username }}"
 
    - name: check user expire
      shell: chage -l "{{ username }}"
      register: expire
    
    - name: Chage {{ username }} result 
      debug:
        msg: "{{ expire.stdout }}" 
  
    - name: Switch to {{ username }} using shell
      become: no
      shell: echo "{{ generate_root_password }}" | su -c 'whoami' {{ username }}
      no_log: true
      ignore_errors: yes
      register: su_result
      tags:
       - check-pass     
    
    - name: Success Login {{ username }}  
      debug:
        msg: "{{ username }} login success"
      when: su_result.stdout ==  username 
      tags:
       - check-pass
    
    - name: Failure Login {{ username }}
      debug:
        msg: "{{ username }} login failure"
      when: "'Authentication failure' in su_result.stderr"
      tags:
        - check-pass

```

### Playbook Çalıştırılması
```bash
ansible-playbook main.yml -kK -u admin
```
### Playbook Output
```
SSH password: 
BECOME password[defaults to SSH password]: 

PLAY [servers] *************************************************************************************************************************************************************************************************************************************************
Monday 24 June 2024  12:36:36 +0300 (0:00:00.009)       0:00:00.009 *********** 

TASK [Gathering Facts] *****************************************************************************************************************************************************************************************************************************************
ok: [10.0.0.2]
ok: [10.0.0.3]
ok: [10.0.0.1]
Monday 24 June 2024  12:36:46 +0300 (0:00:09.649)       0:00:09.659 *********** 

TASK [Generate root password] **********************************************************************************************************************************************************************************************************************************
ok: [10.0.0.1]
ok: [10.0.0.2]
ok: [10.0.0.3]
Monday 24 June 2024  12:36:46 +0300 (0:00:00.094)       0:00:09.754 *********** 

TASK [Change root password] ************************************************************************************************************************************************************************************************************************************
changed: [10.0.0.2]
changed: [10.0.0.1]
changed: [10.0.0.3]
Monday 24 June 2024  12:36:48 +0300 (0:00:02.533)       0:00:12.287 *********** 

TASK [Create CSV File Header] **********************************************************************************************************************************************************************************************************************************
changed: [10.0.0.2 -> localhost]
ok: [10.0.0.1 -> localhost]
ok: [10.0.0.3 -> localhost]
Monday 24 June 2024  12:36:49 +0300 (0:00:00.433)       0:00:12.721 *********** 

TASK [Save root new password to local file] ********************************************************************************************************************************************************************************************************************
changed: [10.0.0.1 -> localhost]
changed: [10.0.0.3 -> localhost]
changed: [10.0.0.2 -> localhost]
Monday 24 June 2024  12:36:49 +0300 (0:00:00.255)       0:00:12.976 *********** 

TASK [never expire password] ***********************************************************************************************************************************************************************************************************************************
changed: [10.0.0.2]
changed: [10.0.0.1]
changed: [10.0.0.3]
Monday 24 June 2024  12:36:52 +0300 (0:00:02.409)       0:00:15.386 *********** 

TASK [check user expire] ***************************************************************************************************************************************************************************************************************************************
changed: [10.0.0.2]
changed: [10.0.0.1]
changed: [10.0.0.3]
Monday 24 June 2024  12:36:54 +0300 (0:00:02.390)       0:00:17.777 *********** 

TASK [Chage root result] ***************************************************************************************************************************************************************************************************************************************
ok: [10.0.0.1] => 
  msg: |-
    Last password change                                    : Jun 24, 2024
    Password expires                                        : never
    Password inactive                                       : never
    Account expires                                         : never
    Minimum number of days between password change          : 0
    Maximum number of days between password change          : 99999
    Number of days of warning before password expires       : 7
ok: [10.0.0.2] => 
  msg: |-
    Last password change                                    : Jun 24, 2024
    Password expires                                        : never
    Password inactive                                       : never
    Account expires                                         : never
    Minimum number of days between password change          : 0
    Maximum number of days between password change          : 99999
    Number of days of warning before password expires       : 7
ok: [10.0.0.3] => 
  msg: |-
    Last password change                                    : Jun 24, 2024
    Password expires                                        : never
    Password inactive                                       : never
    Account expires                                         : never
    Minimum number of days between password change          : 0
    Maximum number of days between password change          : 99999
    Number of days of warning before password expires       : 7
Monday 24 June 2024  12:36:54 +0300 (0:00:00.056)       0:00:17.833 *********** 

TASK [Switch to root using shell] ******************************************************************************************************************************************************************************************************************************
changed: [10.0.0.2]
changed: [10.0.0.1]
changed: [10.0.0.3]
Monday 24 June 2024  12:36:56 +0300 (0:00:02.306)       0:00:20.139 *********** 

TASK [Success Login root] **************************************************************************************************************************************************************************************************************************************
ok: [10.0.0.1] => 
  msg: root login success
ok: [10.0.0.2] => 
  msg: root login success
ok: [10.0.0.3] => 
  msg: root login success
Monday 24 June 2024  12:36:56 +0300 (0:00:00.046)       0:00:20.186 *********** 

PLAY RECAP *****************************************************************************************************************************************************************************************************************************************************
10.0.0.1                : ok=10   changed=5    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
10.0.0.1                : ok=10   changed=6    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
10.0.0.3                 : ok=10   changed=5    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   

Monday 24 June 2024  12:36:56 +0300 (0:00:00.090)       0:00:20.277 *********** 
=============================================================================== 
Gathering Facts ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 9.65s
Generate root password ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.09s
Change root password ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ 2.53s
Create CSV File Header ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.43s
Save root new password to local file -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.26s
never expire password ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 2.41s
check user expire --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 2.39s
Chage root result --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.06s
Switch to root using shell ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ 2.31s
Success Login root -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.05s
Failure Login root -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.09s
Playbook run took 0 days, 0 hours, 0 minutes, 20 seconds
```

**password.csv**
```bash
2024-06-24,testsrv01,10.0.0.1,root,tQOLfz8FSppdtckp
2024-06-24,testsrv03,10.0.0.3,root,wYz9vxkj9wu1eO3
2024-06-24,testsrv02,10.0.0.2,root,m7acjySdPYbGfeL4
```