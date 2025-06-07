# NA-Final-note
## 1. 設定DNS server  
   進入/etc/network/interfaces  
   ```
   auto <interface eg: enp0>  
   iface  <interface eg: enp0s8> inet static  
   address <ip eg: 192.168.16.53>  
   netmask <eg: 255.255.255.0>  
   gateway <eg: 192.168.16.254>  
   ```
   重啟networking
   ```
   sudo systemctl restart networking
   ```
## 2. 設定FQDN
設定 hostname 
```
sudo hostnamectl set-hostname <eg: ns1.16.nasa>
```
確保machine 可以知道自己的FQDN  
進入 /etc/hosts
```
127.0.1.1 <eg: ns1.16.nasa> dnsserver
```
儲存設定
```
sudo systemctl restart systemd-hostnamed
```
## 3.安裝bind9
```
sudo apt install bind9 bind9-utils
```
建立zone file(有包含後面的東西)
在 /etc/bind/zones/db.16.nasa  
順解的zone file
```
$TTL 86400
@       IN SOA ns1.16.nasa. admin.16.nasa. (
        2025042515      ; Serial
        3600    ; Refresh
        1800    ; Retry
        604800  ; Expire
        86400   ; Minimum TTL
)
@       IN      NS       ns1.16.nasa.
ns1     IN      A       192.168.16.53
whoami  IN      A       10.113.16.1
dns     IN      A       192.168.16.153
mail.16.nasa.   MX      10      mail.16.nasa.
mail.16.nasa.   A       192.168.16.25
16.nasa.        MX      10      mail.16.nasa.
ldap    A       192.168.16.16
workstation     A       192.168.16.17
@       IN      TXT     "v=spf1 mx a ip4:192.168.16.254/24 ~all"
_dmarc.16.nasa. IN      TXT     "v=DMARC1; p=reject; adkim=s; aspf=s; rua=mailto:dmarc-report-rua@16.nasa;"

$INCLUDE K16.nasa.+013+14516.key
$INCLUDE K16.nasa.+013+20718.key


mail._domainkey IN      TXT     ( "v=DKIM1; h=sha256; k=rsa; "
          "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxZJcNNfquuiv1Ttjd1M5v1R2ml6XkxQbB2WfKqbOyvOSsuwreu83DTv13BBePpZmcbDBySyNdrgTqL2/22AqrEQATM>          "ZeBnUupX9aLHrbzGAKIsoTCH9V2Dln20cVfjxQ/PIZFOXy1ZdkfQ0oM5RcMk08ov/pKmh5UiOOl9MRs2ZoWx8510efgCBTtYRQ02W6jZe7PMEcBSbdYUNJex3FIKxI7rzogEO5TQ>default._domainkey.mail IN      TXT     ( "v=DKIM1; h=sha256; k=rsa; "
          "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEApxo5kc2QPoUvBPmsBQHjCIaLj3XX9Jn15Jjwmgl0wPIh5t1LnCXYhY1HMpJqQy/Fg2Dp8UvjX2Ww10IFaej5Yh3DwU>          "8/dtT2/3QzzJI36OjrlocLwNLcX5IDPYtRi0G/1dyJ/5R10ZCO+OpLLxg2porhULn7hCINoWiOHmOfei3o4fblsQ7NfqHc2WuGfEx4dogpELsh+Hs3iL4pz5m2UC7bUYKxHa6uiQ>
```
在 /etc/bind/zones/db.192.168.16  
反向解析的zone file
```
$TTL 86400
@       IN SOA  ns1.16.nasa. admin.16.nasa. (
        2024031801 ; Serial
        3600    ; Refresh
        1800    ; Retry
        604800  ; Expire
        86400   ; Minimum TTL
)
@       IN      NS      ns1.16.nasa.
53      IN      PTR     ns1.16.nasa.
153     IN      PTR     dns.16.nasa.

$INCLUDE K16.168.192.in-addr.arpa.+013+04960.key
$INCLUDE K16.168.192.in-addr.arpa.+013+63176.key
```
測試是否有成功解析
```
named-checkzone 16.nasa. /etc/bind/db.16.nasa
```
要讓bind知道 lookup file 在哪裡  
進入/etc/bind/named.conf.local  
add
```
zone "16.nasa" {
        type master;
        file "/etc/bind/zones/db.16.nasa.signed";
};

zone "16.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/zones/db.192.168.16.signed";
};

zone "16.113.10.in-addr.arpa" {
        type master;
        file "/etc/bind/zones/db.10.113.16";
};
```
restart bind9  
```
sudo systemctl restart bind9
```
## 啟用DNSSEC
更改/etc/bind/named.conf.options
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

        // forwarders {
        //      0.0.0.0;
        // };

        //========================================================================
        // If BIND logs error messages about the root key being expired,
        // you will need to update your keys.  See https://www.isc.org/bind-keys
        //========================================================================
        dnssec-validation yes;
        listen-on-v6 { any; };
        recursion no;
        allow-query { any; };
};
```
## 生成key pair(ZSK & KSK)
先進入欲生成的位置
ZSK
```
sudo dnssec-keygen -a ECDSAP256SHA256 -b 256 -n ZONE 16.nasa
sudo dnssec-keygen -a ECDSAP256SHA256 -b 256 -n ZONE 16.168.192.in-addr.arpa
```
KSK
```
sudo dnssec-keygen -a ECDSAP256SHA256 -b 256 -n ZONE -f KSK 16.nasa
sudo dnssec-keygen -a ECDSAP256SHA256 -b 256 -n ZONE -f KSK 16.168.192.in-addr.arpa
```
生成完之後要確定含有TTL以及權限要設對(bind:bind, 600)
## 加入DNS zone record
在 /etc/bind/zones/db.16.nasa加入生成好的key pair(上面已有)
```
$INCLUDE K16.nasa.+013+14516.key
$INCLUDE K16.nasa.+013+20718.key
```
在 /etc/bind/zones/db.192.168.16加入生成好的key pair(上面已有)
```
$INCLUDE K16.168.192.in-addr.arpa.+013+04960.key
$INCLUDE K16.168.192.in-addr.arpa.+013+63176.key
```
sign the zone  
正解的zone
```
sudo dnssec-signzone -A -3 aabbccdd -N increment -o 16.nasa. -t db.16.nasa 
```
反解的zone
```
sudo dnssec-signzone -A -3 aabbccdd -N increment -o 16.168.192.in-addr.arpa. -t db.192.168.16
```
生成完之後要更改調named.conf.local裡面的file到新的檔案
## 輸出DS Record => 送給OJ
```
sudo dnssec-dsfromkey -2 (KSK)
sudo dnssec-dsfromkey -2 (KSK)
```
## 設定resolver
安裝unbound
```
sudo apt install unbound
```
設定 /etc/unbound/unbound.conf，如下
```
server:
#       do-recursive: yes
        tls-service-key: "/etc/unbound/dot_key.pem"
        tls-service-pem: "/etc/unbound/dot_cert.pem"
#       tls-port: 853
        interface: 0.0.0.0
        interface: 0.0.0.0@853
        interface: ::0
        val-permissive-mode: no
        val-log-level: 2
        harden-dnssec-stripped: yes
        #auto-trust-anchor-file: "/var/lib/unbound/root.key"
        access-control: 0.0.0.0/0 allow
        access-control: ::0/0 allow
        domain-insecure: "nasa."
        domain-insecure: "168.192.in-addr.arpa."
#       tls-cert-bundle: /etc/unbound/ca.crt
        verbosity: 3
        unblock-lan-zones: yes
        stub-zone:
                name: "nasa."
                stub-addr: 192.168.254.3
                stub-first: yes

        stub-zone:
                name: "168.192.in-addr.arpa."
#               forward-tls-upstream: yes
                stub-addr: 192.168.254.3

        forward-zone:
                name: "."
                forward-addr: 1.1.1.1
                forward-addr: 1.0.0.1
```

