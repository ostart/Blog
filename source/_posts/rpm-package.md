---
title: RPM-пакет
date: 2021-12-07 13:15:24
tags: [RPM, Linux, bash]
---

Пакет RPM - это просто файл, содержащий другие файлы и информацию о них, необходимую для развёртывания файлов в операционной системе. Для сборки RPM-пакета мною использовалась операционная система **CentOS 7**.

На систему требуется установить следующие пакеты:
```shell
yum install rpm-build rpm-devel rpmlint rpmdevtools
```

Основные команды для сборщика RPM-пакетам содержатся в [SPEC-файле RPM-пакета](https://ostart.github.io/2021/12/07/rpm-spec/).

Для начала работы из домашней директории текущего пользователя выполните команду в терминале:
```bash
rpmdev-setuptree
```

В результате выполнения команды в домашней директории текущего пользователя появится папка **rpmbuild** следующей структуры:

* BUILD
* RPMS
* SOURCES
* SPECS
* SRPMS

## Сборка RPM-пакета

Для сборки потребуется:
1. [SPEC-файл](https://ostart.github.io/2021/12/07/rpm-spec/) (в нашем случае LinuxInstaller-2.15.2.4.spec), который нужно расположить в папке SPECS
2. Архив tar.gz (в нашем случае LinuxInstaller-2.15.2.4.tar.gz) с бинарниками, и в нашем случае bash-скриптом *install*, который нужно расположить в папке SOURCES

**Важно, чтобы имя архива (в нашем случае LinuxInstaller-2.15.2.4.tar.gz) и указанные в SPEC-файле параметры Name, Version и Release соответствовали друг другу**!

Перед сборкой RPM-пакета рекомендуется применить линтер к SPEC-файлу командой в терминале:
```bash
rpmlint rpmbuild/SPECS/LinuxInstaller-2.15.2.4.spec
```
и поправить ошибки и критические замечания обнаруженные линтером в указанном файле.

Для сборки RPM-пакета из домашней директории текущего пользователя выполните следующую команду в терминале:
```bash
rpmbuild -bb rpmbuild/SPECS/LinuxInstaller-2.15.2.4.spec
```

Ключ -bb означает, что RPM-пакет будет собираться из готовых бинарников. Также RPM-пакет можно собрать из исходников. SPEC-файл разворачивает бинарники из папки SOURCES в папку BUILD, копирует их в новую папку BUILDROOT и упаковывает их в RPM-пакет, который появится в папке RPMS (подпапке x86_64) в rpm-файле **LinuxInstaller-2.15.2-4.x86_64.rpm**. При установке rpm-пакет разворачивает содержимое во временную папку /tmp/LinuxInstaller-2.15.2.4/, запускает bash-скрипт *install* командой ```/bin/bash /tmp/%{name}-%{version}.%{release}/install``` и после установки удаляет временную папку. Основная логика развёртывания файлов вынесена в bash-скрипт *install*, несущественного для рассматриваемого здесь вопроса.

## Развёртывание RPM-пакета

Развернуть RPM-пакет можно как из терминала под root-пользователем, так и из специальной программы в графической оболочке операционной системы.

Для развёртывания RPM-пакета из терминала выполните под root-пользователем следующую команду:
```bash
rpm -i LinuxPrintClientInstaller-2.15.2-4.x86_64.rpm
```

Для удаления RPM-пакета из терминала выполните под root-пользователем следующую команду:
```bash
rpm -ev LinuxPrintClientInstaller-2.15.2-4
```

**Обратите внимание, что при удалении *x86_64.rpm* указывать в конце пакета не надо!**

## Обновление RPM-пакета

Обновление пакета требует собрать версию с большими значениями Version и Release. Обратите внимание, что имя и содержимое архива tar.gz также должены соответствовать этим Version и Release.

Если для обновления использовать GUI операционной системы, то RPM-пакет предыдущей версии будет удалён автоматически в случае успешного развертывания пакета обновления. При установке обновления из командной строки предыдущую версию придётся удалять из операционной системы вручную также из командной строки.

Хорошая [Инструкция по RPM-пакетам](https://rpm-packaging-guide.github.io/).
