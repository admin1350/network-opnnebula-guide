# Opennebula guide up internet  for VM
### 1. Создание бриджа в Debian
```bash
apt update && apt install bridge-utils -y
```
### 2. Добавьте в файл `/etc/network/interfaces` стоки 
```bash
# Внутренний мост для OpenNebula
auto br1
iface br1 inet static
    address 192.168.100.1
    netmask 255.255.255.0
    bridge_ports none
    bridge_stp off
    bridge_fd 0
```
### 3. Настройка Nat через iptables что бы подняся интерфейс советую командой `systemctl restart networking` 
#### 3.1  Включаем IP Forwarding 
```bash
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-opennebula-nat.conf
sysctl -p /etc/sysctl.d/99-opennebula-nat.conf
```
#### 3.2 Настраиваем правила (подставьте ваш внешний интерфейс вместо `eth0`):
(Узнать имя внешнего интерфейса можно командой ip route | grep default)
```bash
# Очистка (необязательно, если уже настраивали)
iptables -t nat -F

# Основное правило NAT
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Разрешаем прохождение пакетов
iptables -A FORWARD -i br1 -o eth0 -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```
#### 3.3 Созраняем правила
```bash
apt install iptables-persistent -y
# Во время установки на вопрос "Save current IPv4 rules?" ответьте "Yes"
```
### 4. По желанию проброс портов (DNAT)
```bash
# Проброс порта 2222 хоста на 22 порт ВМ
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 2222 -j DNAT --to-destination 192.168.100.10:22
```
### 5. Последний штрих 
#### 5.1 Заходим в Network > Virtual Network и создаем сеть
##### В  `General` пишем name, например `internet`

<img width="599" height="423" alt="изображение" src="https://github.com/user-attachments/assets/a13f8068-0f09-463e-adf3-e4288ef9f550" />

##### В `Conf` Network делаем mode: Bridged 

<img width="949" height="460" alt="изображение" src="https://github.com/user-attachments/assets/f71e0a2e-cc7d-4d01-b66a-f4ff97bc38ac" />

##### В 'Context' делаем как показано на рис снизу

<img width="1022" height="719" alt="изображение" src="https://github.com/user-attachments/assets/7d813a21-2c31-45d2-8542-f32906076edf" />

#### Теперь нажимаем `Update` заходим в `Addresses` нажимаем на `+Address Range` и выбираем так как показано на скрине, в size пишем сколько вы хотите создать mac адресов, точнее лимит на колличество устройств

<img width="902" height="357" alt="изображение" src="https://github.com/user-attachments/assets/1316a7d9-c01e-4f04-80bd-07784f4aaf21" />

### 6. Поднимаем dns bind9 
##### 6.1 Устанавливаем bind9
```bash
apt update
apt install bind9
```
#### 6.2 Настраиваем `/etc/bind/named.conf.options`
```bash
nano /etc/bind/named.conf.options
```
##### Сам файл `/etc/bind/named.conf.options`
```
options {
        directory "/var/cache/bind";

        // If there is a firewall between you and nameservers you want
        // to talk to, you may need to fix the firewall to allow multiple
        // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders.
        // Uncomment the following block, and insert the addresses replacing
        // the all-0's placeholder.

         forwarders {
                8.8.8.8;
                8.8.4.4;
         };

        //========================================================================
        // If BIND logs error messages about the root key being expired,
        // you will need to update your keys.  See https://www.isc.org/bind-keys
        //========================================================================
        dnssec-validation auto;
        listen-on { 127.0.0.1; 192.168.100.1; };
        listen-on-v6 { none; };
};

```
<img width="955" height="586" alt="изображение" src="https://github.com/user-attachments/assets/fc02e7b8-86d5-4403-8b92-ab1e7959a8e4" />

#### 6.3 Чтобы настройки применились пишем команду `systemctl restart bind9`

### 7. Поднимаем dhcp сервер на ранее созданом `br1`.
#### 7.1 устанавливаем нужный пакет
```bash
sudo apt update
sudo apt install isc-dhcp-server
```
#### 7.2 Что бы dhcp работал именно на `br1` нужно отредатировать файл `/etc/default/isc-dhcp-server` 
```bash
sudo sed -i 's/^INTERFACESv4=".*"/INTERFACESv4="br1"/' /etc/default/isc-dhcp-server
```
#### 7.3 Так же редактируем файл конфигурации `sudo nano /etc/dhcp/dhcpd.conf`, то что приведено будет ниже, нужно просто вставить в конец файла
```bash
subnet 192.168.100.0 netmask 255.255.255.0 {
  range 192.168.100.2 192.168.100.254;
  option routers 192.168.100.1;

  # Указываем IP вашего сервера как основной DNS
  option domain-name-servers 192.168.100.1; 

  option subnet-mask 255.255.255.0;
  option broadcast-address 192.168.100.255;
  default-lease-time 600;
  max-lease-time 7200;
}
```
#### 7.4 Перезапускаем сервис и смотрим статус если статус active поздравляю dhcp работает 
```bash
sudo systemctl restart isc-dhcp-server
systemctl status isc-dhcp-server
```


