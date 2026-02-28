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
### 3. Насройка Nat через iptables что бы подняся интерфейс советую командой `systemctl restart networking` 
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

<img width="947" height="742" alt="изображение" src="https://github.com/user-attachments/assets/64035a96-15ab-4841-afc8-f032d4a65dba" />

#### Теперь нажимаем `Update` заходим в `Addresses` нажимаем на `+Address Range` и выбираем так как показано на скрине, в size пишем сколько вы хотите создать mac адресов, точнее лимит на колличество устройств
<img width="902" height="357" alt="изображение" src="https://github.com/user-attachments/assets/1316a7d9-c01e-4f04-80bd-07784f4aaf21" />
