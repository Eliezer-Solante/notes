### Name Resolution

![[Pasted image 20260715162828.png]]

![[Pasted image 20260715163503.png]]

![[Pasted image 20260715164458.png]]
If ever there are two similar Names in each local and DNS server, the hosts will depend on the order on what to lookup first in the `/etc/nsswitch.conf` which has `hosts:   files dns` meaning in this case, it would lookup for the name in the local files first before the DNS server. 




![[Pasted image 20260715165111.png]]


## 📖 Definition of Name Resolution in Linux

- **Translation process** Converts hostnames (like `example.com`) into IP addresses (like `93.184.216.34`) so systems can communicate.
- **Resolver library** The `glibc` resolver handles hostname lookups for applications, consulting configuration files and services.
- **Configuration files**
    - `/etc/hosts` → static mappings of hostnames to IPs.
        bash
        ```
        192.168.1.11   db
        ```
        
    - `/etc/resolv.conf` → defines DNS servers and search domains.
        bash
        ```
        nameserver 8.8.8.8
        nameserver 1.1.1.1
        search mydomain.local
        ```
        
    - `/etc/nsswitch.conf` → sets the order of resolution sources. (`grep hosts /etc/nsswitch.conf` )
        bash 
        ```
        hosts: files dns
        ```
        
- **System services** Modern Linux systems often use `systemd-resolved` for caching, DNSSEC validation, and per-interface DNS settings.
    

## ⚙️ Resolution Flow

- Application requests hostname resolution.
- Resolver checks `/etc/nsswitch.conf` for source order.
- Local files (`/etc/hosts`) are checked first.
- DNS servers are queried if needed.
- Results are cached for efficiency.

---

### Domain Names

![[Pasted image 20260715165348.png]]

![[Pasted image 20260715165429.png]]

![[Pasted image 20260715165605.png]]

![[Pasted image 20260715165809.png]]

---
### Search Domain

![[Pasted image 20260715170436.png]]
- **Definition** A search domain is a suffix automatically appended to a hostname during resolution if the hostname is not fully qualified.
- **Purpose** It allows users to type short hostnames instead of full domain names, making local network access easier.
- **Configuration** Defined in `/etc/resolv.conf` using the `search` directive.
    bash
    ```
    search mydomain.local company.net
    ```
    - Multiple domains can be listed. 
    - The resolver will try each in order until resolution succeeds.

    

---
### Record Types

![[Pasted image 20260715170940.png]]
## 📖 A Record (IPv4 Address)

- **Definition** Maps a hostname to an IPv4 address.
- **Purpose** Allows systems to resolve a domain name into a 32-bit IPv4 address.
- **Example**
    bash
    ```
    example.com.   IN   A   93.184.216.34
    ```
    → When you run `ping example.com`, the resolver queries DNS and gets `93.184.216.34`.
## 📖 AAAA Record (IPv6 Address)
- **Definition** Maps a hostname to an IPv6 address. 
- **Purpose** Supports modern 128-bit IPv6 addressing, essential for future-proof networking.
- **Example**
    bash
    ```
    example.com.   IN   AAAA   2606:2800:220:1:248:1893:25c8:1946
    ```
    → When you run `ping6 example.com`, the resolver uses the IPv6 address.
## 📖 CNAME Record (Canonical Name / Alias)
- **Definition** Creates an alias from one hostname to another.   
- **Purpose** Simplifies DNS management by pointing multiple names to a single canonical domain. 
- **Example**
    bash
    ```
    www.example.com.   IN   CNAME   example.com.
    ```
    → When you query `www.example.com`, DNS redirects to `example.com`, which then resolves to its A/AAAA record.
## 🛠️ Querying These Records in Linux
- Using **dig**: 
    bash
    ```
    dig example.com A
    dig example.com AAAA
    dig www.example.com CNAME
    ```
- Using **host**: 
    bash 
    ```
    host -t A example.com
    host -t AAAA example.com
    host -t CNAME www.example.com
    ```
## 📊 Quick Comparison

|Record|Purpose|Example|
|---|---|---|
|**A**|IPv4 mapping|`example.com → 93.184.216.34`|
|**AAAA**|IPv6 mapping|`example.com → 2606:2800:...`|
|**CNAME**|Alias|`www.example.com → example.com`|


---

### "nslookup" command
The `nslookup` **command** in Linux (and other systems) is a tool used to query DNS servers and obtain information about domain names, IP addresses, and record types. It’s especially handy for troubleshooting name resolution issues.
![[Pasted image 20260715171426.png]]


---

### "dig" command
The `dig` **command** (Domain Information Groper) is one of the most powerful DNS lookup tools available in Linux. It’s more advanced than `nslookup` and is widely used by administrators for troubleshooting and analyzing DNS records.
![[Pasted image 20260715171516.png]]

