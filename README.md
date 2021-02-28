# SickOS1.2
Desarrollo del CTF SickOS1.2
Download: https://www.vulnhub.com/entry/sickos-12,144/

## Escaneo de Puertos

### 1. Escaneamos todos los puertos TCP

```
nmap -n -P0 -p- -sC -sV -O -T5 -oA full 192.168.78.142
Nmap scan report for 192.168.78.142
Host is up (0.00090s latency).
Not shown: 65533 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 5.9p1 Debian 5ubuntu1.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 66:8c:c0:f2:85:7c:6c:c0:f6:ab:7d:48:04:81:c2:d4 (DSA)
|   2048 ba:86:f5:ee:cc:83:df:a6:3f:fd:c1:34:bb:7e:62:ab (RSA)
|_  256 a1:6c:fa:18:da:57:1d:33:2c:52:e4:ec:97:e2:9e:af (ECDSA)
80/tcp open  http    lighttpd 1.4.28
|_http-server-header: lighttpd/1.4.28
|_http-title: Site doesn't have a title (text/html).
MAC Address: 00:0C:29:E5:F7:14 (VMware)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.10 - 4.11, Linux 3.16 - 4.6, Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


## Enumeración de servicios.

- Sólo encontramos una carpeta llamada /test/
```
root@kali:~/SICKOS# gobuster dir -u http://192.168.78.142/ -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://192.168.78.142/
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-1.0.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2021/02/28 18:41:01 Starting gobuster
===============================================================
/test (Status: 301)
/%7Echeckout%7E (Status: 403)
===============================================================
2021/02/28 18:41:33 Finished
===============================================================
```

<img src="https://github.com/El-Palomo/SickOS1.2/blob/main/sickos1.jpg" width="80%"></img>

- Descargué la imagen y busque cadenas importantes. Sin éxito.
- Probé credenciales básicas sobre SSH. Sin éxito.
- Busqué carpetas y archivos con un listado más grande. Sin éxito.
- La búsqueda con NIKTO no arrojaba nada importante: archivo robot.txt, comentarios en html.
- Lo único que se tiene es la carpeta /test/.

## Identificando Vulnerabilidad

1. Si revisamos con calma identificamos que los métodos soportados en la carpeta /test es diferente al de la carpeta raiz.
Moraleja: Se debe buscar los métodos en cada carpeta

```
root@kali:~/SICKOS# nmap -n -P0 -p 80 --script http-methods.nse 192.168.78.142
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-02-28 18:54 EST
Nmap scan report for 192.168.78.142
Host is up (0.00026s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
MAC Address: 00:0C:29:E5:F7:14 (VMware)

Nmap done: 1 IP address (1 host up) scanned in 15.44 seconds
root@kali:~/SICKOS# nmap -n -P0 -p 80 --script http-methods.nse --script-args http-methods.url-path='/test' 192.168.78.142
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-02-28 18:55 EST
Nmap scan report for 192.168.78.142
Host is up (0.00031s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-methods: 
|   Supported Methods: PROPFIND DELETE MKCOL PUT MOVE COPY PROPPATCH LOCK UNLOCK GET HEAD POST OPTIONS
|   Potentially risky methods: PROPFIND DELETE MKCOL PUT MOVE COPY PROPPATCH LOCK UNLOCK
|_  Path tested: /test
MAC Address: 00:0C:29:E5:F7:14 (VMware)
```
<img src="https://github.com/El-Palomo/SickOS1.2/blob/main/sickos2.jpg" width="80%"></img>












