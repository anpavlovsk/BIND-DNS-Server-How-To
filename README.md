# BIND DNS Server How To

### Prerequisites

In this tutorial, you’ll learn how to install and configure a secure BIND DNS Server and verify that sub-domains are resolved to the correct IP address.

This tutorial will be a hands-on demonstration. To follow along, ensure you have the following:

* A Linux server – This example uses the Ubuntu 20.04 server. 
* A non-root user with root privileges or root/administrator user. 
* A domain name pointed to the server IP address – This demo uses the anpavlovsk.com domain and server IP address 192.168.33.10. 

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
````
root@ubuntu2004:/# sudo systemctl is-enabled named
enabled
root@ubuntu2004:/# sudo systemctl status named
● named.service - BIND Domain Name Server
     Loaded: loaded (/lib/systemd/system/named.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-07-20 17:58:33 UTC; 5 days ago
       Docs: man:named(8)
   Main PID: 3745 (named)
      Tasks: 8 (limit: 2273)
     Memory: 17.9M
     CGroup: /system.slice/named.service
             └─3745 /usr/sbin/named -f -4 -u bind

````


### Configuring BIND DNS Server 

You’ve now installed BIND packages on the Ubuntu server, so it’s time to set up the BIND installation on your Ubuntu server. How? By editing BIND and the named service’s configurations. 

All configuration for BIND is available at the /etc/bind/ directory, and configurations for the named service at /etc/default/named. 

1. Edit the /etc/default/named configuration using your preferred editor and add option -4 on the OPTIONS line, as shown below. This option will run the named service on IPv4 only. 
````
OPTIONS="-4 -u bind"
````

Save the changes you made and close the file. 

2. Next, edit the /etc/bind/named.conf.options file and populate the following configuration below the directory "/var/cache/bind"; line. 

This configuration sets the BIND service to run on default UDP port 53 on the server’s localhost and public IP address (192.168.33.10). At the same time, it allows queries from any host to the BIND DNS server using the Cloudflare DNS 1.1.1.1 as the forwarder. 
````
    // listen port and address
    listen-on port 53 { localhost; 192.168.33.10; };

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
````
root@ubuntu2004:/# sudo named-checkconf
root@ubuntu2004:/#
root@ubuntu2004:/#
````

### Setting Up DNS Zones 

At this point, you’ve configured the basic configuration of the BIND DNS Server. You’re ready to create a DNS Server with your domain and add other sub-domains for your applications. You’ll need to define and create a new DNS zones configuration to do so. 

In this tutorial, you’ll create a new Name Server (ns1.anpavlovsk.com) and sub-domains (www.anpavlovsk.com, mail.anpavlovsk.com, vault.anpavlovsk.com). 

1. Edit the /etc/bind/named.conf.local file using your preferred editor and add the following configuration. 

This configuration defines the forward zone (/etc/bind/zones/forward.anpavlovsk.com), and the reverse zone (/etc/bind/zones/reverse.anpavlovsk.com) for the anpavlovsk.com domain name. 
```
zone "anpavlovsk.com" {
    type master;
    file "/etc/bind/zones/forward.anpavlovsk.com";
};

zone "33.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/reverse.anpavlovsk.com";
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
sudo cp /etc/bind/db.local /etc/bind/zones/forward.anpavlovsk.com

# Copy default reverse zone
sudo cp /etc/bind/db.127 /etc/bind/zones/reverse.anpavlovsk.com

# List contents of the /etc/bind/zones/ directory
ls /etc/bind/zones/
````

4. Now, edit the forward zone configuration (/etc/bind/zones/forward.anpavlovsk.com) using your preferred editor and populate the configuration below. 

The forward zone configuration is where you define your domain name and the server IP address. This configuration will translate the domain name to the correct IP address of the server. 

The configuration below creates the following name server and sub-domains: 

* ns1.anpavlovsk.com – The main Name Server for your domain with the IP address 192.168.33.10. 
* MX record for the anpavlovsk.com domain that is handled by the mail.anpavlovsk.com. The MX record is used for the mail server. 
* Sub-domains for applications: www.anpavlovsk.com, mail.anpavlovsk.com, and vault.anpavlovsk.com. 
````
;
; BIND data file for the local loopback interface
;
$TTL    604800
@       IN      SOA     anpavlovsk.com. root.anpavlovsk.com. (
                            2         ; Serial
                        604800         ; Refresh
                        86400         ; Retry
                        2419200         ; Expire
                        604800 )       ; Negative Cache TTL

; Define the default name server to ns1.anpavlovsk.com
@       IN      NS      ns1.anpavlovsk.com.

; Resolve ns1 to server IP address
; A record for the main DNS
ns1     IN      A       192.168.33.10


; Define MX record for mail
anpavlovsk.com. IN   MX   10   mail.anpavlovsk.com.


; Other domains for anpavlovsk.com
; Create subdomain www - mail - vault
www     IN      A       192.168.33.10
mail    IN      A       192.168.33.20
vault   IN      A       192.168.33.50
````

Save the changes and close the file. 

5. Like the forward zone, edit the reverse zone configuration file (/etc/bind/zones/reverse.anpavlovsk.com) and populate the following configuration. 

The reverse zone translates the server IP address to the domain name. The reverse zone or PTR record is essential for services like the Mail server, which affects the Mail server’s reputation. 

The PTR record uses the last block of the IP address, like the PTR record with the number 10 for the server IP address 172.16.1.10. 

This configuration creates the reverse zone or PTR record for the following domains: 

* Name server ns1.anpavlovsk.com with the reverse zone or PTR record 192.168.33.10. 
* PTR record for the domain mail.anpavlovsk.com to the server IP address 192.168.33.10. 
````
;
; BIND reverse data file for the local loopback interface
;
$TTL    604800
@       IN      SOA     anpavlovsk.com. root.anpavlovsk.com. (
                            1         ; Serial
                        604800         ; Refresh
                        86400         ; Retry
                        2419200         ; Expire
                        604800 )       ; Negative Cache TTL

; Name Server Info for ns1.anpavlovsk.com
@       IN      NS      ns1.anpavlovsk.com.
NS1     IN      A       192.168.33.10


; Reverse DNS or PTR Record for ns1.anpavlovsk.com
; Using the last number of DNS Server IP address: 192.168.33.10
10      IN      PTR     ns1.anpavlovsk.com.


; Reverse DNS or PTR Record for mail.anpavlovsk.com
; Using the last block IP address: 192.168.33.20
20      IN      PTR     mail.anpavlovsk.com.
````

Save the changes and close the file. 

6. Now, run the following commands to check and verify BIND configurations.
````
# Checking the main configuration for BIND
sudo named-checkconf

# Checking forward zone forward.anpavlovsk.com
sudo named-checkzone anpavlovsk.com /etc/bind/zones/forward.anpavlovsk.com

# Checking reverse zone reverse.anpavlovsk.com
sudo named-checkzone anpavlovsk.com /etc/bind/zones/reverse.anpavlovsk.com
````
When your configuration is correct, you’ll see an output similar below. 
````
root@ubuntu2004:/# sudo named-checkconf
root@ubuntu2004:/# sudo named-checkzone anpavlovsk.com /etc/bind/zones/forward.anpavlovsk.com
zone anpavlovsk.com/IN: loaded serial 2
OK
root@ubuntu2004:/# sudo named-checkzone anpavlovsk.com /etc/bind/zones/reverse.anpavlovsk.com
zone anpavlovsk.com/IN: loaded serial 1
OK
root@ubuntu2004:/#
````

7. Lastly, run the systemctl command below to restart and verify the named service. Doing so applies new changes to the named service. 
````
# Restart named service
sudo systemctl restart named

# Verify named service
sudo systemctl status named
````

Below, you can see the named service status is active (running). 
````
root@ubuntu2004:/# sudo systemctl restart named
root@ubuntu2004:/# sudo systemctl status named
● named.service - BIND Domain Name Server
     Loaded: loaded (/lib/systemd/system/named.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2022-07-26 20:16:50 UTC; 10s ago
       Docs: man:named(8)
   Main PID: 9205 (named)
      Tasks: 8 (limit: 2273)
     Memory: 16.3M
     CGroup: /system.slice/named.service
             └─9205 /usr/sbin/named -f -4 -u bind

Jul 26 20:16:50 ubuntu2004.localdomain named[9205]: zone anpavlovsk.com/IN: loaded serial 2
Jul 26 20:16:50 ubuntu2004.localdomain named[9205]: zone localhost/IN: loaded serial 2
Jul 26 20:16:50 ubuntu2004.localdomain named[9205]: zone 127.in-addr.arpa/IN: loaded serial 1
Jul 26 20:16:50 ubuntu2004.localdomain named[9205]: zone 255.in-addr.arpa/IN: loaded serial 1
Jul 26 20:16:50 ubuntu2004.localdomain named[9205]: all zones loaded
Jul 26 20:16:50 ubuntu2004.localdomain named[9205]: running
Jul 26 20:16:50 ubuntu2004.localdomain named[9205]: zone 33.168.192.in-addr.arpa/IN: sending notifies (serial 1)
Jul 26 20:16:50 ubuntu2004.localdomain named[9205]: zone anpavlovsk.com/IN: sending notifies (serial 2)
Jul 26 20:16:50 ubuntu2004.localdomain named[9205]: managed-keys-zone: Key 20326 for zone . is now trusted (acceptance timer complete)
Jul 26 20:16:51 ubuntu2004.localdomain named[9205]: resolver priming query complete
root@ubuntu2004:/#
````


### Verifying BIND DNS Server Installation 

You’ve now completed the BIND DNS installation. But how do you verify your DNS Server installation? The dig command will do the trick. 

Dig is a command-line utility for troubleshooting DNS Server installation. dig performs DNS lookup for the given domain name and displays detailed answers for the domain name target. On the Ubuntu system, dig is part of the bind9-dnsutil package. 

To verify your BIND DNS server installation: 

1. Run each dig command below to verify the sub-domains www.anpavlovsk.com, mail.anpavlovsk.com, and vault.anpavlovsk.com.

If your DNS Server installation is successful, each sub-domain will be resolved to the correct IP address based on the forward.anpavlovsk.com configuration.
````
# Checking the domain names
dig @192.168.33.10 www.anpavlovsk.com
dig @192.168.33.10 mail.anpavlovsk.com
dig @192.168.33.10 vault.anpavlovsk.com
````

Below is the output of the sub-domain www.anpavlovsk.com resolved to the server IP address 192.168.33.10. 
````
root@ubuntu2004:/# dig @192.168.33.10 www.anpavlovsk.com

; <<>> DiG 9.16.1-Ubuntu <<>> @192.168.33.10 www.anpavlovsk.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 43374
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 534d9bb47036240f0100000062e04c66ecaaddcae9e626b8 (good)
;; QUESTION SECTION:
;www.anpavlovsk.com.            IN      A

;; ANSWER SECTION:
www.anpavlovsk.com.     604800  IN      A       192.168.33.10

;; Query time: 0 msec
;; SERVER: 192.168.33.10#53(192.168.33.10)
;; WHEN: Tue Jul 26 20:19:50 UTC 2022
;; MSG SIZE  rcvd: 91

root@ubuntu2004:/#

````

Below is the sub-domain mail.anpavlovsk.com resolved to the server IP address 192.168.33.20. 
````
root@ubuntu2004:/# dig @192.168.33.10 mail.anpavlovsk.com

; <<>> DiG 9.16.1-Ubuntu <<>> @192.168.33.10 mail.anpavlovsk.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 35886
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 8f76a081218212eb0100000062e04c98756766ab98ec9dfd (good)
;; QUESTION SECTION:
;mail.anpavlovsk.com.           IN      A

;; ANSWER SECTION:
mail.anpavlovsk.com.    604800  IN      A       192.168.33.20

;; Query time: 0 msec
;; SERVER: 192.168.33.10#53(192.168.33.10)
;; WHEN: Tue Jul 26 20:20:40 UTC 2022
;; MSG SIZE  rcvd: 92

root@ubuntu2004:/#
````

And below is the sub-domain vault.anpavlovsk.com resolved to the server IP address 192.168.33.50. 
````
root@ubuntu2004:/# dig @192.168.33.10 vault.anpavlovsk.com

; <<>> DiG 9.16.1-Ubuntu <<>> @192.168.33.10 vault.anpavlovsk.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 44512
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: c2915c11e3c5ce360100000062e04cbd4d0b9e9d28349e88 (good)
;; QUESTION SECTION:
;vault.anpavlovsk.com.          IN      A

;; ANSWER SECTION:
vault.anpavlovsk.com.   604800  IN      A       192.168.33.50

;; Query time: 0 msec
;; SERVER: 192.168.33.10#53(192.168.33.10)
;; WHEN: Tue Jul 26 20:21:17 UTC 2022
;; MSG SIZE  rcvd: 93

root@ubuntu2004:/#
````

2. Next, run the dig command below to verify the MX record for the anpavlovsk.com domain. 
````
dig @192.168.33.10 anpavlovsk.com MX
````

You should see the anpavlovsk.com domain has the MX record mail.anpavlovsk.com. 
````
root@ubuntu2004:/# dig @192.168.33.10 anpavlovsk.com MX

; <<>> DiG 9.16.1-Ubuntu <<>> @192.168.33.10 anpavlovsk.com MX
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 26309
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 50490bd4ad1215a40100000062e04d07a427e9958b76f5d9 (good)
;; QUESTION SECTION:
;anpavlovsk.com.                        IN      MX

;; ANSWER SECTION:
anpavlovsk.com.         604800  IN      MX      10 mail.anpavlovsk.com.

;; ADDITIONAL SECTION:
mail.anpavlovsk.com.    604800  IN      A       192.168.33.20

;; Query time: 0 msec
;; SERVER: 192.168.33.10#53(192.168.33.10)
;; WHEN: Tue Jul 26 20:22:31 UTC 2022
;; MSG SIZE  rcvd: 108

root@ubuntu2004:/#
````

3. Lastly, run the following commands to verify the PTR record or reverse zone for the server IP addresses 192.168.33.10 and 192.168.33.20. 

If your BIND installation is successful, each IP address will be resolved to the domain name defined on the reverse.anpavlovsk.com configuration.  
````
# checking PTR record or reverse DNS
dig @192.168.33.10 -x 192.168.33.10
dig @192.168.33.10 -x 192.168.33.20
````

You can see below, the server IP address 192.168.33.10 is resolved to the domain name ns1.anpavlovsk.com. 
````
root@ubuntu2004:/# dig @192.168.33.10 -x 192.168.33.10

; <<>> DiG 9.16.1-Ubuntu <<>> @192.168.33.10 -x 192.168.33.10
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53840
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: bd79e65d36512aef0100000062e04d40c3cec55af10672d5 (good)
;; QUESTION SECTION:
;10.33.168.192.in-addr.arpa.    IN      PTR

;; ANSWER SECTION:
10.33.168.192.in-addr.arpa. 604800 IN   PTR     ns1.anpavlovsk.com.

;; Query time: 0 msec
;; SERVER: 192.168.33.10#53(192.168.33.10)
;; WHEN: Tue Jul 26 20:23:28 UTC 2022
;; MSG SIZE  rcvd: 115

root@ubuntu2004:/#
````

As you see below, the server IP address 192.168.33.20 is resolved to the domain name mail.anpavlovsk.com.
````
root@ubuntu2004:/# dig @192.168.33.10 -x 192.168.33.20

; <<>> DiG 9.16.1-Ubuntu <<>> @192.168.33.10 -x 192.168.33.20
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40772
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: ae0ba41f30d486ff0100000062e04d60552e4257405aedd8 (good)
;; QUESTION SECTION:
;20.33.168.192.in-addr.arpa.    IN      PTR

;; ANSWER SECTION:
20.33.168.192.in-addr.arpa. 604800 IN   PTR     mail.anpavlovsk.com.

;; Query time: 0 msec
;; SERVER: 192.168.33.10#53(192.168.33.10)
;; WHEN: Tue Jul 26 20:24:00 UTC 2022
;; MSG SIZE  rcvd: 116

root@ubuntu2004:/#
````


### Conclusion 

Throughout this tutorial, you’ve learned how to create and set up a secure BIND DNS server on your Ubuntu server. You’ve also created the forward and reverse zone for adding your domain and verified DNS servers by running dig commands.

