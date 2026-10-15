# HTB-Report-Authortiy
Starting with our target (in this case: 10.129.229.56)


### Recon / Scanning 
`sudo nmap -sC -sV -sT -Pn 10.129.229.56 -oA nmap_Authority`

```
-sC: Script scan

-sV: Version Scan

-sT (or -sS for "stealthier"): TCP Connect Scan

-Pn: Disable pinging (host discovery)

-oA: Output all formats
```

The scan returns back an abundance of ports: 

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-14 08:39 -0700
Nmap scan report for 10.129.229.56
Host is up (0.10s latency).
Not shown: 986 closed tcp ports (conn-refused)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-title: IIS Windows Server
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-14 19:22:17Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-10-14T19:23:12+00:00; +3h42m26s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: othername: UPN:AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: othername: UPN:AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
|_ssl-date: 2026-10-14T19:23:10+00:00; +3h42m26s from scanner time.
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: othername: UPN:AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
|_ssl-date: 2026-10-14T19:23:12+00:00; +3h42m26s from scanner time.
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-10-14T19:23:10+00:00; +3h42m26s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: othername: UPN:AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
8443/tcp open  ssl/http      Apache Tomcat (language: en)
| tls-alpn: 
|_  h2
| ssl-cert: Subject: commonName=172.16.2.118
| Not valid before: 2026-10-12T19:14:38
|_Not valid after:  2028-10-14T06:53:02
|_http-title: Site doesn't have a title (text/html;charset=ISO-8859-1).
|_ssl-date: TLS randomness does not represent time
Service Info: Host: AUTHORITY; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: mean: 3h42m25s, deviation: 0s, median: 3h42m25s
| smb2-time: 
|   date: 2026-10-14T19:23:04
|_  start_date: N/A
```
What we learn from this scan: 

`53`: Used to translate URLs into IPs- the browser sends requests to this port it doesn't already know the URL-to-IP. 

`80`: Most likely a HTTP IIS webserver. 

`88 Kerberos`: Used for Kerberos authentication to an Active Directory.
`464`: Kerberos password resetting and password operations. 

`135 MSRPC/RPC`: Helps Windows with locating Remote Procedures Calls, used for internal Windows communications and remote administration. 

`139 NetBIOS-SSN`: Legacy windows networking and session services.

`389 LDAP`: LDAP used for Active directory queries and domain enumeration.  

`445 SMB`: File shares, domain information, and file accessibility.

`636 LDAPS`: LDAP over TLS encryption.  

`3268 LDAP Global Catalog`: Directory searches across an Active Directory forest.

`3269 Global Catalog TLS`: Enrypted Global Catalog Access

`5985 WinRM over HTTP`: Windows Remote Management, authenticaton and authorization.

`8443 Tomcat/HTTPS`: An alternative web service. 



We can infer from these ports (notably: `88, 135, 389, 464, 3267, 3269, 5985`) that this is a `Windows Active Directory Domain Controller`, running a **website** on `port 80`, **LDAP**, and some sort of **Tomcat webpage** on 8443 (Java-based web applications). 




### Next-step Enumeration

For enumerating the services, I like to typically look at the **SMB share (445) and the Website (80)**.



Though I would also like to take a look at DNS (`port 53`to see if there is any domain information.)
Trying commands like (If you don't have authority.htb added to `/etc/hosts` then use the IP alongside the domain):

`dig authority.htb` Which returns nothing particular.

`dig @10.129.229.56 authority.htb NS` returns: 

```
; <<>> DiG 9.20.29-1-Debian <<>> @10.129.229.56 authority.htb NS
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51666
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;authority.htb.			IN	NS

;; ANSWER SECTION:
authority.htb.		3600	IN	NS	authority.authority.htb.

;; ADDITIONAL SECTION:
authority.authority.htb. 3600	IN	A	10.129.229.56
authority.authority.htb. 3600	IN	AAAA	dead:beef::271f:6abf:e52f:68e

;; Query time: 128 msec
;; SERVER: 10.129.229.56#53(10.129.229.56) (UDP)
;; WHEN: Thu Oct 15 08:18:07 MST 2026
;; MSG SIZE  rcvd: 110
```
the name server: `authority.authority.htb`.

Possible additional basic networking information: `nslookup authority.htb 10.129.229.56`

We can see try DNS zone transferring for additional records: `dig @10.129.229.56 authority.htb AXFR`
```
; <<>> DiG 9.20.29-1-Debian <<>> @10.129.229.56 authority.htb AXFR
; (1 server found)
;; global options: +cmd
; Transfer failed.
```
Failed.

More things can be tried like testing to see if we can check Active Directory or LDAP records, but I'm going to move on to SMB. 

Using `NetExec`, we can poke at the SMB share with: `netexec smb 10.129.229.56`

Results: 
```
SMB         10.129.229.56   445    AUTHORITY        [*] Windows 10 / Server 2019 Build 17763 x64 (name:AUTHORITY) (domain:authority.htb) (signing:True) (SMBv1:None) (Null Auth:True)
```

The scan returns the Netbios hostname (**`AUTHORITY`**), the domain name (**`authority.htb`**), Null authentication (**`true`**).

Further attempt to enumerate netexec: `netexec smb 10.129.229.56 -u '' -p '' --shares` and `netexec smb 10.129.229.56 --shares` return 

```"Error enumerating shares: STATUS_ACCESS_DENIED" --- "Error enumerating shares: STATUS_USER_SESSION_DELETED" respectively.``` respectively.

Trying a random username shows that Guest login is enabled: `netexec smb 10.129.229.56 -u 'guest' -p '' --shares` returns: 
```
SMB         10.129.229.56   445    AUTHORITY        [+] authority.htb\Swag: (Guest)
SMB         10.129.229.56   445    AUTHORITY        [*] Enumerated shares
SMB         10.129.229.56   445    AUTHORITY        Share           Permissions     Remark
SMB         10.129.229.56   445    AUTHORITY        -----           -----------     ------
SMB         10.129.229.56   445    AUTHORITY        ADMIN$                          Remote Admin
SMB         10.129.229.56   445    AUTHORITY        C$                              Default share
SMB         10.129.229.56   445    AUTHORITY        Department Shares                 
SMB         10.129.229.56   445    AUTHORITY        Development     READ            
SMB         10.129.229.56   445    AUTHORITY        IPC$            READ            Remote IPC
SMB         10.129.229.56   445    AUTHORITY        NETLOGON                        Logon server share 
SMB         10.129.229.56   445    AUTHORITY        SYSVOL                          Logon server share
```
Some interesting SMB shares namely: Department shares and Development, but we only have access to Development and IPC$.

We can connect to the Development share using: `smbclient \\10.129.229.56\Development --no-pass`


Success! Inside of the directory: **`Automation/Ansible/`** we can see a couple directories: `ADCS, LDAP, PWM, SHARE`. Each contained an Ansible setup configuration inside. Using these we can get specific version numbers that may be valuable in the future- `Ansble verison 2.10`. There is so much information in the shares: `\automation\ansible\PWM\templates\tomcat-users.xml.j2` give a user: `"admin"/"T0mc@tAdm1n" and "robot" password="T0mc@tR00t"`.

Looking up PWM, we see it is "`an open source password self-service application for LDAP directories.`" Going thru the [GitHub](https://github.com/pwm-project/pwm) we can see that it defaults to port 8443, running Tomcat as its container,


Inside of **`/PWM/ansible_inventory`** and **`defaults/main.yml`** we find: `administrator:Welcome1` and hashvalues for the `LDAP/PWM`. This is also alongside `authority.htb.corp` which we can add.

There is tons of information in the SMB share like different versions for the different running software, make sure to check thru these to identify the layout of Ansible, LDAP, etc.. 

Inside of `\Automation\Ansible\PWM\defaults\` there is a `main.yml` 
```
pwm_run_dir: "{{ lookup('env', 'PWD') }}"

pwm_hostname: authority.htb.corp
pwm_http_port: "{{ http_port }}"
pwm_https_port: "{{ https_port }}"
pwm_https_enable: true

pwm_require_ssl: false

pwm_admin_login: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          32666534386435366537653136663731633138616264323230383566333966346662313161326239
          6134353663663462373265633832356663356239383039640a346431373431666433343434366139
          35653634376333666234613466396534343030656165396464323564373334616262613439343033
          6334326263326364380a653034313733326639323433626130343834663538326439636232306531
          3438

pwm_admin_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          31356338343963323063373435363261323563393235633365356134616261666433393263373736
          3335616263326464633832376261306131303337653964350a363663623132353136346631396662
          38656432323830393339336231373637303535613636646561653637386634613862316638353530
          3930356637306461350a316466663037303037653761323565343338653934646533663365363035
          6531

ldap_uri: ldap://127.0.0.1/
ldap_base_dn: "DC=authority,DC=htb"
ldap_admin_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          63303831303534303266356462373731393561313363313038376166336536666232626461653630
          3437333035366235613437373733316635313530326639330a643034623530623439616136363563
          34646237336164356438383034623462323531316333623135383134656263663266653938333334
          3238343230333633350a646664396565633037333431626163306531336336326665316430613566
          3764
```
We can see the encryption type: `Ansible Value;1.1;AES 256`, going to hashcat website we can see that `mode 16900` can be used to decrypt these hashes (obviously if the wordlists we use have them).

[How to crack Ansible passwords.](https://www.bengrewell.com/cracking-ansible-vault-secrets-with-hashcat/)

Trying to crack it with: `hashcat -m 16900 -a 0 hash.txt /usr/share/wordlist/rockyou.txt`





## Web Page

Make sure to add `authority.htb` and `authority.authority.htb` to the `/etc/hosts/`.

Moving on to the webpage on port 80 which loads the default IIS page. Trying to access authority.htb.corp from earlier- same thing. We can now try to gobust/ffuf/feroxbuster to see if there are any undisclosed directories.

Earlier I mentioned how PWM sits on port 8443- so we can try to connect with http://authority.htb:8843/, we get an error "Bad request, requires TLS"... So then we can add https to the URL and we get access to a login page. This can also be seen in the `main.yml` page where `"pwm_https_enable: true"`.


To be continued..
