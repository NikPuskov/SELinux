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

