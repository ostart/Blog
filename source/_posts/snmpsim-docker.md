---
title: SnmpSim. Запускаем в Docker
date: 2021-06-29 13:11:12
tags: [snmpsim, snmp, simulator, printer, принтер, симулятор, docker, докер]
---

В предыдущем посте была рассмотрен [запуск симулятора snmpsim в Windows](https://ostart.github.io/2021/06/22/snmpsim/). Здесь же рассмотрим особенности запуска этого симулятора в Docker.

Для запуска snmpsim в докере мы будем использовать образ python:3.7-slim-buster. Почему не Alpine можно узнать из [статьи на Хабре](https://habr.com/ru/post/486202/). В целом алгоритм достаточно прост: установить из **pip** симулятор *snmpsim*, скопировать папку **data** (содержащую файл *public.snmprec*) и открыть **161** порт. 

И тут **самое важное**: требуется стартовать **snmpsimd.py** непременно из под *process-user* и *process-group* равными **root**.

Ниже подробный Dockerfile:

```dockerfile
FROM python:3.7-slim-buster

RUN pip install --no-cache-dir snmpsim

ADD data /usr/local/snmpsim/data

EXPOSE 161/udp

CMD ["snmpsimd.py", "--agent-udpv4-endpoint=0.0.0.0:161", "--process-user=root", "--process-group=root"]
```
Файл **public.snmprec** из вышеупомянутой папки *data* подробно описан в [предыдущем посте по snmpsim в Windows](https://ostart.github.io/2021/06/22/snmpsim/).

Собственно остаётся перейдти в папку с Dockerfile и выполнить команды:

```cmd
docker build . -t snmp-sim
```
и
```cmd
docker run -p 161:161 snmp-sim
```

Проверить обмен с симулятором можно с помощью программы [MIB Browser](https://www.ireasoning.com/mibbrowser.shtml)

{% asset_img mib-browser.jpg MIB Browser %}