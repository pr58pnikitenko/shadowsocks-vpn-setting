# shadowsocks-vpn-setting

### I. Покупка собственного VPS
Я не буду расписывать как и где купить VPS, выбирайте любого провайдера, который Вам нравиться исходя из цены, удаленности, блокировок в Вашем регионе. Единственное условие, VPS должна быть на kvm\xen\vmware виртуализации.

### II. Установка Linux на VPS
Можете выбрать любой дистрибутив, который Вам предлагает провайдер, но желательно как можно свежую версию. Все дальнейшие команды даны для Ubuntu 20.04. Соответственно для Вашего дистрибутива надо делать поправки на менеджер программ и другие подводные камни.

### III. Подключение к VPS по SSH
Опять же не буду расписывать как это делать. В общих словах: подключение происходит по протоколу ssh, для пользователя root, с паролем, который Вам даст провайдер при установке Linux на VPS. Для Windows используем - Putty, для Linux - ssh агент.

### IV. Установка Shadowsocks в standalone режиме (голый без обфускации)
#### 1.Обновим систему:
```bash
apt update && apt upgrade -y
```

#### 2.Установим nano + mc + htop + wget:
```bash
apt install nano mc htop wget
```

#### 3.Перейдем во временный каталог:
```bash
cd /tmp
```

#### 4.В своем браузере идем на офф. страницу Shadowsocks-rust: [shadowsocks-rust](https://4pda.to/stat/go?u=https%3A%2F%2Fgithub.com%2Fshadowsocks%2Fshadowsocks-rust%2Freleases%2F&e=96860833&f=https%3A%2F%2F4pda.to%2Fforum%2Findex.php%3Fshowtopic%3D744431%26st%3D1580%23entry96860833) и копируем адрес последнего релиза под нашу архитектуру (shadowsocks-v1.13.1.x86_64-unknown-linux-gnu.tar.xz).
В терминале вставляем ссылку и скачиваем последную версию shadowsocks-rust:
```bash
wget https://github.com/shadowsocks/shadowsocks-rust/releases/download/v1.13.1/shadowsocks-v1.13.1.x86_64-unknown-linux-gnu.tar.xz
```
тут вместо “v1.13.1/shadowsocks-v1.13.1.x86_64-unknown-linux-gnu.tar.xz” может быть что-то другое — нужно вставить то, что скопировали выше.

#### 5.Разархивируем полученный архив:
```bash
tar -xf shadowsocks-v1.13.1.x86_64-unknown-linux-gnu.tar.xz
```

#### 6.Копируем разархивированный бинарник в /usr/local/bin, чтобы все красиво работало:
```bash
cp ssserver /usr/local/bin
```

#### 7.Создадим файл конфигурации по адресу /etc/shadowsocks/shadowsocks-rust.json, перед этим создав папку:
```bash
mkdir /etc/shadowsocks/
nano /etc/shadowsocks/shadowsocks-rust.json
```
Вставим следующее:
```json
{
    "server": "0.0.0.0",
    "server_port": 1234,
    "password": "пишешь_сюда_какой_нибудь_длинный_пароль",
    "timeout": 120,
    "method": "chacha20-ietf-poly1305",
    "no_delay": true,
    "fast_open": true,
    "reuse_port": true,
    "workers": 1,
    "ipv6_first": true,
    "nameserver": "8.8.8.8",
    "mode": "tcp_and_udp"
}
```
Для сохранения нажимаем ctrl+x, потом y.
По отзыву форумчанина, иногда "fast_open" мешает, поэтому можете не использовать.
Расшифровка параметров:
* В поле «server» вписываем "0.0.0.0", чтобы SS висел на всех сетевых интерфейсах на порту, который задается переменной "server_port".
* В качестве «server_port» указываем любой порт, с расчетом, что с работы (когда на работе proxy) на него сможете достучаться. Если на работе все режут, то ставьте 443 порт, он всегда открыт.
* «password» — конечно же, делайте пароль подлиннее и рандомнее (используйте генератор паролей).
* «timeout» — время, после которого закрывается соединение, если не поступило никаких данных. На сервере лучше поставить это значение побольше, но не более 600.
* «method» — алгоритм шифрования, «chacha20-ietf-poly1305» очень надёжен, такой трафик никакой злоумышленник не расшифрует, он работает быстро даже на утюге, не поддерживающем аппаратное ускорение шифрования.
* «fast_open» — быстрое открытие соединений, работает на ядрах старше 3.16.
* «reuse_port» — в много поточном приложении позволяет каждому потоку напрямую привязаться к tcp socket’y (адрес:порт). Это позволяет быстрее принимать пакеты.
* «wokers» — количество ядер, доступных серверу.
* «nameserver» — это адрес DNS сервера "1.1.1.1" или "8.8.8.8". Если на VPS есть свой DNS сервер, например unbound, то ставим "127.0.0.1"
* «ipv6_first» — обращение сначала к IPv6-адресу ресурса, если на сервере включен IPv6.
* «mode»:«tcp_and_udp» включает передачу данных как по TCP, так и по UDP.


#### 8.Создадим отдельный systemd unit для shadowsocks-rust:
```bash
nano /usr/lib/systemd/system/shadowsocks-rust.service
```
и вставляем туда это:
```
[Unit]
Description=shadowsocks-rust service
After=network.target
[Service]
ExecStart=/usr/local/bin/ssserver -c /etc/shadowsocks/shadowsocks-rust.json
ExecStop=/usr/bin/killall ssserver
Restart=always
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=ssserver
User=nobody
Group=nogroup
[Install]
WantedBy=multi-user.target
```

#### 9.Сделаем тюнинг нашего VPS для лучшего использования Shadowsocks, отредактируем файл /etc/sysctl.conf:
```bash
nano /etc/sysctl.conf
```
Вставьте в конец файла следующее строчки:
```
# Accept IPv6 advertisements when forwarding is enabled
net.ipv6.conf.all.accept_ra = 2

kernel.sysrq=0
vm.swappiness=0
kernel.core_uses_pid=1
kernel.randomize_va_space=1
kernel.msgmnb=65536
kernel.msgmax=65536
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
net.ipv4.tcp_notsent_lowat = 16384
net.ipv4.tcp_syncookies=0
net.ipv4.ip_forward = 0
net.ipv4.conf.all.accept_source_route=0
net.ipv4.conf.default.accept_source_route=0
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.default.accept_redirects=0
net.ipv4.conf.all.send_redirects=0
net.ipv4.conf.default.send_redirects=0
net.ipv4.conf.all.rp_filter=1
net.ipv4.conf.default.rp_filter=1
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.default.arp_ignore = 1
net.ipv4.icmp_echo_ignore_all=1
net.ipv4.icmp_echo_ignore_broadcasts=1
net.ipv4.icmp_ignore_bogus_error_responses=1
net.ipv4.conf.all.secure_redirects=0
net.ipv4.conf.default.secure_redirects=0
net.ipv4.conf.all.secure_redirects=0
net.ipv4.conf.default.secure_redirects=0

#options for ss

fs.file-max = 131072
net.core.rmem_max = 8388608
net.core.wmem_max = 8388608
net.core.rmem_default = 8388608
net.core.wmem_default = 8388608
net.core.optmem_max = 8388608
net.core.netdev_max_backlog = 131072
net.core.somaxconn = 131072
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_mem = 25600 51200 102400
net.ipv4.tcp_rmem = 4096 1048576 4194304
net.ipv4.tcp_wmem = 4096 1048576 4194304
net.ipv4.tcp_fastopen=3
net.ipv4.tcp_low_latency = 1
net.ipv4.tcp_no_metrics_save = 1
net.ipv4.tcp_adv_win_scale = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_keepalive_time = 150
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_slow_start_after_idle=0
net.ipv4.tcp_max_syn_backlog = 65536
net.ipv4.tcp_max_tw_buckets = 720000
net.ipv4.tcp_mtu_probing = 1

#options for wg if server router
#net.ipv4.ip_forward=1
#net.ipv6.conf.all.forwarding=1
#net.ipv6.conf.all.disable_ipv6=0
#net.ipv6.conf.default.forwarding=1
```

#### 10.Применим изменения без перезагрузки командой:
```bash
sysctl -p
```

#### 11.Проверим результат следующей командой:
```bash
sysctl net.ipv4.tcp_available_congestion_control
```
Должно быть следующее сообщение - net.ipv4.tcp_available_congestion_control = reno cubic bbr.

#### 12.Запускаем сервер SS и делаем его с автозапуском:
```bash
systemctl enable shadowsocks-rust && systemctl start shadowsocks-rust
```

Теперь у Вас есть голый работающий shadowsocks-rust, которым можно пользоваться.
Подключиться к нему можно так: Внешний_IP_адрес:1234

### V. Установка плагина v2ray (websocket-http) НЕ АКТУАЛЬНО, проверяйте пути
Обращаю Ваше внимание, что плагин v2ray будем использовать в websocket-http режиме, так проще, не надо возиться с сертификатами. Кому нужен websocket-tls, google в помощь.
#### 1.Идем на офф. страницу v2ray-plugin: [v2ray-plugin](https://4pda.to/stat/go?u=https%3A%2F%2Fgithub.com%2Fshadowsocks%2Fv2ray-plugin%2Freleases&e=96860833&f=https%3A%2F%2F4pda.to%2Fforum%2Findex.php%3Fshowtopic%3D744431%26st%3D1580%23entry96860833) и копируем адрес последнего релиза под нашу архитектуру (linux-amd64):
```bash
wget https://github.com/shadowsocks/v2ray-plugin/releases/download/v1.3.0/v2ray-plugin-linux-amd64-v1.3.0.tar.gz
```
тут вместо “v1.3.0/v2ray-plugin-linux-amd64-v1.3.0.tar.gz” может быть что-то другое — нужно вставить то, что скопировали выше.

#### 2.Разархивируем сам плагин:
```bash
tar -xf v2ray-plugin-linux-amd64-v1.3.0.tar.gz
```

#### 3.Переносим и переименовываем плагин:
```bash
mv v2ray-plugin_linux_amd64 /etc/shadowsocks-libev/v2ray-plugin
```

#### 4.Позволим v2ray-plugin биндиться к привилегированным портам и сделаем его исполняемым:
```bash
setcap "cap_net_bind_service=+eip" /etc/shadowsocks-libev/v2ray-plugin && chmod +x /etc/shadowsocks-libev/v2ray-plugin
```

#### 5. Создадим отдельный systemd unit для v2ray-plugin, который будет висеть на 80 порту, так как у нас websocket-http режим:
```bash
nano /etc/systemd/system/ss-v2ray-80.service
```
и вставляем туда это:
```
[Unit]
Description=v2ray standalone server service
Documentation=man:shadowsocks-rust(8)
After=network.target
[Service]
Type=simple
User=nobody
Group=nogroup
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
LimitNOFILE=51200
ExecStart=/etc/shadowsocks-libev/v2ray-plugin -server -host www.google.com -localAddr IP_server -localPort 80 -remoteAddr 127.0.0.1 -remotePort 1234 -loglevel none
[Install]
WantedBy=multi-user.target
```
Здесь Вам надо поменять "IP_server" - это внешний IP вашего VPS и "-remotePort 1234" - это порт на котором висит Shadowsocks из раздела "IV".

#### 6.Запускаем сервис и делаем его с автозапуском:
```bash
systemctl enable ss-v2ray-80 && systemctl restart ss-v2ray-80
```
Поздравляю, теперь у Вас есть Shadowsocks + v2ray (websocket-http) !
Подключиться к нему можно так: Внешний_IP_адрес:80

### Va. Установка плагина simple-tls (TLS1.3)
Обращаю Ваше внимание, что плагин simple-tls может сам генерировать ключ и сертификаты для произвольного домена - в этом заключается его фишка, и мы этим воспользуемся, у кого есть настоящие свои сертификаты к своему домену, могут подгружать их. Более подробно читайте здесь [simple-tls](https://4pda.to/stat/go?u=https%3A%2F%2Fgithub.com%2FIrineSistiana%2Fsimple-tls&e=96860833&f=https%3A%2F%2F4pda.to%2Fforum%2Findex.php%3Fshowtopic%3D744431%26st%3D1580%23entry96860833).
#### 1.Перейдем во временный каталог:
```bash
cd /tmp
```

#### 1a.В своем браузере идем на офф. страницу simple-tls: [simple-tls-plugin](https://4pda.to/stat/go?u=https%3A%2F%2Fgithub.com%2FIrineSistiana%2Fsimple-tls%2Freleases%2F&e=96860833&f=https%3A%2F%2F4pda.to%2Fforum%2Findex.php%3Fshowtopic%3D744431%26st%3D1580%23entry96860833) и копируем адрес последнего релиза под нашу архитектуру (simple-tls-linux-amd64).
В терминале вставляем ссылку и скачиваем последную версию simple-tls-plugin:
```bash
wget https://github.com/IrineSistiana/simple-tls/releases/download/v0.7.0/simple-tls-linux-amd64.zip
```
тут вместо “v0.7.0/simple-tls-linux-amd64.zip” может быть что-то другое — нужно вставить то, что скопировали выше.

#### 2.Разархивируем сам плагин:
```bash
unzip simple-tls-linux-amd64.zip
```

#### 3.Переносим и переименовываем плагин:
```bash
mv simple-tls /etc/shadowsocks/simple-tls-plugin
```

#### 4.Позволим simple-tls-plugin биндиться к привилегированным портам и сделаем его исполняемым:
```bash
setcap "cap_net_bind_service=+eip" /etc/shadowsocks/simple-tls-plugin && chmod +x /etc/shadowsocks/simple-tls-plugin
```

#### 5. Создадим отдельный systemd unit для simple-tls-plugin, который будет висеть на 443 порту (хотя можно любой порт, например 4431), так как у нас TLS режим:
```bash
nano /etc/systemd/system/ss-simple-tls-443.service
```
и вставляем туда это:
```
[Unit]
Description=simple-tls standalone server service
Documentation=man:shadowsocks-rust(8)
After=network.target
[Service]
Type=simple
User=nobody
Group=nogroup
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
LimitNOFILE=51200
ExecStart=/etc/shadowsocks/simple-tls-plugin -b :localPort -d 127.0.0.1:remotePort -s -key /etc/shadowsocks/my_ecc_cert.key -cert /etc/shadowsocks/my_ecc_cert.cert
[Install]
WantedBy=multi-user.target
```
Здесь Вам надо поменять ":localPort" - указав порт на котором будет висеть simple-tls, у нас это 443. Также надо поменять "127.0.0.1:remotePort" указать только порт на котором висит Shadowsocks из раздела "IV", у нас 1234. Кроме того следует указать пути до ключа и сертификата.

#### 5.1.Сгенерируем сертификат и ключ:
Перейдем в папку с плагином
```bash
cd /etc/shadowsocks/
```
Выполним генерацию сертификата и ключа для произвольного домена, например google.com:
```bash
./simple-tls-plugin -gen-cert -n google.com -key ./my_ecc_cert.key -cert ./my_ecc_cert.cert
```
В папке "/etc/shadowsocks/" будет лежать ключ "my_ecc_cert.key" и сертификат "my_ecc_cert.cert". Кроме того на экране будет выведен hash сертфиката, его надо скопировать, так как его надо будет вставлять в клиентах.
Здесь "google.com" можно поменять на любой другой сайт, но его также потом нужно будет писать в настройках клиентов.
Если вы вдруг не записали hash сертфиката, то его всегда можно посмореть, для этого достаточно в папке где лежит сертификат выполнить:
```bash
./simple-tls-plugin -hash-cert ./my_ecc_cert.cert
```

6.Запускаем сервис и делаем его с автозапуском:
```bash
systemctl enable ss-simple-tls-443 && systemctl restart ss-simple-tls-443
```

Проверьте, запустился ли сервис и нет ли ошибок:
```bash
systemctl status ss-simple-tls-443
```
Поздравляю, теперь у Вас есть Shadowsocks + simple-tls!
Подключиться к нему можно так: Внешний_IP_адрес:443

### VI. Настройка firewall (ufw)
Так как наш сервер виден всему миру, его надо защитить от ботов и атак. Ковыряться в iptables мы не будем, поэтому для этих целей будем использовать UFW.
#### 1.Установим UFW:
```bash
apt install ufw
```

#### 2.Откроем файл настроек /etc/default/ufw:
```bash
nano /etc/default/ufw
```
Там должно быть "IPV6=yes"

#### 3.Пропишем следующие базовые настройки по умолчанию (все входящие запретить, все исходящие разрешить):
```bash
ufw default deny incoming
```
```bash
ufw default allow outgoing
```

#### 4.Открываем нужные нам порты:
Для ssh
```bash
ufw allow 22/tcp # port for ssh
```
Для голого SS из пункта "IV"
```bash
ufw allow 1234 # port for SS
```
Для v2ray из пункта "V"
```bash
ufw allow 80 # port for v2ray
```
Для simple-tls из пункта "Va"
```bash
ufw allow 443 # port for simple-tls
```

#### 5.Включаем UFW:
```bash
ufw enable
```

#### 6.Смотрим статус работы:
```bash
ufw status
```

### Авторство статьи: scorpid
[4pda profile](https://4pda.to/forum/index.php?showuser=1446501)
