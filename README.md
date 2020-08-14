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

#####  1.1 Переключатели setsebool
  
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
Aug 14 09:15:47 docker systemd[1]: nginx.service failed.```

