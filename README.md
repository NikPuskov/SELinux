# SELinux

Описание домашнего задания

1. Запустить Nginx на нестандартном порту 3-мя разными способами:

переключатели setsebool;

добавление нестандартного порта в имеющийся тип;

формирование и установка модуля SELinux.

К сдаче:

README с описанием каждого решения (скриншоты и демонстрация приветствуются). 

2. Обеспечить работоспособность приложения при включенном selinux.

развернуть приложенный стенд https://github.com/Nickmob/vagrant_selinux_dns_problems; 

выяснить причину неработоспособности механизма обновления зоны (см. README);

предложить решение (или решения) для данной проблемы;

выбрать одно из решений для реализации, предварительно обосновав выбор;

реализовать выбранное решение и продемонстрировать его работоспособность

-----------------------------------------------------------------------------------------------------------------------------------

1. Разворачиваем предложенный стенд на Vagrant (https://github.com/Nickmob/vagrant_selinux) с доработками (Vagrantfile прилагается)

`vagrant up`

Видим что во время развёртывания стенда попытка запустить nginx завершилась с ошибкой:

![Image alt](https://github.com/NikPuskov/SELinux/blob/main/selinux0.jpg)

Заходим на виртуалку и переходим в root

`vagrant ssh` `sudo -i`

Проверим, что в ОС отключен файервол: `systemctl status firewalld`

Проверим, что конфигурация nginx настроена без ошибок: `nginx -t`

Проверим режим работы SELinux: `getenforce`

![Image alt](https://github.com/NikPuskov/SELinux/blob/main/selinux1.jpg)

I) Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool

Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта

Копируем время, в которое был записан этот лог, и, с помощью утилиты audit2why смотрим `grep 1737705837.722:701 /var/log/audit/audit.log | audit2why`

![Image alt](https://github.com/NikPuskov/SELinux/blob/main/selinux2.jpg)

Утилита audit2why покажет почему трафик блокируется. Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled.

Включим параметр nis_enabled и перезапустим nginx

`setsebool -P nis_enabled on`

`systemctl restart nginx`

`systemctl status nginx`

![Image alt](https://github.com/NikPuskov/SELinux/blob/main/selinux3.jpg)

Заходим в любой браузер на хосте и переходим по адресу `http://127.0.0.1:4881` либо со стенда `curl localhost:4481`

Проверить статус параметра можно с помощью команды: `getsebool -a | grep nis_enabled`

Вернём запрет работы nginx на порту 4881 обратно. Для этого отключим nis_enabled: `setsebool -P nis_enabled off`

После отключения nis_enabled служба nginx снова не запустится.

II) Разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип

Поиск имеющегося типа, для http трафика: `semanage port -l | grep http` 

Добавим порт в тип http_port_t: `semanage port -a -t http_port_t -p tcp 4881` `semanage port -l | grep  http_port_t`

Перезапускаем службу nginx и проверим её работу: `systemctl restart nginx` `systemctl status nginx`

![Image alt](https://github.com/NikPuskov/SELinux/blob/main/selinux4.jpg)

Заходим в любой браузер на хосте и переходим по адресу `http://127.0.0.1:4881` либо со стенда `curl localhost:4481`

Удалить нестандартный порт из имеющегося типа можно с помощью команды: `semanage port -d -t http_port_t -p tcp 4881`

`semanage port -l | grep  http_port_t`

`systemctl restart nginx`

`systemctl status nginx`

![Image alt](https://github.com/NikPuskov/SELinux/blob/main/selinux5.jpg)

III) Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux

Попробуем снова запустить Nginx: `systemctl start nginx`

Посмотрим логи SELinux, которые относятся к Nginx: `grep nginx /var/log/audit/audit.log`

Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту: `grep nginx /var/log/audit/audit.log | audit2allow -M nginx`

Audit2allow сформировал модуль, и сообщил нам команду, с помощью которой можно применить данный модуль: `semodule -i nginx.pp`

Попробуем снова запустить nginx: `systemctl start nginx`

`systemctl status nginx`

![Image alt](https://github.com/NikPuskov/SELinux/blob/main/selinux6.jpg)

После добавления модуля nginx запустился без ошибок. При использовании модуля изменения сохранятся после перезагрузки.

Просмотр всех установленных модулей: `semodule -l`

Для удаления модуля воспользуемся командой: `semodule -r nginx`

------------------------------------------------------------------------------------------------------------------------------

2. Для того, чтобы развернуть стенд потребуется хост, с установленным git и ansible

Работа выполнена на хостовой виртуальной машине с Ubuntu 24.04 с установленным Vagrant, Ansible, VirtualBox, Git

Выполним клонирование репозитория: `git clone https://github.com/Nickmob/vagrant_selinux_dns_problems.git`

Перейдём в каталог со стендом: `cd vagrant_selinux_dns_problems`

В моём случае отредактируем Vagrantfile добавив сторчку `ENV['VAGRANT_SERVER_URL'] = 'https://vagrant.elab.pro'` и закомментировав строчку `#config.vm.box_version = "9.4.20240805"`

Развернём 2 ВМ с помощью vagrant: `vagrant up`

После того, как стенд развернется, проверим ВМ с помощью команды: `vagrant status`

Подключимся к клиенту: `vagrant ssh client`

Попробуем внести изменения в зону: `nsupdate -k /etc/named.zonetransfer.key`

![Image alt](https://github.com/NikPuskov/SELinux/blob/main/selinux7.jpg)

Изменения внести не получилось. Давайте посмотрим логи SELinux, чтобы понять в чём может быть проблема.

Для этого воспользуемся утилитой audit2why: `sudo -i` `cat /var/log/audit/audit.log | audit2why`

Тут мы видим, что на клиенте отсутствуют ошибки. 

Не закрывая сессию на клиенте, подключимся к серверу ns01 и проверим логи SELinux: `vagrant ssh ns01`

`sudo -i` `cat /var/log/audit/audit.log | audit2why`

В логах мы видим, что ошибка в контексте безопасности. Целевой контекст `named_conf_t`.

![Image alt](https://github.com/NikPuskov/SELinux/blob/main/selinux8.jpg)

Для сравнения посмотрим существующую зону (localhost) и её контекст: `ls -alZ /var/named/named.localhost`

У наших конфигов в `/etc/named` вместо типа `named_zone_t` используется тип `named_conf_t`.

Проверим данную проблему в каталоге /etc/named: `ls -laZ /etc/named`

![Image alt](https://github.com/NikPuskov/SELinux/blob/main/selinux9.jpg)

Тут мы также видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды: `sudo semanage fcontext -l | grep named`

Изменим тип контекста безопасности для каталога /etc/named: `sudo chcon -R -t named_zone_t /etc/named`

`ls -laZ /etc/named`

![Image alt](https://github.com/NikPuskov/SELinux/blob/main/selinux10.jpg)

Попробуем снова внести изменения с клиента: `nsupdate -k /etc/named.zonetransfer.key`

`> server 192.168.50.10`

`> zone ddns.lab`

`> update add www.ddns.lab. 60 A 192.168.50.15`

`> send`

`> quit`

`dig www.ddns.lab`

Видим, что изменения применились. Попробуем перезагрузить хосты и ещё раз сделать запрос с помощью dig: `dig @192.168.50.10 www.ddns.lab`

![Image alt](https://github.com/NikPuskov/SELinux/blob/main/selinux11.jpg)

Всё правильно. После перезагрузки настройки сохранились. 

Важно, что мы не добавили новые правила в политику для назначения этого контекста в каталоге. Значит, что при перемаркировке файлов контекст вернётся на тот, который прописан в файле политики.

Для того, чтобы вернуть правила обратно, можно ввести команду: `restorecon -v -R /etc/named`

----------------------------------------------------------------------------------------------------------------------------------------
Причина неработоспособности механизма обновления заключается в том что Selinux блокировал доступ к обновлению файлов динамического обновления для DNS сервера, а также к некоторым файлам ОС, к которым DNS сервер обращается во время своей работы

Другие способы решения:

- Выключить Selinux совсем (не рекомендуется)

- Создать некоторое количество модулей по конкретным ошибкам
