# Ansible Lineinfile Modülü Nedir?

![](./img/ansiblerhellogo.png)


Ansible lineinfile modülü, bir dosya üzerinde veya bir servise (apache,rsyslog vb) ait konfigürasyon dosyasında, tek bir satırda satır değiştirme, satırı güncelleme veya belirli bir satır ekleme gibi çeşitli eylemleri gerçekleştiren bir modüldür.

Bu modül sayesinde envanterimizdeki yüzlerce sunuculara tek tek bağlanmadan  hızlıca parametre değişikliklerini ve eklemelerini gerçekleştirebiliriz. Zamandan da ciddi anlamda tasarruf sağlıyoruz.

Başlıca neler yapılabilir?

1.  Bir kullanıcıyı sudo yetkisi verilmesi için sudoers dosyasına eklenebilir.
2.  Apache configurasyon dosyasında port değişikliği yapılabilir
3.  Host dosyasından tanım eklenebilir çıkartılabilir.
4.  Belirli satır öncesine veya sonrasına ekleme yapılabilir
5.  #'lenen bir parametre “#” kaldırılarak aktif hale getirilebilir.

> Ansible Lineinfile, yalnızca eşleşen veya bulunan son satırı değiştirir.  Dosyada birden fazla eşleşme varsa veya birden fazla satırda işlem yapılacak ise replace modülünü kullanmalısınız.
> 
> Replace Modülü için [bknz](https://docs.ansible.com/ansible/2.9/modules/replace_module.html)

# Parametreler

**backup** : Değişikliklik yapılacak olan dosyanın bulunduğun dizinde tarih ve zaman damgası ile dosyanın backup'ını alır. 

**regexp** : Değiştirilecek veya eklenecek olan kelimeyi dosya içerisindeki tüm satırlarda arar.

**state** : present/absent  state, aranılan kelime bulunduğunda değiştirir bulunamazsa ekler.  Absent ise kaldırır.

**create:** dosya yok ise oluşturur.

Daha fazla parametreler için [bknz](https://docs.ansible.com/ansible/2.9/modules/lineinfile_module.html)

Yapacağımız örneğimizde rsyslog servisine ait konfigürasyon dosyasında ikinci bir log sunucusu tanımı yapacağız ve sonrasında ilgili servisi restart edeceğiz.

Öncelikle işlem yapacağımız sunucuları inventory file oluşturup içerisine sunucu tanımlarını yapalım.

```plaintext
#inventory.ini
[local]
IP1
IP2
IP3
IP4
```

Ansible-playbook yaml dosyasını oluşturalım

```plaintext
# rsyslog_change.yml
---
- name: Rsyslog Sunucu Tanımı Ekleme 
  become: yes
  hosts: local
  tasks:
    - name: Rsyslog Conf Dosyasına Tanım Ekleme
      lineinfile:
        path: /etc/rsyslog.conf
        state: present
        line: '*.* @@IP:514'
        backup: yes

    - name: Rsyslog Servisi Restart Ediliyor
      service:
        name: rsyslog.service
        state: restarted
```

SSH parolası ile erişim sağladığım için ve sudo yetkili kullanıcı kullandığım için  aşağıdaki gibi çalıştırıyorum. Eğer root iseniz  user ve ssh password parametrelerini kullanmanıza gerek yok.

**\-i** oluşturulan inventory file gösterilir

**\-u** kullanıcı adı

**\-kK** sshpassword ve sudo password belirtme

```plaintext
#ansible-playbook rsyslog_change.yml -i inventory.ini -u username -kK
```

**İşlem Çıktısı**

```plaintext
# ansible-playbook rsyslog_change.yml -i inventory.ini -u username -kK
SSH password: 
SUDO password[defaults to SSH password]: 

PLAY [ Rsyslog Sunucu Tanımı Ekleme ] ****************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************

ok: [IP1]

TASK [ Rsyslog Conf Dosyasına Tanım Ekleme] **************************************************************************************************************************************************************
ok: [IP1]


TASK [Rsyslog Servisi Restart Ediliyor] *******************************************************************************************************************************************************
changed: [IP1]

PLAY RECAP ***********************************************************************************************************************************************************************************
IP1             : ok=1    changed=1    unreachable=0    failed=0 
```

Eğer değişiklik yapmak isteseydim yaml dosyamızdaki tanım alanı aşağıdaki gibi olacaktı.

```plaintext
 - name: Rsyslog Conf Dosyasına Tanım Ekleme
      lineinfile:
        path: /etc/rsyslog.conf
        regex: '*.* @@IP:514'
        line: '*.* @@IP2:514'
        backup: yes
        state: present
```

  Yine [bu](https://docs.ansible.com/ansible/2.9/modules/lineinfile_module.html) sayfadaki örneklerden faydalanarak yapacağınıza işe göre yaml hazırlayabilirsiniz.