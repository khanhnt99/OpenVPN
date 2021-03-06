# Cấu Hình OpenVPN Client-to-Site trên Centos7 sử dụng routing-tunel
## Mục lục

[1.Mô hình mạng] (#1)

[2.Giới thiệu] (#2)

[3.Các bước triển khai] (#3)

[4.Cấu hình chi tiết] (#4)

[5.Tham Khảo] (#5)

<a name="1"></a>
### 1.Mô hình mạng
<img src="http://image.prntscr.com/image/94c6defab9314fdca825036b6f64f298.png" />

<a name="2"></a>
### 2.Giới thiệu
```sh
--------------------------------------------------------------
OpenVPN Server
--------------------------------------------------------------
IP Wan		:		192.168.100.13
IP Lan    :   192.168.20.254
IP tunel  :   192.168.30.x
OS				:		Centos 7 Final
```

```sh
--------------------------------------------------------------
VPN Client (bên ngoài)
--------------------------------------------------------------
IP address		:		192.168.100.9
IP tunel:     :   192.168.30.x
OS				:		Windows 7
```

```sh
--------------------------------------------------------------
VPN Client (bên trong mạng lan)
--------------------------------------------------------------
IP address		:		192.168.20.10
OS				:		Windows 7
```

<a name="3"></a>
### 3.Các bước triển khai
- Enable the epel-repository in CentOS.
- Install openvpn, easy-rsa .
- Configure easy-rsa.
- Configure openvpn.
- Start openVPN Server.
- Setting up the OpenVPN client application
- Install OpenVPN Client on Linux

<a name="4"></a>
### 4.Cấu hình chi tiết
#### 4.1.Enable the epel-repository
- `rpm -ivh http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-6.noarch.rpm`

#### 4.2.Install openvpn, easy-rsa
- `yum -y install openvpn easy-rsa`
- Sau khi quá trình cài đặt hoàn tất , chép file server.conf đến thư mục /etc/openvpn/
```sh
# cd /usr/share/doc/openvpn-*/sample/sample-config-files/
# cp server.conf /etc/openvpn
```
#### 4.3.Configure easy-rsa
- Copy các scprit để thiết lập CA,certificate và private keys.
```sh
# mkdir /etc/openvpn/rsa
# cp –rf /usr/share/easy-rsa/2.0/* /etc/openvpn/rsa
```
- Sau đó tiến hành thiết lập theo link sau: https://github.com/kieulam141/OpenVPN/blob/master/CA_Certificate_Keys.md

#### 4.4.Configure OpenVPN Server
```sh
# vi /etc/openvpn/server.conf
```
- chú ý đến đường dẫn các file ca.crt ,server.crt ,server.key
```sh
ca /etc/openvpn/rsa/keys/ca.crt
cert /etc/openvpn/rsa/keys/server.crt
key /etc/openvpn/rsa/keys/server.key  # This file should be kept secret

# Diffie hellman parameters.
# Generate your own with:
#   openssl dhparam -out dh2048.pem 2048
dh /etc/openvpn/rsa/keys/dh2048.pem
```
- Đường mạng sẽ cấp địa chỉ IP khi VPN Client kết nối đến VPN Server
```sh
# Configure server mode and supply a VPN subnet
# for OpenVPN to draw client addresses from.
# The server will take 10.8.0.1 for itself,
# the rest will be made available to clients.
# Each client will be able to reach the server
# on 10.8.0.1. Comment this line out if you are
# ethernet bridging. See the man page for more info.
server 192.168.30.0 255.255.255.0
```
- Route đường mạng trên để có thể kết nối được tới Client bên trong mạng lan
```sh
# Push routes to the client to allow it
# to reach other private subnets behind
# the server.  Remember that these
# private subnets will also need
# to know to route the OpenVPN client
# address pool (10.8.0.0/255.255.255.0)
# back to the OpenVPN server.
;push "route 192.168.10.0 255.255.255.0"
;push "route 192.168.20.0 255.255.255.0"
push "route 192.168.20.0 255.255.255.0"
```
- Cho phép tất cả lưu lượng (như duyệt web và phân giải DNS )đi qua VPN bằng cách bỏ “;” tại dòng này.Nhưng khuyến cáo không nên để tham số này vì
client VPN bên ngoài khi truy cập Internet sẽ chiếm băng thông của VPN Server ra Internet.
```sh
push "redirect-gateway def1 bypass-dhcp"
```
- DNS để phân giải
```sh
# Certain Windows-specific network settings
# can be pushed to clients, such as DNS
# or WINS server addresses.  CAVEAT:
# http://openvpn.net/faq.html#dhcpcaveats
# The addresses below refer to the public
# DNS servers provided by opendns.com.
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
```
- Mặc định khi kết nối đến VPN thì client chỉ có thể nhìn thấy VPN Server mà không nhìn thấy các client # cùng kết nối đến, nếu bạn muốn
thì có thể bỏ ";" tại dòng này
```sh
# Uncomment this directive to allow different
# clients to be able to "see" each other.
# By default, clients will only see the server.
# To force clients to only see the server, you
# will also need to appropriately firewall the
# server's TUN/TAP interface.
;client-to-client
```

#### 4.5.Configure Iptables
- Install the iptables và disable firewalld dùng các câu lệnh sau
```sh
yum install iptables-services -y
systemctl mask firewalld
systemctl enable iptables
systemctl stop firewalld
systemctl start iptables
iptables --flush
```
- Tiếp theo add rules cho iptables để forward các định tuyến của kết nối (mạng 192.168.30.0) tới OpenVPN subnet (mạng 192.168.100.0)
```sh
iptables -I INPUT -i eth0 -m state --state NEW -p udp --dport 1194 -j ACCEPT    #eth0 card wan
# Allow TUN interface connections to OpenVPN server
iptables -I INPUT -i tun+ -j ACCEPT 
# Allow TUN interface connections to be forwarded through other interfaces
iptables -I FORWARD -i tun+ -j ACCEPT
iptables -I FORWARD -i tun+ -o eth0 -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT
iptables -I FORWARD -i eth0 -o tun+ -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT
iptables -t nat -A POSTROUTING -s 192.168.30.0/24 -o eth0 -j MASQUERADE // nat Dải LAN bên trong để có thể ra Internet
```
- Bật Port Forwarding : thêm dòng `net.ipv4.ip_forward=1` vào file /etc/sysctl.conf

#### 4.6 Cấu hình cho client
- Sau khi copy các file trên sang client, thì tiến hành cài đặt OpenVPN client theo link: http://openvpn.net/index.php/open-source/downloads.html
- Sau khi cài đặt copy file client.openvpn ở đường dẫn "C:\Program Files\OpenVPN\sample-config" vào trong đường dẫn "C:\Program Files\OpenVPN\config" và bạn có thể đổi tên theo ý của bạn.
- Copy file "ca.crt", "client01.crt", "client01.key" vào chung đường dẫn với file config hoặc 1 folder nào đó, và phải nhớ đường dẫn để chỉ ra trong file config.
- Mở file client.ovpn và chỉnh sửa:
```sh
client
;dev tap0
dev tun
;proto tcp
proto udp
remote 192.168.100.17 1194      #192.168.100.17:IP VPNServer
resolv-retry infinite
nobind
user nobody
group nobody
persist-key
persist-tun
ca ca.crt           # Nếu bạn để các file certificate của client ở thư mục # thì phải chỉ rõ đường dẫn.
cert client01.crt
key client01.key
remote-cert-tls server
comp-lzo
verb 3
route no-pull # Ra Internet bang gateway cua chinh client, chu khong qua VPN.
route 8.8.8.8
```
- Cuối cùng mở ứng dụng OpenVPN trên client với quyền admin, connect tới VPN Server và sẽ được cấp 1 ip trong dải 192.168.30.0/24.
- Ping đến địa chỉ tunel của VPNServer và địa chỉ của VPN client bên trong để kiểm tra đã thành công hay chưa.
<a name="5"></a>
### 5.Tham Khảo
- https://www.unixmen.com/install-openvpn-centos-7/
