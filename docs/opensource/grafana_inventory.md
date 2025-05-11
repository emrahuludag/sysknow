# RVTools Grafana Dashboard

RVTools scripti ile envanter export almıştık ancak Excel'de envanter tutmaktan yine kurtulamamıştık. Bu bölümde alınan csv exportu bir grafana kurup dashboard hazırlamaya değineceğiz.

### Grafana Kurulum
Docker veya podman kurulumuna değinmiyorum, ufak bir google araması ile ulaşabilirsiniz.

Data dizinimizi oluşturalım

```bash
mkdir -p /opt/grafana/data
```

Grafana container'imizi başlatalım
```bash
podman run -d \
  --name=grafana \
  -p 3000:3000 \
  -v /opt/grafana/data:/var/lib/grafana:z \
  -e "GF_SECURITY_ADMIN_USER=admin" \
  -e "GF_SECURITY_ADMIN_PASSWORD=P@ssW@ord" \
  docker.io/grafana/grafana-oss
```

İsterneniz systemd servisi haline getirebilirsiniz.

```bash
podman generate systemd --name grafana --files --restart-policy=always
cp grafana.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now grafana.service
```


Container'ımız çalışmaya başladı. 

```bash
CONTAINER ID  IMAGE                                 COMMAND     CREATED      STATUS        PORTS                   NAMES
cf3fc9758cdd  docker.io/grafana/grafana-oss:latest              7 hours ago  Up 2 seconds  0.0.0.0:3000->3000/tcp  grafana
```
### CSV Plugin Kurulum

http://IP:3000/ UI'u açalım ve öncelikle Plugins bölümünden CSV plugin'in kuralım.

![](./img/grafana1.png? ':size=80%')


### Data Source Eklenmesi
Kurmuş olduğumuz plugin'den bir data source oluşturuyoruz. 

![](./img/grafana2.png? ':size=80%')

RVTools  ile export aldığımız csv dosyasını local veya bir web-server üzerinden gösterebilirsiniz. Ortamınıza göre değerlendirebilirsiniz. Burada örnek olarak generate etmiş olduğum bir datayı server üzerinden gösteriyorum.

![](./img/grafana3.png? ':size=80%')

### Dashboard Oluşturulması

Dashboard > Add visualization

![](./img/grafana4.png? ':size=80%')

Oluşturduğumuz data source'u seçiyoruz.

![](./img/grafana5.png? ':size=80%')

Daha sonra Sağ panel den Visualization > Table seçiyoruz ve aşağıdaki gibi parametreleri değiştirip kaydediyoruz. Tercihinize göre düzenleyebilirsiniz.

![](./img/grafana6.png? ':size=80%')


Panel Options
Title: Inventory

Table
Show table header  > ON
Column filter > ON

Cell Options
Cell type > Colored text

Standart Options
Color scheme > Classic Palette


Save Dashboard ile yaptığımız değişiklikleri kayıt ediyoruz.

### Dashboard Yayınlanması

Share > Share externally ile isterseniz bir link oluşturuyoruz. Bu linki erişmenizi istediğiniz ekip arkadaşlarınız paylaşarak herkesin erişimine açabilirsiniz.

![](./img/grafana7.png? ':size=80%')

![](./img/grafana_final.png? ':size=80%')


### Sonuc

* Scriptler yardımı ile almış olduğumuz envanterimizi herkese farklı versiyonda Excel ile dağıtmak yerine tek bir noktadan eriştirebilir hale getirdik.
* Sunucularınızı vmware de taglıyorsanız bu dataları da rvtools ile alabiliyorsunuz. Ekip bazlı filtrelemek kolay hale geliyor.
* Scriptinizi ortamınızın network erişimlerine göre cron joblar ile otomatik hale getirebilirsiniz. Ya da rutin olarak haftalık aylık bunu yaparak güncelleyebilirsiniz.
* Fiziksel ortamlar çok sık değişmeyeceğini düşünerek statik olarak farklı bir csv hazırlayıp dashboard'a ekleyebilirsiniz.
* Bu bir CMDB değildir. Gerektiğinde hızlıca envantere ulaşabilmek hedeflenmiştir.


