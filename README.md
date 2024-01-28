```
1. Запустить nginx на нестандартном порту 3-мя разными способами:

- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.
    К сдаче:
README с описанием каждого решения (скриншоты и демонстрация приветствуются).

2. Обеспечить работоспособность приложения при включенном selinux.
- развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
- выяснить причину неработоспособности механизма обновления зоны (см. README);
- предложить решение (или решения) для данной проблемы;
- выбрать одно из решений для реализации, предварительно обосновав выбор;
- реализовать выбранное решение и продемонстрировать его работоспособность.
```
1. Запустить nginx на нестандартном порту 3-мя разными способами. При разворачивании стенда сервис nginx отработал с ошибками
```
   selinux: ● nginx.service - The nginx HTTP and reverse proxy server
    selinux:    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    selinux:    Active: failed (Result: exit-code) since Sun 2024-01-28 17:24:14 UTC; 30ms ago
```
Проверка что файрволл отключен
```
[vagrant@selinux ~]$ sudo -i
[root@selinux ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```
Проверка конфигурации nginx
```
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Проверка режима работы SELinux
```
[root@selinux ~]# getenforce
Enforcing
```
Устанавливаем утилиты, а именно audit2why. Находим причину блокировки порта
```
[root@selinux ~]# grep denied /var/log/audit/audit.log
type=AVC msg=audit(1706462654.704:809): avc:  denied  { name_bind } for  pid=2861 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

[root@selinux ~]# yum install policycoreutils-python

[root@selinux ~]# grep 1706462654.704:809 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1706462654.704:809): avc:  denied  { name_bind } for  pid=2861 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```
```
Итак ответ для решения проблемы
 	Allow access by executing:
	# setsebool -P nis_enabled 1
```
```
setsebool -P nis_enabled 1
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2024-01-28 19:26:32 UTC; 9s ago
  Process: 21888 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21885 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21884 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21890 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21890 nginx: master process /usr/sbin/nginx
           └─21892 nginx: worker process

Jan 28 19:26:32 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 28 19:26:32 selinux nginx[21885]: nginx: the configuration file /etc/nginx/nginx.conf synt...s ok
Jan 28 19:26:32 selinux nginx[21885]: nginx: configuration file /etc/nginx/nginx.conf test is ...sful
Jan 28 19:26:32 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
Hint: Some lines were ellipsized, use -l to show in full.
```
Проверка доступности сервиса с хоста
```
user@user-VirtualBox:~/SE$ curl http://127.0.0.1:4881
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
```
Проверка параметра nis_enabled
```
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
```
Отключаем параметр, чтобы разрешить в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип
```
[root@selinux ~]# setsebool -P nis_enabled  off
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> off

[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989

[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881

[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Перезапуск и проверка запуска службы nginx
```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2024-01-28 19:37:17 UTC; 9s ago
  Process: 21933 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21930 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21929 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21935 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21935 nginx: master process /usr/sbin/nginx
           └─21937 nginx: worker process
Jan 28 19:37:17 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jan 28 19:37:17 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 28 19:37:17 selinux nginx[21930]: nginx: the configuration file /etc/nginx/nginx.conf synt...s ok
Jan 28 19:37:17 selinux nginx[21930]: nginx: configuration file /etc/nginx/nginx.conf test is ...sful
Jan 28 19:37:17 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
Hint: Some lines were ellipsized, use -l to show in full.
```
Проверка сервиса и дальнейшая блокировка порта
```
user@user-VirtualBox:~/SE$ curl http://127.0.0.1:4881
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>

[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http_port_t        
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux
```
[root@selinux ~]# grep nginx /var/log/audit/audit.log
------------
type=SERVICE_START msg=audit(1706470906.746:891): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```
Создание модуля с помощью утилиты audit2allow на основе логов SELinux
```
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
[root@selinux ~]# semodule -i nginx.pp
```
Проверяем
```
[root@selinux ~]# systemctl start nginx 
[root@selinux ~]# systemctl status nginx 
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2024-01-28 19:48:15 UTC; 7s ago
  Process: 21987 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21985 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21984 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21989 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21989 nginx: master process /usr/sbin/nginx
           └─21991 nginx: worker process

Jan 28 19:48:15 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 28 19:48:15 selinux nginx[21985]: nginx: the configuration file /etc/nginx/nginx.conf synt...s ok
Jan 28 19:48:15 selinux nginx[21985]: nginx: configuration file /etc/nginx/nginx.conf test is ...sful
Jan 28 19:48:15 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
Hint: Some lines were ellipsized, use -l to show in full.
```
Удаляю модуль
```
[root@selinux ~]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
```
2. Обеспечение работоспособности приложения при включенном SELinux. Развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems
```
user@user-VirtualBox:~/SE2$ git clone https://github.com/mbfx/otus-linux-adm.git
Клонирование в «otus-linux-adm»...
remote: Enumerating objects: 558, done.
remote: Counting objects: 100% (456/456), done.
remote: Compressing objects: 100% (303/303), done.
remote: Total 558 (delta 125), reused 396 (delta 74), pack-reused 102
Получение объектов: 100% (558/558), 1.38 МиБ | 3.41 МиБ/с, готово.
Определение изменений: 100% (140/140), готово.

user@user-VirtualBox:~/SE2/otus-linux-adm/selinux_dns_problems$ cd otus-linux-adm/selinux_dns_problems

user@user-VirtualBox:~/SE2/otus-linux-adm/selinux_dns_problems$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.

ser@user-VirtualBox:~/SE2/otus-linux-adm/selinux_dns_problems$ vagrant ssh client
Last login: Sun Jan 28 20:05:09 2024 from 10.0.2.2
###############################
### Welcome to the DNS lab! ###
###############################

- Use this client to test the enviroment
- with dig or nslookup. Ex:
    dig @192.168.50.10 ns01.dns.lab

- nsupdate is available in the ddns.lab zone. Ex:
    nsupdate -k /etc/named.zonetransfer.key
    server 192.168.50.10
    zone ddns.lab 
    update add www.ddns.lab. 60 A 192.168.50.15
    send

- rndc is also available to manage the servers
    rndc -c ~/rndc.conf reload

###############################
### Enjoy! ####################
###############################
```
Пробую внести изменения в зону
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key 
> server 192.168.50.15
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```
Смотрю логи на клиенте в поисках причины
```
[vagrant@client ~]$ sudo -i
[root@client ~]# cat /var/log/audit/audit.log | audit2why
[root@client ~]# 
```
Проверяем логи на ns-сервере
```
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1706473239.641:1937): avc:  denied  { create } for  pid=5150 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
Смотрю в каком каталоги должны лежать файлы, чтобы на них распространялись правильные политики SELinux и меняю тип контекста безопасности для каталога /etc/named
```
[root@ns01 ~]# semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0 
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0

[root@ns01 ~]# chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```
Пробую внести изменения на клиенте снова
```
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53258
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 2 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sun Jan 28 20:33:00 UTC 2024
;; MSG SIZE  rcvd: 96
```
Перезапуск клиента и сервера
```
[root@ns01 ~]# shutdown -r -t 0
Shutdown scheduled for Sun 2024-01-28 20:37:50 UTC, use 'shutdown -c' to cancel.

[root@client ~]# shutdown -r -t 0
Shutdown scheduled for Sun 2024-01-28 20:39:17 UTC, use 'shutdown -c' to cancel.
```
Проверка после перезапуска 
```
[vagrant@client ~]$ dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16430
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 10 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sun Jan 28 20:57:59 UTC 2024
;; MSG SIZE  rcvd: 96
```
Возвращение правил обратно делается на сервере
```
[root@ns01 ~]# restorecon -v -R /etc/named
restorecon reset /etc/named context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic context unconfined_u:object_r:named_zone_t:s0->unconfined_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1.jnl context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.newdns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.50.168.192.rev context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
```
