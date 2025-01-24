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

