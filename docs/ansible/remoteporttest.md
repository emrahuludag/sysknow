# Ansible ile Hedef Sunucularda Erişim Testi

Hayal edin ki yönettiğiniz 1000+ Linux sunucunuz var ve yeni bir uygulamayı devreye alıyorsunuz. Tüm erişim izinlerini açtınız, ancak bunu doğrulamanız ve teyit etmeniz gerekiyor. Ansible playbook'u kullanarak, wait\_for modülü sayesinde bu kontrolü hızlı bir şekilde gerçekleştirebilir ve önemli ölçüde zaman kazanabilirsiniz.

Aşağıda paylaştığım örnek, mevcut sistemlerinizde erişim kontrolünü rutin bir şekilde yapmanıza olanak tanır. 

```bash
git clone https://github.com/emrahuludag/remote-server-port-test.git
```

**Playbook Çalıştırılması**

```bash
ansible-playbook remote-server-port-test.yml -u ssh_username -kK
```

>[!TIP|style:flat]
> eğer ssh*key kullanıyorsanız  -Kk  sudo ve sudo pass'i atlayabilirsiniz.

**Playbook**

```bash
---
- name: Remote Server Application Port Test
  hosts: server
  gather_facts: false
  vars:
    web_server: 10.10.10.1
    syslog_server: 10.10.10.2
    monitoring_server: 10.10.10.3
  tasks:

   - name: Monitoring Server IP
     wait_for:
      host: "{{ monitoring_server }}"
      port: "{{ item }}"
      state: started         # Port should be open
      delay: 0               # No wait before first check (sec)
      timeout: 3             # Stop checking after timeout (sec)
     ignore_errors: true
     with_items:
       - 1515
       - 1514

   - name:  Web Server Port Connection Test
     wait_for:
      host: "{{ web_server }}"
      port: "{{ item }}"
      state: started
      delay: 0
      timeout: 3
     ignore_errors: true
     with_items:
       - 80
       - 443

   - name:  Log Collector Port Connection Test
     wait_for:
      host: "{{ syslog_server }}"
      port: "{{ item }}"
      state: started
      delay: 0
      timeout: 3
     ignore_errors: true
     with_items:
       - 514
```

**Outputs**

>[!TIP|style:flat]
> 'Timeout when waiting' hatası alıyorsanız, bu erişim olmadığı anlamına gelir

>[!TIP|style:flat]> Eğer 'ok: \[10.20.30.2\] => (item=80)' mesajını alıyorsanız, bu erişim sağladığınız anlamına gelir

```bash
#ansible-playbook remote-port-test.yml -u ssh_user -kK
SSH password: 
BECOME password[defaults to SSH password]: 

PLAY [Remote Server Application Port Test] **************************************************************************************************************************************************************

TASK [Monitoring Server IP] *****************************************************************************************************************************************************************************
failed: [10.20.30.1] (item=1510) => changed=false 
  ansible_facts:
    discovered_interpreter_python: /usr/libexec/platform-python
  ansible_loop_var: item
  elapsed: 4
  item: 1510
  msg: Timeout when waiting for 10.10.10.1:1510
failed: [10.20.30.3] (item=1510) => changed=false 
  ansible_facts:
    discovered_interpreter_python: /usr/libexec/platform-python
  ansible_loop_var: item
  elapsed: 4
  item: 1510
  msg: Timeout when waiting for 10.10.10.1:1510
failed: [10.20.30.2] (item=1510) => changed=false 
  ansible_facts:
    discovered_interpreter_python: /usr/libexec/platform-python
  ansible_loop_var: item
  elapsed: 4
  item: 1510
  msg: Timeout when waiting for 10.10.10.1:1510
failed: [10.20.30.2] (item=1511) => changed=false 
  ansible_loop_var: item
  elapsed: 4
  item: 1511
  msg: Timeout when waiting for 10.10.10.1:1511
...ignoring
failed: [10.20.30.1] (item=1511) => changed=false 
  ansible_loop_var: item
  elapsed: 4
  item: 1511
  msg: Timeout when waiting for 10.10.10.1:1511
...ignoring
failed: [10.20.30.3] (item=1511) => changed=false 
  ansible_loop_var: item
  elapsed: 4
  item: 1511
  msg: Timeout when waiting for 10.10.10.1:1511
...ignoring

TASK [Web Server Port Connection Test] ******************************************************************************************************************************************************************
ok: [10.20.30.2] => (item=80)
ok: [10.20.30.1] => (item=80)
ok: [10.20.30.3] => (item=80)
ok: [10.20.30.2] => (item=443)
ok: [10.20.30.1] => (item=443)
ok: [10.20.30.3] => (item=443)

TASK [Rsyslog Port Connection Test] *********************************************************************************************************************************************************************
ok: [10.20.30.2] => (item=514)
ok: [10.20.30.1] => (item=514)
failed: [10.20.30.3] (item=514) => changed=false 
  ansible_loop_var: item
  elapsed: 4
  item: 514
  msg: Timeout when waiting for 10.10.10.2:514
...ignoring

PLAY RECAP **********************************************************************************************************************************************************************************************
10.20.30.1               : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=1   
10.20.30.2               : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=1   
10.20.30.3              : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=2  
```

Örneğin bazı subnetleriniz spesifik sistemlerle sadece erişimi var bunu when ile kontrol de yapabiliriz. Örneğin Eğer ip adresi 192.168.x.x ile başlıyorsa x port erişimi test et taskı da yazabilirsiniz.

```bash
  when: ansible_default_ipv4.address is match('^192.168.*')
```

```bash
   - name: Monitoring Server IP
     wait_for:
      host: "{{ monitoring2_server }}"
      port: "{{ item }}"
      state: started         # Port should be open
      delay: 0               # No wait before first check (sec)
      timeout: 3             # Stop checking after timeout (sec)
     ignore_errors: true
     when: ansible_default_ipv4.address is match('^192.168.*')
     with_items:
       - 1515
       - 1514
```