# BIND DNS Server

## What is DNS

![DNS](https://github.com/okorsi/Bind-DNS-how-To/blob/main/screenshots/what-is-dns-server.png?raw=true)

### Prerequisites

This tutorial will be a hands-on demonstration. To follow along, ensure you have the following:

* A Linux server – This example uses the Ubuntu 20.04 server. 
* A non-root user with root privileges or root/administrator user. 
* A domain name pointed to the server IP address – This demo uses the atadomain.io domain and server IP address 172.16.1.10. 

### Installing BIND Packages

The default Ubuntu repository provides BIND packages but doesn’t come installed with your system. You can install BIND as the main DNS Server or authoritative only. BIND gives you powerful features, such as master-slave installation support, DNSSEC support, and built-in Access Control Lists (ACL).

To get started with BIND DNS, you’ll first need to install the BIND packages on your machine with the apt package manager. 

1. Open your terminal and log in to your server.

2. Next, run the apt update command below to update and refresh the repository package index. This command ensures that you are installing the latest version of packages.
```
sudo apt update
```

3. Once updated, run the below apt install command to install BIND packages for the Ubuntu server. 

The bind9-utils and bind9-dnsutils packages provide additional command-line tools for BIND. These packages are useful for testing and managing the BIND DNS server. 
````
sudo apt install bind9 bind9-utils bind9-dnsutils -y
````

4. Lastly, run the systemctl command below to verify the BIND service. 

The BIND package comes with the service named and is automatically started and enabled during the BIND package installation. 
````
# Check if named service enabled
sudo systemctl is-enabled named

# Check named service status
sudo systemctl status named
````

Now you should see the BIND named service is enabled with the status as active (running). At this point, the BIND service will run automatically at system startup/boot. 

### Configuring BIND DNS Server 

You’ve now installed BIND packages on the Ubuntu server, so it’s time to set up the BIND installation on your Ubuntu server. How? By editing BIND and the named service’s configurations. 

All configuration for BIND is available at the /etc/bind/ directory, and configurations for the named service at /etc/default/named. 

1. Edit the /etc/default/named configuration using your preferred editor and add option -4 on the OPTIONS line, as shown below. This option will run the named service on IPv4 only. 
````
OPTIONS="-4 -u bind"
````

Save the changes you made and close the file. 

2. Next, edit the /etc/bind/named.conf.options file and populate the following configuration below the directory "/var/cache/bind"; line. 

This configuration sets the BIND service to run on default UDP port 53 on the server’s localhost and public IP address (172.16.1.10). At the same time, it allows queries from any host to the BIND DNS server using the Cloudflare DNS 1.1.1.1 as the forwarder. 
````
    // listen port and address
    listen-on port 53 { localhost; 172.16.1.10; };

    // for public DNS server - allow from any
    allow-query { any; };

    // define the forwarder for DNS queries
    forwarders { 1.1.1.1; };

    // enable recursion that provides recursive query
    recursion yes;
````

At the bottom, comment out the listen-on-v6 { any; }; line, to disable the named service from running on IPv6. 

3. Lastly, run the following command to verify the BIND configuration. 
````
sudo named-checkconf
````

If there’s no output, the BIND configurations are correct without any error. 

### Setting Up DNS Zones 

At this point, you’ve configured the basic configuration of the BIND DNS Server. You’re ready to create a DNS Server with your domain and add other sub-domains for your applications. You’ll need to define and create a new DNS zones configuration to do so. 

In this tutorial, you’ll create a new Name Server (ns1.atadomain.io) and sub-domains (www.atadomain.io, mail.atadomain.io, vault.atadomain.io). 

1. Edit the /etc/bind/named.conf.local file using your preferred editor and add the following configuration. 

This configuration defines the forward zone (/etc/bind/zones/forward.atadomain.io), and the reverse zone (/etc/bind/zones/reverse.atadomain.io) for the atadomain.io domain name. 
```
zone "atadomain.io" {
    type master;
    file "/etc/bind/zones/forward.atadomain.io";
};

zone "1.16.172.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/reverse.atadomain.io";
};
````

Save the changes and close the file. 

2. Next, run the below command to create a new directory (/etc/bind/zones) for string DNS zones configurations. 
````
mkdir -p /etc/bind/zones/
````

3. Run each command below to copy the default forward and reverse zones configuration to the /etc/bind/zones directory. 
```
# Copy default forward zone
sudo cp /etc/bind/db.local /etc/bind/zones/forward.atadomain.io

# Copy default reverse zone
sudo cp /etc/bind/db.127 /etc/bind/zones/reverse.atadomain.io

# List contents of the /etc/bind/zones/ directory
ls /etc/bind/zones/
````

4. Now, edit the forward zone configuration (/etc/bind/zones/forward.atadomain.io) using your preferred editor and populate the configuration below. 

The forward zone configuration is where you define your domain name and the server IP address. This configuration will translate the domain name to the correct IP address of the server. 

The configuration below creates the following name server and sub-domains: 

* ns1.atadomain.io – The main Name Server for your domain with the IP address 172.16.1.10. 
* MX record for the atadomain.io domain that is handled by the mail.atadomain.io. The MX record is used for the mail server. 
* Sub-domains for applications: www.atadomain.io, mail.atadomain.io, and vault.atadomain.io. 
````
;
; BIND data file for the local loopback interface
;
$TTL    604800
@       IN      SOA     atadomain.io. root.atadomain.io. (
                            2         ; Serial
                        604800         ; Refresh
                        86400         ; Retry
                        2419200         ; Expire
                        604800 )       ; Negative Cache TTL

; Define the default name server to ns1.atadomain.io
@       IN      NS      ns1.atadomain.io.

; Resolve ns1 to server IP address
; A record for the main DNS
ns1     IN      A       172.16.1.10


; Define MX record for mail
atadomain.io. IN   MX   10   mail.atadomain.io.


; Other domains for atadomain.io
; Create subdomain www - mail - vault
www     IN      A       172.16.1.10
mail    IN      A       172.16.1.20
vault   IN      A       172.16.1.50
````

Save the changes and close the file. 

5. Like the forward zone, edit the reverse zone configuration file (/etc/bind/zones/reverse.atadomain.io) and populate the following configuration. 

The reverse zone translates the server IP address to the domain name. The reverse zone or PTR record is essential for services like the Mail server, which affects the Mail server’s reputation. 

The PTR record uses the last block of the IP address, like the PTR record with the number 10 for the server IP address 172.16.1.10. 

This configuration creates the reverse zone or PTR record for the following domains: 

* Name server ns1.atadomain.io with the reverse zone or PTR record 172.16.1.10. 
* PTR record for the domain mail.atadomain.io to the server IP address 172.16.1.20. 
````
;
; BIND reverse data file for the local loopback interface
;
$TTL    604800
@       IN      SOA     atadomain.io. root.atadomain.io. (
                            1         ; Serial
                        604800         ; Refresh
                        86400         ; Retry
                        2419200         ; Expire
                        604800 )       ; Negative Cache TTL

; Name Server Info for ns1.atadomain.io
@       IN      NS      ns1.atadomain.io.
NS1     IN      A       172.168.1.10


; Reverse DNS or PTR Record for ns1.atadomain.io
; Using the last number of DNS Server IP address: 172.16.1.10
10      IN      PTR     ns1.atadomain.io.


; Reverse DNS or PTR Record for mail.atadomain.io
; Using the last block IP address: 172.16.1.20
20      IN      PTR     mail.atadomain.io.
````

Save the changes and close the file. 

6. Now, run the following commands to check and verify BIND configurations.
````
# Checking the main configuration for BIND
sudo named-checkconf

# Checking forward zone forward.atadomain.io
sudo named-checkzone atadomain.io /etc/bind/zones/forward.atadomain.io

# Checking reverse zone reverse.atadomain.io
sudo named-checkzone atadomain.io /etc/bind/zones/reverse.atadomain.io
````
When your configuration is correct, you’ll see an output similar below. 

Paste output

7. Lastly, run the systemctl command below to restart and verify the named service. Doing so applies new changes to the named service. 
````
# Restart named service
sudo systemctl restart named

# Verify named service
sudo systemctl status named
````

Below, you can see the named service status is active (running). 

Paste output






Bind 9 service is managed by systemd. We can start the Bind DNS service and enable it to start at system reboot using the following command:
````
systemctl start named
systemctl enable named
````
check the status of the Bind service
````
systemctl status named
````
we get the following result:
````
● named.service - BIND Domain Name Server
     Loaded: loaded (/lib/systemd/system/named.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2022-05-26 21:38:44 UTC; 1min 14s ago
       Docs: man:named(8)
   Main PID: 90096 (named)
      Tasks: 6 (limit: 2188)
     Memory: 7.5M
        CPU: 316ms
     CGroup: /system.slice/named.service
             └─90096 /usr/sbin/named -u bind

Jul 19 21:38:44 ubuntus named[90096]: network unreachable resolving './DNSKEY/IN': 2001:500:2f::f#53
Jul 19 21:38:44 ubuntus named[90096]: network unreachable resolving './NS/IN': 2001:500:2f::f#53
Jul 19 21:38:44 ubuntus named[90096]: network unreachable resolving './DNSKEY/IN': 2001:500:a8::e#53
Jul 19 21:38:44 ubuntus named[90096]: network unreachable resolving './NS/IN': 2001:500:a8::e#53
Jul 19 21:38:44 ubuntus named[90096]: network unreachable resolving './DNSKEY/IN': 2001:500:1::53#53
Jul 19 21:38:44 ubuntus named[90096]: network unreachable resolving './NS/IN': 2001:500:1::53#53
````
Bind DNS server’s configuration files are located inside /etc/bind directory
````
root@ubuntus:/home/oleksii# ls /etc/bind
bind.keys  db.127  db.empty  named.conf                named.conf.local    rndc.key
db.0       db.255  db.local  named.conf.default-zones  named.conf.options  zones.rfc1918
root@ubuntus:/home/oleksii#
````
Need to edit /etc/bind/named.conf.options file and add forwarders. DNS query will be forwarded to the forwarders when your local DNS server is unable to resolve the query
Uncomment and change the following lines:
````
forwarders {
       8.8.8.8;
};
````
Edit the /etc/bind/named.conf.local file to define the zone for our domain.
````
zone "okorsi.com" {
 type master;
 file "/etc/bind/db.local";
};
zone "0.16.172.in-addr.arpa" {
 type master;
 file "/etc/bind/db.172";
};
````
A brief explanation of above file is shown below:

* okorsi.com is our forward zone.
* 0.16.172.in-addr.arpa is our reverse zone.
* db.local is the name of the forward lookup zone file.
* db.172 is the name of the reverse lookup zone file.

Then, verify the configuration file for any error using the following command:
````
named-checkconf
````
if we have not received any information - then everything is fine

## Configure Forward and Reverse Lookup Zone

![DNS](https://github.com/okorsi/Bind-DNS-how-To/blob/main/screenshots/forward-and-reverse-zones.png?raw=true)

Edit the forward lookup zone file:
````
nano /etc/bind/db.local
````
Make the following changes:
````
$TTL    604800
@       IN      SOA     ubuntus.okorsi.com. root.ubuntus.okorsi.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
@          IN      NS      ubuntus.okorsi.com.
ubuntus    IN      A       172.16.0.10
web1       IN      A       172.16.0.20
web2       IN      A       172.16.0.30
@          IN      AAAA    ::1
````
Where:

* 172.16.0.10: IP address of DNS server.
* 172.16.0.20: IP address of WEB1 server.
* 172.16.0.20: IP address of WEB2 server.
* NS: Name server record.
* A: Address record.
* SOA: Start of authority record.

Edit the reverse lookup zone file:
````
nano /etc/bind/db.172
````
Make the following changes:
````
$TTL    604800
@       IN      SOA     ubuntus.okorsi.com. root.ubuntus.okorsi.com. (
                              1
                         604800
                          86400
                        2419200
                         604800 )
@       IN      NS      ubuntus.okorsi.com.
ubuntus    IN      A       172.16.0.10
web1       IN      A       172.16.0.20
web2       IN      A       172.16.0.30
10       IN      PTR     ubuntus.okorsi.com.
````
Edit the /etc/resolv.conf file and define our DNS server:
````
nano /etc/resolv.conf
````
Add the following lines:
````
search okorsi.com
nameserver 172.16.0.10
````
restart the Bind DNS service to apply the changes:
```
systemctl restart named
````
Next, check the forward and reverse lookup zone file for any syntax error with the following command:
````
named-checkzone forward.okorsi db.local
````
If everything is fine. We should see the following output:
````
zone forward.okorsi/IN: loaded serial 4
OK
`````
To check the reverse lookup zone file, run the following command:
````
named-checkzone reverse.okorsi db.172
````
If everything is fine. We should see the following output:
````
If everything is fine. You should see the following output:
````

## Verify Bind DNS Server

First, run the dig command against our DNS nameserver:
````
dig ubuntus.okorsi.com
````
We should see the following output:
````
; <<>> DiG 9.16.1-Ubuntu <<>> ubuntus.okorsi.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 29810
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: a5919d692166f92001000000627498b926c5a27d5f9c28a1 (good)
;; QUESTION SECTION:
;ubuntus.okorsi.com.	IN	A

;; ANSWER SECTION:
ubuntus.okorsi.com. 604800	IN	A	172.16.0.10
web1.okorsi.com. 604800 IN A 172.16.0.20
web2.okorsi.com. 604800 IN A 172.16.0.30

;; Query time: 0 msec
;; SERVER: 172.16.0.10#53(172.16.0.10)
;; WHEN: Tue May 26 03:40:41 UTC 2022
;; MSG SIZE  rcvd: 96
````
Now, run the dig command against the DNS server’s IP to perform the reverse lookup query as shown below:
````
dig -x 172.16.0.10
````
We will get the following output:
````
; <<>> DiG 9.16.1-Ubuntu <<>> -x 172.16.0.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 55197
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 14afadf1d320160e01000000627498d32b3036329829ce2f (good)
;; QUESTION SECTION:
;10.0.16.172.in-addr.arpa.	IN	PTR

;; ANSWER SECTION:
10.0.16.172.in-addr.arpa. 604800 IN	PTR	ubuntus.okorsi.com.

;; Query time: 0 msec
;; SERVER: 172.16.0.10#53(172.16.0.10)
;; WHEN: Tue May 26 03:41:07 UTC 2022
;; MSG SIZE  rcvd: 118
````
We can also use nslookup command against the DNS server to confirm DNS server name resolution.
````
nslookup ubuntus.okorsi.com
````
We should see name to IP resolution in the following output:
````
Server:		172.16.0.10
Address:	172.16.0.10#53

Name:	ubuntus.okorsi.com
Address: 172.16.0.10
````
Now, run the nslookup command against DNS server IP address to confirm the reverse lookup:
````
nslookup 172.16.0.10
````
We should see the IP address to name resolution in the following output:
````
10.0.16.172.in-addr.arpa	name = ubuntus.okorsi.com.
````







