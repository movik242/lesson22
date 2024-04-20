# lesson22

***Домашнее задание***

Между двумя виртуалками поднять vpn в режимах: </br>
tun </br>
tap </br>
Описать в чём разница, замерить скорость между виртуальными машинами в туннелях, сделаь вывод об отличающихся показателях скорости. </br>

Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на виртуалку </br>
TUN/TAP режимы VPN </br>
Для выполнения первого пункта необходимо написать Vagrantfile, который будет поднимать 2 виртуальные машины server и client </br>
Типовой Vagrantfile для данной задачи: </br>

После запуска машин из Vagrantfile выполняем следующие действия на server и client машинах устанавливаем epel репозиторий </br>

---
      yum install -y epel-release
---
устанавливаем пакет openvpn, easy-rsa и iperf3 </br>

---
      yum install -y openvpn iperf3
---
Отключаем SELinux (при желании можно написать правило для openvpn) </br>

---
      setenforce 0
---
Работает до ребута 

Настройка openvpn сервера </br>
создаем файл ключ </br>

---
      openvpn --genkey --secret /etc/openvpn/static.key
---
создаём конфигурационный файл vpn-сервера </br>

---
      vi /etc/openvpn/server.conf 
---
Запускаем openvpn сервер и добавлāем в автозагрузку </br>

---
      systemctl start openvpn@server
      systemctl enable openvpn@server
---
Настройка openvpn клиента </br>

---
    vi /etc/openvpn/server.conf
---
На сервер клиента в директории /etc/openvpn/ скопируем файл-ключ static.key, который был создан на сервере </br>

Запускаем openvpn клиент и добавляем в автозагрузку </br>

---
      systemctl start openvpn@server
      systemctl enable openvpn@server
---
Далее необходимо замерить скорость в туннеле  </br>
на openvpn сервере запускаем iperf3 в режиме сервера </br>

---
      iperf3 -s &
---
на openvpn клиенте запускаем iperf3 в режиме клиента и замеряем скорость в туннеле </br>

---
      iperf3 -c 10.10.10.1 -t 40 -i 5
---

![image](https://github.com/movik242/lesson22/assets/143793993/348b1649-2f75-415f-934b-285eb2d03dee)

Повторяем для режима работы tun. Конфигурационные файлы сервера и клиента изменяться только в директиве dev. Делаем выводы о режимах, их достоинствах и недостатках

![image](https://github.com/movik242/lesson22/assets/143793993/9be9d373-3a56-41ef-ac0a-ffee4738164b)

В tap режиме быстрее при проведение теста на виртуалках.

**RAS** на базе OpenVPN

Для выполнения данного задания можно восполязоваться ВМ client, только удалить ключ static.key и остановить сервис openvpn.

Устанавливаем необходимые пакеты

---
    yum install -y easy-rsa
---
Переходим в директорию /etc/openvpn/ и инициализируем pki

---
    cd /etc/openvpn/
    /usr/share/easy-rsa/3.0.8/easyrsa init-pki
---
Сгенерируем необходимые ключи и сертификаты для сервера

---
    echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa build-ca nopass
    echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa gen-req server nopass
    echo 'yes' | /usr/share/easy-rsa/3.0.8/easyrsa sign-req server server
    /usr/share/easy-rsa/3.0.8/easyrsa gen-dh
    openvpn --genkey --secret rr.key
---
    
Сгенерируем сертификаты для клиента

---
    echo 'client' | /usr/share/easy-rsa/3.0.8/easyrsa gen-req client nopass
    echo 'yes' | /usr/share/easy-rsa/3.0.8/easyrsa sign-req client client
---
    
Создадим конфигурационный файл /etc/openvpn/server.conf

---
    port 1207
    proto udp
    dev tun
    ca /etc/openvpn/pki/ca.crt
    cert /etc/openvpn/pki/issued/server.crt
    key /etc/openvpn/pki/private/server.key
    dh /etc/openvpn/pki/dh.pem
    server 10.10.10.0 255.255.255.0
    push "route 192.168.56.0 255.255.255.0"
    ifconfig-pool-persist ipp.txt
    client-to-client
    client-config-dir /etc/openvpn/client
    keepalive 10 120
    comp-lzo
    persist-key
    persist-tun
    status /var/log/openvpn-status.log
    log /var/log/openvpn.log
    verb 3
---

Зададим параметр iroute для клиента 

---
    echo 'iroute 192.168.56.0 255.255.255.0' > /etc/openvpn/client/client
---
Запускаем openvpn сервер и добавляем в автозагрузку

---
    systemctl start openvpn@server
    systemctl enable openvpn@server
---    
Скопируем следующие файлы сертификатов и ключ для клиента на хостмашину 

---
    /etc/openvpn/pki/ca.crt
    /etc/openvpn/pki/issued/client.crt
    /etc/openvpn/pki/private/client.key
---
    
Файлы рекомендуется расположить в той же директории, что и client.conf

Создадим конфигурационный файл клиента client.conf на хост-машине

---
    dev tun
    proto udp
    remote 192.168.56.10 1207
    client
    resolv-retry infinite
    ca ./ca.crt
    cert ./client.crt
    key ./client.key
    persist-key
    persist-tun
    comp-lzo
    verb 3
---    
В этом конфигурационном файле указано, что файлы сертификатов располагаются в директории, где располагается client.conf. Но при желании можно разместить сертификаты в других директориях и в конфиге скорректировать пути.

После того, как все готово, подключаемся к openvpn сервер с хост-машины

---
    openvpn --config client.conf
---
При успешном подключении проверяем пинг в внутреннему IP адресу сервера в туннеле


![image](https://github.com/movik242/lesson22/assets/143793993/126279f9-2c2c-4964-95be-52b447bf0083)
