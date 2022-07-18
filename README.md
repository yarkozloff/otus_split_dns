# Цели домашнего задания
Создать домашнюю сетевую лабораторию. Изучить основы DNS, научиться работать с технологией Split-DNS в Linux-based системах.
# Описание домашнего задания
Взять стенд https://github.com/erlong15/vagrant-bind
- добавить еще один сервер client2
- завести в зоне dns.lab имена:
-- web1 - смотрит на клиент1
-- web2 смотрит на клиент2
- завести еще одну зону newdns.lab
-- завести в ней запись
- www - смотрит на обоих клиентов

Настроить split-dns
- клиент1 - видит обе зоны, но в зоне dns.lab только web1
- клиент2 видит только dns.lab

Дополнительное задание
* настроить все без выключения selinux

Формат сдачи ДЗ - vagrant + ansible

# Пошаговое выполнение ДЗ
Среда выполнения:
```
root@yarkozloff:/otus/vagrant-bind# hostnamectl | grep "Operating System"
  Operating System: Ubuntu 20.04.3 LTS
root@yarkozloff:/otus/vagrant-bind# vagrant --version
Vagrant 2.2.19
root@yarkozloff:/otus/vagrant-bind# vboxmanage --version
6.1.26_Ubuntur145957
```
## Работа со стендом и настройка DNS
Скачаем себе стенд https://github.com/erlong15/vagrant-bind, перейдём в скаченный каталог и изучим содержимое файлов:
```
root@yarkozloff:/otus# git clone https://github.com/erlong15/vagrant-bind.git
Cloning into 'vagrant-bind'...
remote: Enumerating objects: 27, done.
remote: Total 27 (delta 0), reused 0 (delta 0), pack-reused 27
Unpacking objects: 100% (27/27), 4.38 KiB | 195.00 KiB/s, done.

root@yarkozloff:/otus# cd vagrant-bind/

root@yarkozloff:/otus/vagrant-bind# ls -l
total 12
drwxr-xr-x 2 root root 4096 Jul 16 14:54 provisioning
-rw-r--r-- 1 root root  414 Jul 16 14:54 README.md
-rw-r--r-- 1 root root  820 Jul 16 14:54 Vagrantfile
```
Откроем в vim Vagrantfile. Укажем использовать локальный бокс (чтобы не поднимать vpn для скачивания образа centos7), он загружен с названием centos7.
Каждой машине будет выделено по 256 МБ ОЗУ. В начале файла есть модуль, который отвечает за настройку ВМ с помощью Ansible.
* Параметр ansible.sudo = "true" рекомендуется замененить на
ansible.become = "true", так как ansible.sudo скоро перестанет
использоваться
* Добавлено описание виртуальной машины client2.

Чтобы провижинить машины потребуются некоторые файлы, они находятся в каталоге provisioning:
- playbook.yml — это Ansible-playbook, в котором содержатся инструкции по
настройке нашего стенда
- client-motd — файл, содержимое которого будет появляться перед
пользователем, который подключился по SSH
- named.ddns.lab и named.dns.lab — файлы описания зон ddns.lab и dns.lab
соответсвенно
- master-named.conf и slave-named.conf — конфигурационные файлы, в которых хранятся настройки DNS-сервера
- client-resolv.conf и servers-resolv.conf — файлы, в которых содержатся
IP-адреса DNS-серверов

Содержимое файла Playbook.yml
```
---
- hosts: all
  become: yes
  tasks:
#Установка пакетов bind, bind-utils и ntp
#  - name: install packages
#    yum:
#      name:
#        - bind
#        - bind-utils
#        - ntp
#        - vim
#      state: latest
#      update_cache: true
#Копирование файла named.zonetransfer.key на хосты с правами 0644
#Владелец файла — root, група файла — named
  - name: copy transferkey to all servers and the client
    copy: src=named.zonetransfer.key dest=/etc/named.zonetransfer.key owner=root group=named mode=0644
#Настройка одинакового времени с помощью NTP
  - name: stop and disable chronyd
    service: name=chronyd state=stopped enabled=no
  - name: start and enable ntpd
    service: name=ntpd state=started enabled=yes

#Настройка хоста ns01
- hosts: ns01
  become: yes
  tasks:
#Копирование конфигурации DNS-сервера
  - name: copy named.conf
    copy: src=master-named.conf dest=/etc/named.conf owner=root group=named mode=0640
   
#Копирование файлов с настроками зоны.
  - name: copy zones
    copy: 
      src: "{{ item }}" 
      dest: /etc/named/ 
      owner: root 
      group: named 
      mode: 0660
    with_fileglob:
      - named.d*
      - named.newdns.lab

#Перезапуск службы Named и добавление её в автозагрузку
  - name: ensure named is running and enabled
    service:
      name: named
      state: restarted
      enabled: yes

#Копирование файла resolv.conf
  - name: copy resolv.conf to the servers
    template: src=servers-resolv.conf.j2 dest=/etc/resolv.conf owner=root group=root mode=0644
  
#Изменение прав каталога /etc/named
#Права 670, владелец — root, группа — named
  - name: set /etc/named permissions
    file: path=/etc/named owner=root group=named mode=0670

#Перезапуск службы Named и добавление её в автозагрузку    
  - name: ensure named is running and enabled
    service: name=named state=restarted enabled=yes

#Настройки хоста ns02
- hosts: ns02
  become: yes
  tasks:
  - name: copy named.conf
    copy: src=slave-named.conf dest=/etc/named.conf owner=root group=named mode=0640
 
  - name: copy resolv.conf to the servers
    template: src=servers-resolv.conf.j2 dest=/etc/resolv.conf owner=root group=root mode=0644

  - name: set /etc/named permissions
    file: path=/etc/named owner=root group=named mode=0670

  - name: ensure named is running and enabled
    service: name=named state=restarted enabled=yes

#Настройка клиентских машин    
- hosts: client,client2
  become: yes
  tasks:
  - name: copy resolv.conf to the client
    copy: src=client-resolv.conf dest=/etc/resolv.conf owner=root group=root mode=0644
  
#Копирование конфигруационного файла rndc
  - name: copy rndc conf file
    copy: src=rndc.conf dest=/home/vagrant/rndc.conf owner=vagrant group=vagrant mode=0644

#Настройка сообщения при входе на сервер
  - name: copy motd to the client
    copy: src=client-motd dest=/etc/motd owner=root group=root mode=0644
```
### Заметки по Ansible-playbook:
1) С помощью модуля yum установим необходимые пакеты. Сделать это лучше заранее, поэтому это первая задача в Ansible playbook.
2) Для нормальной работы DNS-серверов, на них должно быть настроено одинаковое время. Для того, чтобы на всех серверах было одинаковое время, нам потребуется настроить NTP. Т.к. в CentOS по умолчанию уже есть NTP-клиент Chrony, его необходимо остановить и выключить. Сделаем это через таску ansible:
```
#Настройка одинакового времени с помощью NTP
  - name: stop and disable chronyd
    service: name=chronyd state=stopped enabled=no
  - name: start and enable ntpd
    service: name=ntpd state=started enabled=yes
```
3) Проверяем на каком адресе и порту работают DNS-серверы
```
[root@ns01 ~]# ss -ulpn | grep named
UNCONN     0      0      192.168.50.10:53                       *:*                   users:(("named",pid=28809,fd=512))
UNCONN     0      0        [::1]:53                    [::]:*                   users:(("named",pid=28809,fd=513))


[root@ns02 ~]# ss -ulpn | grep named
UNCONN     0      0      192.168.50.11:53                       *:*                   users:(("named",pid=28403,fd=512))
UNCONN     0      0        [::1]:53                    [::]:*                   users:(("named",pid=28403,fd=513))
```
Исходя из данной информации, нам нужно подкорректировать файл /etc/resolv.conf для DNS-серверов: на хосте ns01 указать nameserver 192.168.50.10, а на хосте ns02 — 192.168.50.11
В Ansible для этого можно воспользоваться шаблоном с Jinja. Изменим имя файла servers-resolv.conf на servers-resolv.conf.j2 и укажем там следующие условия:
```
root@yarkozloff:/otus/vagrant-bind/provisioning# cp servers-resolv.conf servers-resolv.conf.j2
root@yarkozloff:/otus/vagrant-bind/provisioning# vim servers-resolv.conf.j2
root@yarkozloff:/otus/vagrant-bind/provisioning# cat servers-resolv.conf.j2

domain dns.lab
search dns.lab
#Если имя сервера ns02, то указываем nameserver 192.168.50.11
{% if ansible_hostname == 'ns02' %}
nameserver 192.168.50.11
{% endif %}
#Если имя сервера ns01, то указываем nameserver 192.168.50.10
{% if ansible_hostname == 'ns01' %}
nameserver 192.168.50.10
{% endif %}
```
После внесение измений в файл, внесём измения в ansible-playbook:
Используем вместо модуля copy модуль template:
```
#Копирование файла resolv.conf
  - name: copy resolv.conf to the servers
    template: src=servers-resolv.conf.j2 dest=/etc/resolv.conf owner=root group=root mode=0644
```
### Добавление имён в зону dns.lab с помощью Ansible
На хосте ns01 в файл /etc/named/named.dns.lab необходимо добавить имена клиентов. Для этого допишем в конец файла named.dns.lab (который будет копироваться на сервер с помощью ansible, каталог provisioning) строки:
```
;Web
web1            IN      A       192.168.50.15
web2            IN      A       192.168.50.16
```
Выполняем роль ansible, подклчючаемся к клиенту и проверяем:
```
[vagrant@client ~]$ dig @192.168.50.10 web1.dns.lab
...
;; ANSWER SECTION:
web1.dns.lab.           3600    IN      A       192.168.50.15
...

[vagrant@client ~]$ dig @192.168.50.11 web2.dns.lab
...
;; ANSWER SECTION:
web2.dns.lab.           3600    IN      A       192.168.50.16
...
```
### Создание новой зоны и добавление в неё записей с помощью Ansible
Для того, чтобы прописать на DNS-серверах новую зону нам потребуется:
● На хосте ns01 добавить зону в файл /etc/named.conf. Он же файл provisioning/master-named.conf копируемый на сервер ns01, который далее отправим на сервер ns01:
```
// lab's newdns zone
zone "newdns.lab" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    allow-update { key "zonetransfer.key"; };
    file "/etc/named/named.newdns.lab";
};
```
● На хосте ns02 также добавить зону и указать с какого сервера запрашивать
информацию об этой зоне (фрагмент файла /etc/named.conf). Он же файл provisioning/slave-named.conf, который далее отправим на сервер ns02:
```
// lab's newdns zone
zone "newdns.lab" {
    type slave;
    masters { 192.168.50.10; };
    file "/etc/named/named.newdns.lab";
};
```
● На хосте ns01 создадим файл /etc/named/named.newdns.lab. В конце этого файла добавим записи www. У файла должны быть права 660, владелец — root, группа — named. Относительно ansible это файл provisioning/named.newdns.lab, который далее отправим на сервер ns01:
```
;WWW
www             IN      A       192.168.50.15
www             IN      A       192.168.50.16
```
Добавим в модуль copy наш файл named.newdns.lab:
```
  - name: copy zones
    copy:
      src: "{{ item }}"
      dest: /etc/named/
      owner: root
      group: named
      mode: 0660
    with_fileglob:
      - named.d*
      - named.newdns.lab
```
## Настройка Split-DNS с помощью Ansible
У нас уже есть прописанные зоны dns.lab и newdns.lab. Однако по заданию client1 должен видеть запись web1.dns.lab и не видеть запись web2.dns.lab. Client2 может видеть обе записи из домена dns.lab, но не должен видеть записи домена newdns.lab Осуществить данные настройки нам поможет технология Split-DNS.
Для настройки Split-DNS нужно:
1) Создать дополнительный файл зоны dns.lab, в котором будет прописана только
одна запись. Сразу создаем файл в каталоге provisioning:
```
root@yarkozloff:/otus/vagrant-bind# cat provisioning/named.dns.lab.client
$TTL 3600
$ORIGIN dns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2711201408 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.50.10
ns02            IN      A       192.168.50.11

;Web
web1            IN      A       192.168.50.15
```
2) Внести изменения в файл /etc/named.conf на хостах ns01 и ns02
Прежде всего нужно сделать access листы для хостов client и client2. Сначала сгенерируем ключи для хостов client и client2, для этого на хосте ns01 запустим утилиту tsig-keygen 2 раза. 
Далее вносим изменения в файл master-named.conf, который скопируем на сервер ns01 в /etc/named.conf
Вносим изменения в файл slave-named.conf, который скопируем на сервер ns02 в /etc/named.conf
Провиженим машины, проверяем.
Проверка на client:
```
[root@client ~]# ping www.newdns.lab
PING www.newdns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from client (192.168.50.15): icmp_seq=1 ttl=64 time=0.396 ms
64 bytes from client (192.168.50.15): icmp_seq=2 ttl=64 time=0.033 ms
^C
--- www.newdns.lab ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.033/0.214/0.396/0.182 ms
[root@client ~]# ping web1.dns.lab
PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from client (192.168.50.15): icmp_seq=1 ttl=64 time=0.020 ms
64 bytes from client (192.168.50.15): icmp_seq=2 ttl=64 time=0.028 ms
^C
--- web1.dns.lab ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.020/0.024/0.028/0.004 ms
[root@client ~]# ping web2.dns.lab
ping: web2.dns.lab: Name or service not known
```
На хосте мы видим, что client видит обе зоны (dns.lab и newdns.lab), однако информацию о хосте web2.dns.lab он получить не может.
Проверка на client2:
```
[root@client2 ~]# ping www.newdns.lab
ping: www.newdns.lab: Name or service not known
[root@client2 ~]# ping web1.dns.lab
PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from 192.168.50.15 (192.168.50.15): icmp_seq=1 ttl=64 time=3.43 ms
64 bytes from 192.168.50.15 (192.168.50.15): icmp_seq=2 ttl=64 time=1.33 ms
^C
--- web1.dns.lab ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.332/2.385/3.439/1.054 ms
[root@client2 ~]# ping web2.dns.lab
PING web2.dns.lab (192.168.50.16) 56(84) bytes of data.
64 bytes from client2 (192.168.50.16): icmp_seq=1 ttl=64 time=0.022 ms
64 bytes from client2 (192.168.50.16): icmp_seq=2 ttl=64 time=0.027 ms
^C
--- web2.dns.lab ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.022/0.024/0.027/0.005 ms
```
