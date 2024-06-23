# Ansible ile Linux Server Update Otomasyonu

![](./img/ansiblerhellogo.png)

Günümüz şartlarında tek tek yüzlerce sunucuyu manuel update etmek ciddi zaman gerektiren bir iş yükü. Tekrarlayan,  yorum gerektirmeyen işleri otomatize etmeli ve sürecin takibini yapmalıyız. Bugün ki yazımızda dağıtım bağımsız Redhat, CentOS, Oracle linux gibi sunucularda update yapabileceğimiz bir yml hazırlayacağız. Ek olarak güvenlik nedeniyle internete çıkışın açılamadığı ortamlar var ise local repository kullanarak ( Bkz:[Linux Sunucular İçin Repository Server Kurulumu](https://wiki.acikkaynakfikirler.com/tr/linux/Repository/Repository-Server-Kurulumu)) repo dağıtımı da yapacağız. İnternet erişimi sorununuz yok ise repository dağıtımını atlayabilirsiniz. 

# Yaml Dosyaların Hazırlanması ve Dizinlerin Oluşturulması

main.yml dosyası ana çalıştıracağımız playbook ve ilgili rollere ait yml dizin yapısı

```bash
[root@akf akf]# tree

├── inventory.ini
├── main.yml
└── roles
    ├── os-patching
    │   └── tasks
    │       ├── after_update.yml
    │       └── before_update.yml
    └── repo
        └── tasks
            ├── centos7.yml
            ├── repo_master.yml
```

# Update İçin Gerekli Yml Dosyaları

inventory.ini dosyasında update işlemi gerçekleştireceğimiz sunuculara ait ip bilgisi

```bash

[local]
192.168.1.2
192.168.1.3

[local:vars]
remote_tmp = /tmp/.ansible-${USER}/tmp
```

**main.yml**

Genel anlamda tüm işlemlerden burada bahsedeceğim. İlk adımda patch geçişi yapılacak işletim sistemleri olası uygulamaların update edilmesini engellemek için  repository dosyalarını "old" adında bir klasöre taşıyoruz ve local repomuza ait repository dosyalarını ekliyoruz.  Daha sonra update gerçekleştirmeden önce olası conf değişikliklerinde, /etc dizini ve sunucudan disklerin mount bilgisi gibi bazı bilgileri update sonrasında eksik olmaması için sunucu içerisine bir dosya export alıyoruz (`before_update.yml)`. 

Update işlemi gerçekleştirildikten sonra OS'in kontrolü için kontrolü için (after\_update.yml) export alıyoruz ve update edilen paketlerin listesini ekrana yazıyoruz.

```bash
---

- name: OS Patching Start
  become: yes
  hosts: local
  tasks:
   - name: Adding Local Repository File role
     include_role:
       name: repo
       tasks_from: repo_master.yml
       
#Update once etc dizini, route,lsblk ve fstab export edilmesi
   - name: Backup etc and os information role
     include_role:
       name: os-patching
       tasks_from: before_update.yml

#Update baslatilması
   - name: Update Latest Packages role
     yum:
      name: '*'
      state: latest
      update_cache: yes
      update_only: yes
     register: reboot_required
     when: ansible_facts['distribution'] == "CentOS" or ansible_facts['distribution'] == "RedHat" or ansible_facts['distribution'] == "OracleLinux"

#Update basarili ise reboot gerceklestirilmesi ve kontrolu
   - when: reboot_required.changed
     block:
        - name: reboot the server if required
          shell: sleep 3; reboot
          ignore_errors: true
          async: 1
          poll: 0
        - name: wait for server to come back after reboot
          wait_for_connection:
            timeout: 600
            delay: 20
          register: reboot_result
        - name: reboot time
          debug:
            msg: "The system rebooted in {{ reboot_result.elapsed }} seconds."
        - name: Export os information for check role
          include_role:
           name: os-patching
           tasks_from: after_update.yml
 
 #Update edilen paketleri listelenmesi   
   - name: List updated packages for Rhel Based Servers
     shell: rpm -qa --last | grep "$(date +%a\ %d\ %b\ %Y)" |cut -f 1 -d " "
     register: result
     args:
      warn: no
   - name: Updates packages
     debug: msg="{{ result.stdout_lines }}"
```

**before\_update.yml**

```bash
---
#backup icin dizin olusturulması
    - name: Create Backup directory 
      file:
        path: /akf/backup/  
        state: directory  
        mode: "0777"
#etc dizinin backup alınması        
    - name: Backup etc Folder
      archive:
        path: /etc
        dest: "/akf/backup/etc-backup-{{ansible_date_time.date}}-{{ansible_date_time.time}}.tar.gz"
        format: gz
        force_archive: true
#update oncesi karsilastırma yapmak icin bilgilerin alınması
    - name: Export fstab,release,route,lsblk before update
      shell: |
        date=$(date +%d-%m-%Y)
        echo "######################__RELEASE__##################################" >> /akf/before_update_$HOSTNAME"_"$date.txt
        cat /etc/*-release >> /akf/before_update_$HOSTNAME"_"$date.txt
        echo "######################__IFCONFIG__##################################" >> /akf/before_update_$HOSTNAME"_"$date.txt
        ifconfig -a >> /akf/before_update_$HOSTNAME"_"$date.txt
        echo "######################__NETSTAT__##################################" >> /akf/before_update_$HOSTNAME"_"$date.txt
        netstat -rn >> /akf/before_update_$HOSTNAME"_"$date.txt
        echo "###################__DF__##########################################" >> /akf/before_update_$HOSTNAME"_"$date.txt
        df -hT >> /akf/before_update_$HOSTNAME"_"$date.txt
        echo "####################__LSBLK__######################################" >> /akf/before_update_$HOSTNAME"_"$date.txt
        lsblk >> /akf/before_update_$HOSTNAME"_"$date.txt
        echo "###################__FSTAB__########################################" >> /akf/before_update_$HOSTNAME"_"$date.txt
        cat /etc/fstab >> /akf/before_update_$HOSTNAME"_"$date.txt
```

**after\_update.yml**

```bash
---
   - name: Export fstab,release,route,lsblk after update
     shell: |
       date=$(date +%d-%m-%Y)
        echo "######################__RELEASE__##################################" >> /akf/after_update_$HOSTNAME"_"$date.txt
        cat /etc/*-release >> /akf/after_update_$HOSTNAME"_"$date.txt
        echo "######################__IFCONFIG__##################################" >> /akf/after_update_$HOSTNAME"_"$date.txt
        ifconfig -a >> /akf/after_update_$HOSTNAME"_"$date.txt
        echo "######################__NETSTAT__##################################" >> /akf/after_update_$HOSTNAME"_"$date.txt
        netstat -rn >> /akf/after_update_$HOSTNAME"_"$date.txt
        echo "###################__DF__##########################################" >> /akf/after_update_$HOSTNAME"_"$date.txt
        df -hT >> /akf/after_update_$HOSTNAME"_"$date.txt
        echo "####################__LSBLK__######################################" >> /akf/after_update_$HOSTNAME"_"$date.txt
        lsblk >> /akf/after_update_$HOSTNAME"_"$date.txt
        echo "###################__FSTAB__########################################" >> /akf/after_update_$HOSTNAME"_"$date.txt
        cat /etc/fstab >> /akf/after_update_$HOSTNAME"_"$date.txt
```

**repo\_master.yml**

Burada OS release bilgisine göre repository dosyalarını atmak çok önemli. Oracle bir sunucuya Centos'a ait bir depo eklemek ciddi sorun oluşturabilir. İlk defa yapıyorsanız mutlaka kontrol etmenizi tavsiye ederim. 

```bash
---

   - name: Adding Local Repository file on Redhat 7 Based Servers
     include_role:
       name: repo
       tasks_from: rhel7.yml
     when: ansible_facts['distribution_release'] == "Maipo"
   
   - name: Adding Local Repository file on Redhat 8  Servers
     include_role:
       name: repo
       tasks_from: rhel8.yml
     when: ansible_facts['distribution_release'] == "Ootpa"

   - name: Adding Local Repository file on CentOS 7  Servers
     include_role:
       name: repo
       tasks_from: centos7.yml
     when: ansible_facts['distribution'] == "CentOS" and ansible_facts['distribution_major_version'] == "7"
 
   - name: Adding Local Repository file on CentOS 8  Servers
     include_role:
       name: repo
       tasks_from: centos8.yml
     when: ansible_facts['distribution'] == "CentOS" and ansible_facts['distribution_major_version'] == "8"

   - name: Adding Local Repository file on Oracle Linux 7  Servers
     include_role:
       name: repo
       tasks_from: oel7.yml
     when: ansible_facts['distribution'] == "OracleLinux" and ansible_facts['distribution_major_version'] == "7"
  
   - name: Adding Local Repository file on Oracle Linux 8  Servers
     include_role:
       name: repo
       tasks_from: oel8.yml
     when: ansible_facts['distribution'] == "OracleLinux" and ansible_facts['distribution_major_version'] == "8"

   - name: Adding Local Repository file on Oracle Linux 8  Servers
     include_role:
       name: repo
       tasks_from: oel8.yml
     when: ansible_facts['distribution'] == "OracleLinux" and ansible_facts['distribution_major_version'] == "8"
```

**centos7.yml** 

Örnek olması açısından sadece CentOS için hazırladım. Sisteminizde yönetmiş olduğunuz Redhat 7-8 OEL7-8 veya Ubuntu için ayrı ayrı örnekte vermiş olduğum baz alarak yml oluşturabilirsiniz.

```bash
---
#Daha önceden eklenen repoların boşaltılması
  - name: Empty /etc/yum.repos.d first
    shell: |
      mkdir -p /etc/yum.repos.d/old
      touch /etc/yum.repos.d/temp.repo
      mv -f /etc/yum.repos.d/*.repo /etc/yum.repos.d/old
#Reponun eklenmesi
  - name: Create CentOS 7 Local Repo
    copy:
      dest: "/etc/yum.repos.d/akf7centoslocal.repo"
      content: |
        [base]
        name=AKF-CentOS-$releasever - Base
        baseurl=http://repo.akf.local/7_Centos/base
        gpgcheck=1
        gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
        enabled=1

        #released updates
        [updates]
        name=AKF-CentOS-$releasever - Updates
        baseurl=http://repo.akf.local/7_Centos/updates
        gpgcheck=1
        gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
        enabled=1

        #additional packages that may be useful
        [extras]
        name=AKF-CentOS-$releasever - Extras
        baseurl=http://repo.akf.local/7_Centos/extras
        gpgcheck=1
        gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
        enabled=1
```

# Playbook'un Çalıştırılması ve Örnek İşlem Çıktısı

Hedef sunucuya ssh password ve sudo password için -kK parametreleri kullanıyorum. Direk sshkey ile root bağlandıysanız bu parametre gerekli değil.

> Update işlemi yapıldıktan geri dönüşü olmayacağı için mutlaka backup almalı ve önce test ortamlarında uygulamanıza ederim.

```bash
[root@akf akf]# ansible-playbook main.yml -i inventory.ini -u akf-kK
```

Örnek Çıktı

```bash
PLAY [OS Patching Start] *******************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************************************************************************************************************
ok: [192.168.1.2]

TASK [Update Latest Packages] **************************************************************************************************************************************************************************************************************
changed: [192.168.1.2]

TASK [reboot the server if required] *******************************************************************************************************************************************************************************************************
changed: [192.168.1.2]

TASK [wait for server to come back after reboot] *******************************************************************************************************************************************************************************************
ok: [192.168.1.2]


TASK [reboot time] *************************************************************************************************************************************************************************************************************************
ok: [192.168.1.2] => {
    "msg": "The system rebooted in 126 seconds."
}

TASK [List updated packages for Rhel Based Servers] ****************************************************************************************************************************************************************************************
changed: [192.168.1.2]

TASK [Updates packages] ********************************************************************************************************************************************************************************************************************
ok: [192.168.1.2] => {
    "msg": [
        "kernel-3.10.0-1160.76.1.el7.x86_64", 
        "scap-security-guide-0.1.63-1.el7_9.noarch", 
        "kernel-devel-3.10.0-1160.76.1.el7.x86_64", 
        "libpcap-1.5.3-13.el7_9.x86_64", 
        "gzip-1.5-11.el7_9.x86_64", 
        "bind-export-libs-9.11.4-26.P2.el7_9.10.x86_64", 
        "xz-5.2.2-2.el7_9.x86_64", 
        "rsyslog-8.24.0-57.el7_9.3.x86_64", 
        "rsync-3.1.2-11.el7_9.x86_64", 
        "open-vm-tools-11.0.5-3.el7_9.4.x86_64", 
        "openssl-1.0.2k-25.el7_9.x86_64", 
        "libgudev1-219-78.el7_9.7.x86_64", 
        "microcode_ctl-2.1-73.14.el7_9.x86_64", 
        "nss-tools-3.79.0-4.el7_9.x86_64", 
        "gtk3-3.22.30-8.el7_9.x86_64", 
        "grub2-2.02-0.87.el7_9.9.x86_64", 
        "bind-utils-9.11.4-26.P2.el7_9.10.x86_64", 
        "systemd-sysv-219-78.el7_9.7.x86_64", 
        "samba-4.10.16-20.el7_9.x86_64", 
        "kernel-tools-3.10.0-1160.76.1.el7.x86_64", 
        "tuned-2.11.0-12.el7_9.noarch", 
        "subscription-manager-1.24.51-1.el7_9.x86_64", 
        "java-1.8.0-openjdk-1.8.0.345.b01-1.el7_9.x86_64", 
        "glibc-headers-2.17-326.el7_9.x86_64", 
        "glibc-devel-2.17-326.el7_9.x86_64", 
        "kernel-headers-3.10.0-1160.76.1.el7.x86_64", 
        "java-1.8.0-openjdk-headless-1.8.0.345.b01-1.el7_9.x86_64", 
        "tzdata-java-2022d-1.el7.noarch", 
        "subscription-manager-rhsm-certificates-1.24.51-1.el7_9.x86_64", 
        "subscription-manager-rhsm-1.24.51-1.el7_9.x86_64", 
        "samba-libs-4.10.16-20.el7_9.x86_64", 
        "samba-common-tools-4.10.16-20.el7_9.x86_64", 
        "python-syspurpose-1.24.51-1.el7_9.x86_64", 
        "python-perf-3.10.0-1160.76.1.el7.x86_64", 
        "python-libs-2.7.5-92.el7_9.x86_64", 
        "python-2.7.5-92.el7_9.x86_64", 
        "kernel-tools-libs-3.10.0-1160.76.1.el7.x86_64", 
        "gtk-update-icon-cache-3.22.30-8.el7_9.x86_64", 
        "grub2-tools-extra-2.02-0.87.el7_9.9.x86_64", 
        "grub2-pc-2.02-0.87.el7_9.9.x86_64", 
        "expat-2.1.0-15.el7_9.x86_64", 
        "bind-libs-9.11.4-26.P2.el7_9.10.x86_64", 
        "nss-sysinit-3.79.0-4.el7_9.x86_64", 
        "nss-softokn-3.79.0-4.el7_9.x86_64", 
        "nss-3.79.0-4.el7_9.x86_64", 
        "grub2-tools-minimal-2.02-0.87.el7_9.9.x86_64", 
        "grub2-tools-2.02-0.87.el7_9.9.x86_64", 
        "bind-libs-lite-9.11.4-26.P2.el7_9.10.x86_64", 
        "samba-client-libs-4.10.16-20.el7_9.x86_64", 
        "samba-common-libs-4.10.16-20.el7_9.x86_64", 
        "samba-common-4.10.16-20.el7_9.noarch", 
        "libwbclient-4.10.16-20.el7_9.x86_64", 
        "systemd-219-78.el7_9.7.x86_64", 
        "xz-libs-5.2.2-2.el7_9.x86_64", 
        "systemd-libs-219-78.el7_9.7.x86_64", 
        "zlib-1.2.7-20.el7_9.x86_64", 
        "openssl-libs-1.0.2k-25.el7_9.x86_64", 
        "nss-util-3.79.0-1.el7_9.x86_64", 
        "nspr-4.34.0-3.1.el7_9.x86_64", 
        "krb5-libs-1.15.1-54.el7_9.x86_64", 
        "glibc-2.17-326.el7_9.x86_64", 
        "glibc-common-2.17-326.el7_9.x86_64", 
        "nss-softokn-freebl-3.79.0-4.el7_9.x86_64", 
        "tzdata-2022d-1.el7.noarch", 
        "grub2-pc-modules-2.02-0.87.el7_9.9.noarch", 
        "bind-license-9.11.4-26.P2.el7_9.10.noarch", 
        "ca-certificates-2022.2.54-74.el7_9.noarch", 
        "grub2-common-2.02-0.87.el7_9.9.noarch"
    ]
}



PLAY RECAP *********************************************************************************************************************************************************************************************************************************
192.168.1.2             : ok=7    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

> Debian Base Sunucular için
> 
> ```bash
>    - name:  Upgrade Ubuntu
>      apt:
>       update_cache: yes
>       force_apt_get: yes
>       autoremove: yes
>       autoclean: yes
>       cache_valid_time: 3600
>       upgrade: yes
>      register: reboot_required
>      when: ansible_facts['distribution'] == "Ubuntu"
> 
>    - when: reboot_required.changed
>      block:
>         - name: reboot the server if required
>           shell: sleep 3; reboot
>           ignore_errors: true
>           async: 1
>           poll: 0
>         - name: wait for server to come back after reboot
>           wait_for_connection:
>             timeout: 600
>             delay: 20
>           register: reboot_result
>         - name: reboot time
>           debug:
>             msg: "The system rebooted in {{ reboot_result.elapsed }} seconds."
>         - name: Export os information for check role
>           include_role:
>            name: os-patching
>            tasks_from: after_update.yml
> ```
> 
> ```bash
>    - name: List updated packages for Ubuntu Servers
>      shell: grep -E "^$(date +%Y-%m-%d).+ (install|upgrade) " /var/log/dpkg.log |cut -d " " -f 3-5
>      register: result_ubuntu
>      args:
>       warn: no
>    - name: Show Ubuntu Output
>      debug: msg="{{ result_ubuntu.stdout_lines }}"
> ```

> ubuntu2004.yml
> 
> ```bash
> ---
>   - name: Empty etc/apt/sources.list.d/ first
>     shell: |
>       mkdir -p /etc/apt/sources.list.d/old
>       touch /etc/apt/sources.list.d/test.list
>       mv -f /etc/apt/sources.list.d/*.list /etc/apt/sources.list.d/old
>       mv -f /etc/apt/sources.list /etc/apt/sources.list.old
>   - name: Create Ubuntu 20.04 Local Repo
>     copy:
>       dest: "/etc/apt/sources.list"
>       content: |
>         #################################UBUNTU 20.04 LTS #######################################
>         deb http://akf.local/ubuntu focal main restricted universe multiverse
>         deb http://akf.local/ubuntu focal-security main restricted universe multiverse
>         deb http://akf.local/ubuntu focal-updates main restricted universe multiverse
> ```