# ДЗ №17 SELinux
### 1. Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.
К сдаче:
- README с описанием каждого решения (скриншоты и демонстрация приветствуются).

### 2. Обеспечить работоспособность приложения при включенном selinux.
- Развернуть приложенный стенд
https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems
- Выяснить причину неработоспособности механизма обновления зоны (см. README);
- Предложить решение (или решения) для данной проблемы;
- Выбрать одно из решений для реализации, предварительно обосновав выбор;
- Реализовать выбранное решение и продемонстрировать его работоспособность.
К сдаче:
- README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
- Исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.
--------------------------------------------------------------------------------------------
Прежде чем приступить к выполнению ДЗ установим ```semanage``` с помощью команды ```yum install policycoreutils-python -y```. Далее включим анализ логов с помощью команды - ```audit2why < /var/log/audit/audit.log```

### 1. Запустить nginx на нестандартном порту 3-мя разными способами.

#### 1.1 Переключатели setsebool
  
- Поскольку пакет nginx уже установлен с Vagrantfile-a предназначенного для данного ДЗ, то сразу зайдем в ```/etc/nginx/nginx.conf``` и поменяем стандартный 80-й порт на 11988. 
- ```[root@docker vagrant]# systemctl restart nginx```

```Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.```

```[root@docker vagrant]# journalctl -u nginx -n 30```

```-- Logs begin at Fri 2020-08-14 08:39:06 UTC, end at Fri 2020-08-14 09:15:49 UTC. --
Aug 14 09:03:22 docker systemd[1]: Unit nginx.service cannot be reloaded because it is inactive.
Aug 14 09:15:47 docker systemd[1]: Starting The nginx HTTP and reverse proxy server...
Aug 14 09:15:47 docker nginx[4171]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Aug 14 09:15:47 docker nginx[4171]: nginx: [emerg] bind() to 0.0.0.0:11988 failed (13: Permission denied)
Aug 14 09:15:47 docker nginx[4171]: nginx: configuration file /etc/nginx/nginx.conf test failed
Aug 14 09:15:47 docker systemd[1]: nginx.service: control process exited, code=exited status=1
Aug 14 09:15:47 docker systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Aug 14 09:15:47 docker systemd[1]: Unit nginx.service entered failed state.
Aug 14 09:15:47 docker systemd[1]: nginx.service failed.
```

- Как описано по выводам команд сверху, selinux не позваляет nginx поменять стандартный порт службы на 11988, для выяснения причины опять же воспользуемся ```audit2why``` (команда - ```audit2why < /var/log/audit/audit.log```). 
- Утилита audit2why рекомендует выполнить команду - ```setsebool -P nis_enabled 1```. После выполнения включения данного boolean по рекомендации ```audit2why```, nginx успешно запускается.
```
[root@docker vagrant]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1597396547.646:1226): avc:  denied  { name_bind } for  pid=4171 comm="nginx" src=11988 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
[root@docker vagrant]# setsebool -P nis_enabled 1
[root@docker vagrant]# systemctl restart nginx
[root@docker vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2020-08-14 09:28:01 UTC; 5s ago
```
```
[root@docker vagrant]# netstat -tunap
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      336/rpcbind         
tcp        0      0 0.0.0.0:11988           0.0.0.0:*               LISTEN      4211/nginx: master  
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      611/sshd            
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      701/master          
tcp        0      0 10.0.2.15:22            10.0.2.2:51250          ESTABLISHED 3957/sshd: vagrant  
tcp6       0      0 :::111                  :::*                    LISTEN      336/rpcbind         
tcp6       0      0 :::11988                :::*                    LISTEN      4211/nginx: master  
```

#### 1.2 Добавление нестандартного порта в имеющийся тип

- Для того чтобы пролистать все порты входящие в тип - http_port_t, достаточно выполнить команду - ```semanage port -l | grep http```. 
```
[root@docker vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
- Для добавления нестандартного порта внужный нам тип опять же воспользуемся утилитой ```semanage```.
```
[root@docker vagrant]# semanage port -a -t http_port_t -p tcp 11988
[root@docker vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      11988, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989

```

#### 1.3 Формирование и установка модуля SELinux.

- Опять же поменяем порт 80 на 11988 в ```/etc/nginx/nginx.conf```. 
- Далее скомпилируем модуль на основе лог файла - ```/var/log/audit/audit.log```, в котором содержится информация о запретах и установим созданный модуль.
```
[root@docker vagrant]# audit2allow -M httpd_new --debug < /var/log/audit/audit.log
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i httpd_new.pp
[root@docker vagrant]# ll
total 8
-rw-r--r--. 1 root root 964 Aug 14 10:46 httpd_new.pp
-rw-r--r--. 1 root root 261 Aug 14 10:46 httpd_new.te
[root@docker vagrant]# semodule -i httpd_new.pp
[root@docker vagrant]# semodule -l | grep http
httpd_new	1.0
[root@docker vagrant]# 
```
- Проверим работу nginx по настроенному порту 
```
[root@docker vagrant]# netstat -tunap
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      352/rpcbind         
tcp        0      0 0.0.0.0:11988           0.0.0.0:*               LISTEN      3950/nginx: master  
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      613/sshd            
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      701/master          
tcp        0      0 10.0.2.15:22            10.0.2.2:52344          ESTABLISHED 3731/sshd: vagrant  
tcp6       0      0 :::111                  :::*                    LISTEN      352/rpcbind         
tcp6       0      0 :::11988                :::*                    LISTEN      3950/nginx: master  
```

### 2. Обеспечить работоспособность приложения при включенном selinux.
- Скачиваем стенд из репозитория https://github.com/mbfx/otus-linux-adm, и поднимаем стенд с помощью команды ```vagrant up``` из под папки selinux_dns_problems/
- После подключения к клиентсой машине попробуем выполнить след. команду, предназначенную для обновления зоны и добавления в нее новой записи, после чего получим ошибку
```
[root@client vagrant]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> 
```
- Выполним команду ```audit2why < /var/log/audit/audit.log``` для того чтобы выяснить по какой именно причине selinux нам не дает обновить нашу зону и какой модль ему для этого необходим.
```
[root@ns01 vagrant]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1598303918.278:786): avc:  denied  { write } for  pid=2953 comm="isc-worker0000" path="/etc/named/dynamic/named.ddns.lab.view1.jnl" dev="sda1" ino=12185 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
``` 
```
[root@ns01 vagrant]# audit2allow -a 


#============= named_t ==============

#!!!! WARNING: 'etc_t' is a base type.
allow named_t etc_t:file create;
```
- Как видим нужно дать необходимые разрешения для named_t 

- Для этого воспользуемся утилитой audit2allow и выполним след команды, после сразу перезапустим нашу службу dns
```
[root@ns01 vagrant]# audit2allow -a -M named_t 
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i named_t.pp

[root@ns01 vagrant]# semodule -i named_t.pp
[root@ns01 vagrant]# systemctl restart named
```
- Вернемся на клиентскую ВМ и заново попробуем добавить хостовую запись на наш dns-сервер
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> 
```
- Как видим, никаких проблем на этот раз у нас не возникло, а это значит, что внесенные нами изменения дали нужный результат, чтобы окончательно в этом убедиться проверим состояние службы, где увидим что наша запись добавлена.
```
[root@ns01 vagrant]# systemctl status named
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2020-08-24 21:24:42 UTC; 1min 40s ago
  Process: 3004 ExecStop=/bin/sh -c /usr/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 3017 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 3015 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 3019 (named)
   CGroup: /system.slice/named.service
           └─3019 /usr/sbin/named -u named -c /etc/named.conf

Aug 24 21:24:42 ns01 named[3019]: network unreachable resolving './DNSKEY/IN': 2001:500:a8::e#53
Aug 24 21:24:42 ns01 named[3019]: network unreachable resolving './NS/IN': 2001:500:a8::e#53
Aug 24 21:24:42 ns01 named[3019]: network unreachable resolving './DNSKEY/IN': 2001:503:c27::2:30#53
Aug 24 21:24:42 ns01 named[3019]: network unreachable resolving './NS/IN': 2001:503:c27::2:30#53
Aug 24 21:24:52 ns01 named[3019]: managed-keys-zone/view1: Unable to fetch DNSKEY set '.': timed out
Aug 24 21:24:52 ns01 named[3019]: resolver priming query complete
Aug 24 21:24:52 ns01 named[3019]: managed-keys-zone/default: Unable to fetch DNSKEY set '.': timed out
Aug 24 21:24:52 ns01 named[3019]: resolver priming query complete
Aug 24 21:26:01 ns01 named[3019]: client @0x7f0b1803c3e0 192.168.50.15#48036/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
Aug 24 21:26:01 ns01 named[3019]: client @0x7f0b1803c3e0 192.168.50.15#48036/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': adding an RR at 'www.ddns.lab' A 192.168.50.15
```
