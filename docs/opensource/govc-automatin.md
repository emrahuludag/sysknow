# VMware vSphere'de Govc ile Bulk VM Deployment

Büyük ölçekli ortamlarda onlarca sanal sunucuları tek tek oluşturmak ciddi zaman kayıplarına yol açıyor. Sektörde genellikle sanal sunucu kurulumlarını Ansible ve Terrafom gibi çeşitli opensource çözümler kullanılıyor. Ancak bazen yeni projelerde süreçler tam olarak hazırlanmamış olabiliyor. Örneğin ITSM entegre otomasyon sisteminiz var ise henüz kuracağınız ortamın entegrasyonu yapılmamış veya network erişimleri henüz tamamlanmış olabiliyor. Elinizde bulk olarak oluşturulması gereken bir sunucu listesi var ise kurulum günlerce sürebiliyor. Bulk olarak ortalama 2dk gibi bir sürede vm oluşturabilmek için vmware tarafından opensource olarak geliştirilen govc cli tool'u ile bir script hazırladım.  Bu script ile onlarca vm'i çok kısa sürede otomatik olarak template'den oluşturup hostname,ip,disk gibi disk atamalarını yapabiliriz. 

# >[!TIP|style:flat]
> govc, VMware tarafından geliştirilen açık kaynak bir CLI aracıdır.govc, VMware tarafından geliştirilen açık kaynak bir CLI aracıdır. VMware'in Go dilinde yazılmış resmi vSphere API istemcisi olan govmomi kütüphanesinin bir parçasıdır VMware'in Broadcom tarafından satın alınmasının ardından proje Broadcom bünyesinde geliştirilmeye devam etmektedir. . GitHub: https://github.com/vmware/govmomi 
> VMware ortamınıza erişen linux bir sunucuda veya Windows WSL'de kullanabiliriz.

# Files
```bash
govc-deploy-automation/
├── govc-vm-deploy.sh        # Main entry point (interactive menu)
├── bin/
│   ├── deploy_linux.sh      # Linux VM deployment logic
│   └── deploy_windows.sh    # Windows VM deployment logic
└── vms/
    ├── linuxvm.csv          # Linux VM inventory
    └── msvm.csv             # Windows VM inventory
```



# Örnek Linux CSV
```bash
folder;vmname;hostname;osdisk;disk1;disk2;disk3;cpu;memory;vlan;ip;netmask;gw;dns1;dns2;domain;vmvenv;vmteam
Unix;tr-lb01;tr-lb01;100;150;0;0;4;16;VLAN_703;10.20.10.1;255.255.255.0;10.20.10.254;10.10.11.1;10.10.11.2;domain.local;prod;unix
Unix;tr-lb02;tr-lb02;100;150;0;0;4;16;VLAN_703;10.20.10.2;255.255.255.0;10.20.10.254;10.10.11.1;10.10.11.2;domain.local;prod;unix
```

# Örnek Windows CSV
```bash
folder;vmname;hostname;osdisk;disk1;disk2;disk3;cpu;memory;vlan;ip;netmask;gw;dns1;dns2;domain;vmenv;vmteam
Microsoft;tr-db01;tr-db01;150;0;0;0;4;32;VLAN_703;10.10.10.1;255.255.255.0;10.10.10.254;10.10.0.1;10.10.0.12;domain.local;prod;microsoft
Microsoft;tr-db02;tr-db02;150;0;0;0;4;16;VLAN_704;10.10.10.2;255.255.255.1;10.10.10.255;10.10.0.1;10.10.0.13;domain.local;prod;microsoft
Microsoft;tr-db03;tr-db03;150;0;0;0;4;16;VLAN_705;10.10.10.3;255.255.255.2;10.10.10.256;10.10.0.1;10.10.0.14;domain.local;prod;microsoft

```

# >[!TIP|style:flat]
> GOVC_INSECURE=1 varsayılan olarak ayarlanmıştır — self-signed vCenter sertifikaları kabul edilir.
> Script set -euo pipefail kullanır; herhangi bir govc hatasında işlem anında durur.
> Her VM için IP bekleme zaman aşımı 5 dakikadır (govc vm.ip -wait 5m).
> Linux için, template üzerinde vmware-toolsd servisinin çalışıyor olması gerekir; böylece customization işlemi doğru şekilde uygulanır.
> Windows için template üzerinde VMware Tools ve geçerli bir Sysprep yapılandırması bulunmalıdır.
> Windows VM’ler yerel bir administrator şifresi tanımlanmadan açılır. İlk yapılandırma (domain join, şifre atama vb.) vCenter üzerinden VM konsoluna bağlanılarak yapılmalıdır.


# Kullanım
```bash
git clone https://github.com/emrahuludag/govc-deploy-automation.git

==============================================================================================================
 ░▀█▀░█▀█░█▀▀░█▀▄░█▀█░░░█▀▀░█░░░█▀█░█░█░█▀▄░░░█▀█░█░░░█▀█░▀█▀░█▀▀░█▀█░█▀▄░█▄█░█▀▀░░░▀█▀░█▀▀░█▀█░█▄█
 ░░█░░█░█░█▀▀░█▀▄░█▀█░░░█░░░█░░░█░█░█░█░█░█░░░█▀▀░█░░░█▀█░░█░░█▀▀░█░█░█▀▄░█░█░▀▀█░░░░█░░█▀▀░█▀█░█░█
 ░▀▀▀░▀░▀░▀░░░▀░▀░▀░▀░░░▀▀▀░▀▀▀░▀▀▀░▀▀▀░▀▀░░░░▀░░░▀▀▀░▀░▀░░▀░░▀░░░▀▀▀░▀░▀░▀░▀░▀▀▀░░░░▀░░▀▀▀░▀░▀░▀░▀
==============================================================================================================
==============================================================================================================
                                SSD Team - GOVC vSphere Automation
                             VMware | Linux & Microsoft Provisioning
==============================================================================================================
[1].Deploy Linux
[2].Deploy Windows
[3].Install govc on Linux
--------------------------------------------------------------------------------------------------------------
[q].Quit
Please Select Tool for RUN: 1
░ Deploy Linux from ./vms/linuxvms.csv
Deployment will start in 5 seconds. Press Ctrl+C to cancel...
Sun Jun 14 16:04:45 +03 2026
Please enter vCenter connection details:
vCenter_URL (e.g. https://vcenter.example.local):https://tr.example.com
vCenter_USERNAME (e.g. deploy@vsphere.local): deploy@vsphere.local
vCenter_PASSWORD: 
vCenter_DATACENTER (e.g. TR_DC): TR_Datacenter
vCenter_CLUSTER (e.g. TR_CLS): TR_Cluster
vCenter_DATASTORE (e.g. TR-DS01): TR-DS01
vCenter Template Name (e.g. RH9_tmp or UBNT2204_tmp): RH9_tmp
Testing vCenter connection...
●:OK: Connection successful!
==========================================
Creating folder: Unix
Started: 2026-06-14 16:05:57
==========================================
Folder already exist!
==========================================
Cloning VM: tr-lb01 from template: RH9_tmp
==========================================
[14-06-26 16:06:20] Cloning /TR_Datacenter/vm/Templates/RH9_tmp to tr-lb01...OK
Adding Network to VM: tr-lb01 VLAN_70
Setting CPU and Memory for tr-lb01
Customizing VM tr-lb01 (hostname, IP, DNS)
Adding Annotations
Powering on VM: tr-lb01
Powering on VirtualMachine:vm-87... OK
Waiting for IP address...
10.20.10.1
IP assigned in 70 seconds
----------------------------------------
Sun Jun 14 16:08:06 +03 2026

==========================================
tr-lb01 DEPLOYMENT SUMMARY
==========================================
Folder         : Unix
VM Name        : tr-lb01
Hostname       : tr-lb01
IP Address     : 10.20.10.1
Netmask        : 255.255.255.0
Gateway        : 10.20.10.254
Dns1, Dns2     : 10.10.11.1 10.10.11.2
Vlan           : VLAN_70
Cpu            : 4
Memory         : 16
OS Disk        : 150
Disk 2         : 0
Disk 3         : 0
Environment    : prod
Team           : unix
----------------------------------------
Task Complete  : 2m 5s
==========================================
==========================================
Creating folder: Unix
Started: 2026-06-14 16:08:06
==========================================
Folder already exist!
==========================================
Cloning VM: tr-lb02 from template: RH9_tmp
==========================================
[14-06-26 16:08:27] Cloning /TR_Datacenter/vm/Templates/RH9_tmp to tr-lb02...OK
Adding Network to VM: tr-lb02 VLAN_70
Setting CPU and Memory for tr-lb02
Customizing VM tr-lb02 (hostname, IP, DNS)
Adding Annotations
Powering on VM: tr-lb02
Powering on VirtualMachine:vm-88... OK
Waiting for IP address...
10.20.10.2
IP assigned in 76 seconds
----------------------------------------
Sun Jun 14 16:10:21 +03 2026

==========================================
tr-lb02 DEPLOYMENT SUMMARY
==========================================
Folder         : Unix
VM Name        : tr-lb02
Hostname       : tr-lb02
IP Address     : 10.20.10.2
Netmask        : 255.255.255.0
Gateway        : 10.20.10.254
Dns1, Dns2     : 10.10.11.1 10.10.11.2
Vlan           : VLAN_70
Cpu            : 4
Memory         : 16
OS Disk        : 150
Disk 2         : 0
Disk 3         : 0
Environment    : prod
Team           : unix
----------------------------------------
Task Complete  : 2m 12s
==========================================
==========================================
Creating folder: Unix
Started: 2026-06-14 16:10:21
==========================================
Folder already exist!
==========================================
Cloning VM: tr-lb03 from template: RH9_tmp
==========================================
[14-06-26 16:10:43] Cloning /TR_Datacenter/vm/Templates/RH9_tmp to tr-lb03...OK
Adding Network to VM: tr-lb03 VLAN_70
Setting CPU and Memory for tr-lb03
Customizing VM tr-lb03 (hostname, IP, DNS)
Adding Annotations
Powering on VM: tr-lb03
Powering on VirtualMachine:vm-89... OK
Waiting for IP address...
10.20.10.3
IP assigned in 67 seconds
----------------------------------------
Sun Jun 14 16:12:26 +03 2026

==========================================
tr-lb03 DEPLOYMENT SUMMARY
==========================================
Folder         : Unix
VM Name        : tr-lb03
Hostname       : tr-lb03
IP Address     : 10.20.10.3
Netmask        : 255.255.255.0
Gateway        : 10.20.10.254
Dns1, Dns2     : 10.10.11.1 10.10.11.2
Vlan           : VLAN_70
Cpu            : 4
Memory         : 16
OS Disk        : 150
Disk 2         : 0
Disk 3         : 0
Environment    : prod
Team           : unix
----------------------------------------
Task Complete  : 2m 2s
==========================================

```

Faydalı olması dileğiyle.