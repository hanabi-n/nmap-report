# nmap-report
В этом репозитории используются практики, которые описаны в книге **"The Art of Penetration Testing"**. 
Здесь, я использовала команды и знания из "Chapter 2: Discovering Network hosts" и "Chapter 3: Discovering network services"

Prerequisite
------
До того как приступить к работе, я установила nmap в виртуальной машине Ubuntu и запустила vulnerable virtual machine через hackthebox (вроде называется Photobomb).

## Chapter 2: Discovering Network hosts

В книге есть схема процесса, по которому будем сканировать сеть:

![image](https://user-images.githubusercontent.com/53466808/202814848-195c3a1f-cf28-4ad6-8acf-c6a5831750e8.png)

1. Первый шаг это найти доступные IP адреса, название операционной системы и т.д. Этот шаг позволит отделить нужные и не нужные targets.
2. В следующем этапе, мы получаем открытые порты, запускаем скрипты и узнаем версии операционных систем или программ, которые есть.
3. В последнем шаге уже ищем уязвимости. Можем использовать открытые базы данных с уязвимостями или использовать инструменты для этого. 

Шаг 1
------

Следует начать с того, что в основе nmap лежит обыкновенная команда ping. Она отправляет ICMP запросы, и если хост работает, то он будет отвечать на эти запросы (если не отвечает, то либо у тебя нет IP адреса, либо у него. Ну или его роутер/firewall может блокировать запросы. ладно не буду о cisco). В теории можно взять каждый IP адрес в сабнете и через цикл пинговать каждый. Чтобы не заниматься такой ерундой и придумали :sparkles:**Nmap**:sparkles:.

Первая команда которую, написала (10.10.11.182 это адрес машины из hackthebox):
```
sudo nmap -sn 10.10.11.182 -oA pingsweep -PE
```
Здесь:

```
-sn: Ping Scan - отключить сканирование портов (на это уйдет много времени. главная цель здесь найти открытые хосты).
-oA <basename>: получить оутпут на 3-х форматах: grep, xml, nmap.
-PE/PP/PM: ICMP echo, timestamp, and netmask request discovery probes (делаем обычный пинг)
```

Если отфильтровать через grep, то:

![image](https://github.com/hanabi-n/nmap-report/blob/main/Pictures/Screenshot%20from%202022-11-16%2011-01-32.png?raw=true)

Это говорит о том, что у нас больше 20 открытых хостов в этом сабнете. Все полученные IP адреса, перенесла в файл targets.txt:
![image](https://github.com/hanabi-n/nmap-report/blob/main/Pictures/Screenshot%20from%202022-11-16%2011-02-36.png?raw=true)

Шаг 2
------

Также важно проверить наш хост на открытых 5 популярных портов. Потому что по опыту автора, они в большинстве случаев оказывались самыми эффективными:
```
nmap -Pn -n -p 22,80,443,2222,3389 10.10.11.182 -oA rmisweep
```
Как и предполагалось, они у нас тоже открытые: 
![image](https://user-images.githubusercontent.com/53466808/202817738-82ce94b3-6689-4527-815c-679489af66db.png)

Здесь:
```
-Pn: Treat all hosts as online -- skip host discovery. Потому, что мы уже знаем, что этот хост онлайн.
-n/-R: Never do DNS resolution/Always resolve [default: sometimes]
-p <port ranges>: Only scan specified ports
```
Если этого не достаточно и нужно сканировать все 256 хостов, то автор предлагает классный трюк. Он говорит 

"With two additional flags, the exact same scan can
be sped up drastically by forcing Nmap to test all 256 hosts at a time instead of in
64-host groups, as well as by setting the minimum packets-per-second rate to 1,280." То есть отправляем 1,280 пакетов в секунду каждому хосту. 

```
nmap -Pn -n -p 22,80,443,3389,2222 -iL 10.10.11.1-254 -oA rmisweep --min-hostgroup 256 --min-rate 1280
```
Чтобы получить результат не нужно долго ждать:

![image](https://github.com/hanabi-n/nmap-report/blob/main/Pictures/Screenshot%20from%202022-11-16%2011-09-26.png?raw=true)

Чтобы просканировать такое количество хостов, ушло 4 секунды:

![image](https://github.com/hanabi-n/nmap-report/blob/main/Pictures/Screenshot%20from%202022-11-16%2011-10-37.png?raw=true)
Также здесь можно заметить у каких хостов открытые порты (в некоторых все были закрыты).

Шаг 3
------

Помимо сканирования range of IP addresses нужно также найти сабнеты. Здесь проверяем на доступные сабнеты (их 2):

![image](https://github.com/hanabi-n/nmap-report/blob/main/Pictures/Screenshot%20from%202022-11-16%2011-14-54.png?raw=true)

Здесь сканируются сабнеты адресы, которые **заканчиваются на 1** (то есть по сабнету, они начинаются). Это хорошая практика потому, что по опыту автора многие адреса начинаются на 1 в каждом сабнете. 

Если проверить на сабнеты адреса, которые заканчиваются на 182:

![image](https://user-images.githubusercontent.com/53466808/202819132-5e0dbc78-aab4-4b5d-8d76-b9f58099092b.png)

Как и предполагалось, он только 1 и это наша машина.


## Chapter 3: Discovering network services

В этом разделе подробнее изучается information gathering: находим ОС уязвимой машины, применяем скрипты и т.д. Подробная схема этого раздела:

![image](https://user-images.githubusercontent.com/53466808/202819627-01ecfb41-c6a9-456c-9356-b136876b07de.png)

Шаг 1
------
На данный момент уже есть список targets.txt. На них (уже проверенных, открытых хостах) находим открытые порты и применяем скрипты. 

Если мы хотим знать ОС тип, то можно написать простую команду curl и не делать ничего сверхъестественного:

![image](https://user-images.githubusercontent.com/53466808/202820131-e9a433e2-1e4d-440e-83b3-0fd21fa56e93.png)

Наша машина работает на Ubuntu с сервером Nginx, даже можно заметить его название - photobomb.

Шаг 2
------
Многие сервисы могут находиться на известных портах. Теперь на уже открытых хостах можно это проверить и это не займет много времени:

![image](https://github.com/hanabi-n/nmap-report/blob/main/Pictures/Screenshot%20from%202022-11-16%2011-20-53.png?raw=true)

```
nmap -Pn -n -p 22,25,53,80,443,445,1433,3306,3389,5800,5900,8080,8443 -iL targets.txt -oA quick-sweep
```
Шаг 3
------
Но если ты паранойк, то можно просканировать все 65,535:
```
map -Pn -n -iL hosts/targets.txt -p 0-65535 -sV -A -oA services/full-sweep --min-rate 50000 --min-hostgroup 22
```
Но у себя, я написала (хотела быстрее, поэтому 4000 портов):
```
map -Pn -n -iL targets.txt -p 0-4000 -sV -A -oA services/full-sweep --min-rate 50000 --min-hostgroup 22 --max-retries -T5
```
Это сделала из-за предупреждения, который постоянно выходил (чтобы типа пропускали эти предупреждения):

![image](https://github.com/hanabi-n/nmap-report/blob/main/Pictures/Screenshot%20from%202022-11-19%2003-08-43.png?raw=true)

Здесь:
```
-sV: Probe open ports to determine service/version info
-A: Enable OS detection, version detection, script scanning, and traceroute
```
Через grep можно увидеть список скриптов NSE, которые запустились:

![image](https://user-images.githubusercontent.com/53466808/202821169-10c693f5-07a7-4587-9ed9-8475600d4ca0.png)

Можно также увидеть список http titles, которые были найдены:
![image](https://user-images.githubusercontent.com/53466808/202821346-132c0f0a-735f-4cd7-9a1f-113b80404e6c.png)

Шаг 4 
------

Ruby parsing должен быть здесь, но уже 6 утра так, что не успею сделать, а нужно еще экономику сделать.





