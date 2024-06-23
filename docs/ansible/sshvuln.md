# Ansible İle SSH Servisindeki CBC KexAlgorithms Bulguların Kapatılması
![](./img/ansiblerhellogo.png)


Zafiyet tarama testlerinde LOW severity de bulunan **SSH Server CBC Mode Ciphers Enabled** ve **SSH Weak Key Exchange Algorithms Enabled** bulgularını tek tek tüm sunucuların sshd servisinde işlem yapmak yerine ansible ile uzaktan yapılandırmasını sağlayacağız. Bulguları kapatmak için /etc/ssh/sshd\_config  config dosyasında secure Ciphers ve KexAlgorithms parametrelerini girip ssh servisini restart edeceğiz. Yapılandırma için ansible lineinfile modülünü kullanacağız. Daha önceki lineinfile modülü ile ilgili yazımıza linkten ulaşabilirsiniz. *(*[*bknz)*](https://wiki.acikkaynakfikirler.com/e/tr/ansible/Ansible-ile-yapilandirma) 

Bulgular hakkında bilgi için ;

**SSH Weak Key Exchange Algorithms Enabled >** https://www.tenable.com/plugins/nessus/153953

**SSH Server CBC Mode Ciphers Enabled >** https://www.tenable.com/plugins/nessus/70658

### **YML Dosyasın Oluşturulması**

YML dosyasında öncelikle sshd\_config dosyasının güncel tarihli yedeğini alıyoruz.

Daha sonra ilgili parametretleri güncelleyip sshd servisi reload ederek config değişikliğini algılamasını sağlıyoruz.

 sshd\_hardening.yml  dosyası

```bash
---
- name: Fix SSH Weak Key Exchange Algorithms, SSH Weak Key Exchange Algorithms Enabled, and MACs Enabled
  hosts: local
  become: yes

  tasks:

  - name: Backup sshd_config  {{ ansible_fqdn }}
    copy: 
     src: "{{ item.src }}"
     dest: "{{ item.dest }}"
     remote_src: yes
    with_items:
     - { src: '/etc/ssh/sshd_config', dest: '/etc/ssh/sshd_config{{ansible_date_time.date}}_ansiblebackup' }

  - name: Adding KexAlgorithms line {{ ansible_fqdn }}
    lineinfile:
      path: /etc/ssh/sshd_config
      regexp: '^KexAlgorithms'
      line: 'KexAlgorithms diffie-hellman-group14-sha256,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,diffie-hellman-group-exchange-sha256'
      state: present

  - name: Adding ciphers line  {{ ansible_fqdn }}
    lineinfile:
      path: /etc/ssh/sshd_config
      regexp: '^Ciphers'
      line: 'Ciphers aes128-ctr,aes192-ctr,aes256-ctr,aes128-gcm@openssh.com,aes256-gcm@openssh.com'
      state: present

  - name: Adding MACs line  {{ ansible_fqdn }}
    lineinfile:
      path: /etc/ssh/sshd_config
      regexp: '^MACs'
      line: 'MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512,hmac-sha2-256,umac-128@openssh.com'
      state: present

  - name: Reloading sshd.service on {{ ansible_fqdn }}
    service:
      name: sshd
      state: reloaded
```

inventory.ini 

```bash
[local]
192.168.1.10
```

### Ansible Plabook Çalıştırılması

```bash
[root@akf euworks]# ansible-playbook sshd_hardening.yml -i inventory.ini -u admin -Kk
```

Hedef sunucuya  ssh parola ve sudo yetkili kullanıcı ile eriştiğim için -u -kK parametreleri ile çalıştırıyorum. Eğer root(sshkey) iseniz user ve ssh password parametrelerini kullanmanıza gerek yok.

> **\-i** oluşturulan inventory file gösterilir
> 
> **\-u** kullanıcı adı
> 
> **\-kK** sshpassword ve sudo password belirtme

### Playbook Çıktısı

```bash
[root@akf euworks]# ansible-playbook sshd_hardening.yml -i inventory.ini -u admin -Kk
SSH password: 
BECOME password[defaults to SSH password]: 

PLAY [Fix SSH Weak Key Exchange Algorithms, SSH Weak Key Exchange Algorithms Enabled, and MACs Enabled] ********************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************************************************************************************************
ok: [192.168.1.10]

TASK [Backup sshd_conf  tstlnx01] ***************************************************************************************************************************************************************************************************************************
changed: [192.168.1.10] => (item={u'dest': u'/etc/ssh/sshd_config2022-08-22_ansiblebackup', u'src': u'/etc/ssh/sshd_config'})

TASK [Adding KexAlgorithms line tstlnx01] *******************************************************************************************************************************************************************************************************************
changed: [192.168.1.10]

TASK [Adding ciphers line  tstlnx01] ************************************************************************************************************************************************************************************************************************
changed: [192.168.1.10]

TASK [Adding MACs line  tstlnx01] ***************************************************************************************************************************************************************************************************************************
changed: [192.168.1.10]

TASK [Reloading sshd.service on tstlnx01] *******************************************************************************************************************************************************************************************************************
changed: [192.168.1.10]

PLAY RECAP *****************************************************************************************************************************************************************************************************************************************************
192.168.1.10            : ok=6    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```

> Rhel 8 ve türevi sunucularda aşağıdaki değişikliğin yapılması gerekmektedir.
> 
>  vi /etc/sysconfig/sshd
> 
> uncomment yapılır. Aktif edildiğinde policy'I sshd\_config içerisinde kontrol eder.
> 
> CRYPTO\_POLICY = 

### Ek Bilgiler ve Değişikliklerin Kontrolü

sshd\_config dosyasındaki yapılan değişikliği servisi restart etmeden önce validate edilmesi.

```bash
sshd -t 
```

Varolan Config Dosyasının listelenmesi

```bash
root@akf # sshd -T | grep "\(ciphers\|macs\|kexalgorithms\)"
gssapikexalgorithms gss-group14-sha256-,gss-group16-sha512-,gss-nistp256-sha256-,gss-curve25519-sha256-,gss-group14-sha1-,gss-gex-sha1-
ciphers aes128-ctr,aes192-ctr,aes256-ctr,aes128-gcm@openssh.com,aes256-gcm@openssh.com
macs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512,hmac-sha2-256,umac-128@openssh.com
kexalgorithms diffie-hellman-group14-sha256,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,diffie-hellman-group-exchange-sha256
```

Yüklü  ssh servisinin desteklediklerini listeler

> Ciphers: ssh -Q cipher  
> MACs: ssh -Q mac  
> KexAlgorithms: ssh -Q kex  
> PubkeyAcceptedKeyTypes: ssh -Q key

Son olarak başka bir NMAP  yüklü sunucu ile hedef sunucudaki parametre değişikliklerinin kontrol edilmesi

```bash
[root@akf euworks]# nmap --script ssh2-enum-algos 192.168.1.10

Starting Nmap 6.40 ( http://nmap.org ) at 2022-08-22 00:07 +03
Nmap scan report for 192.168.1.10
Host is up (0.0018s latency).
Not shown: 997 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
| ssh2-enum-algos: 
|   kex_algorithms (4)
|       diffie-hellman-group14-sha256
|       diffie-hellman-group16-sha512
|       diffie-hellman-group18-sha512
|       diffie-hellman-group-exchange-sha256
|   server_host_key_algorithms (5)
|       rsa-sha2-512
|       rsa-sha2-256
|       ssh-rsa
|       ecdsa-sha2-nistp256
|       ssh-ed25519
|   encryption_algorithms (5)
|       aes128-ctr
|       aes192-ctr
|       aes256-ctr
|       aes128-gcm@openssh.com
|       aes256-gcm@openssh.com
|   mac_algorithms (6)
|       hmac-sha2-512-etm@openssh.com
|       hmac-sha2-256-etm@openssh.com
|       umac-128-etm@openssh.com
|       hmac-sha2-512
|       hmac-sha2-256
|       umac-128@openssh.com
|   compression_algorithms (2)
|       none
|_      zlib@openssh.com
```