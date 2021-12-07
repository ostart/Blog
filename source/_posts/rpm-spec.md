---
title: SPEC-файл RPM-пакета
date: 2021-12-07 13:15:54
tags: [SPEC, RPM, Linux, bash]
---

SPEC-файл описывает как сборку, так и развёртывание RPM-пакета. SPEC-файл состоит из преамбулы и тела. В преамбуле указаны основные константы нашего будущего RPM пакета. Некоторые параметры являются необходимыми, другие - опциональными.

Необходимыми параметра в преамбуле нижеуказанного SPEC-файла являются:
* Name
* Version
* Release
* Source0
* Group

**Важно, чтобы имя архива (в нашем случае LinuxInstaller-2.15.2.4.tar.gz) и указанные в SPEC-файле параметры Name, Version и Release соответствовали друг другу**!

Наш SPEC-файл разворачивает бинарники из архива LinuxInstaller-2.15.2.4.tar.gz, размещённого в  папке SOURCES, в папку BUILD, копирует их в новую папку BUILDROOT и упаковывает их в RPM-пакет, который появится в папке RPMS (подпапке x86_64) в виде rpm-файла LinuxInstaller-2.15.2-4.x86_64.rpm.
При этом папки BUILD и BUILDROOT очистятся в случае успешной сборки (см. секцию %clean).

SPEC-файл описывает, что rpm-пакет разворачивает содержимое в папку /tmp/LinuxInstaller-2.15.2.4/, запускает bash-скрипт *install* командой ```/bin/bash /tmp/%{name}-%{version}.%{release}/install``` и после установки удаляет папку /tmp/LinuxInstaller-2.15.2.4/ командой:
```bash
rm -rf /tmp/%{name}-%{version}.%{release}/
```

Обратите внимание, что с помощью преамбулы Requires rpm-пакет проверяет наличие установленных пакетов/служб bash и cups, причем cups версии не ниже 2.2.

Секция if проверяет ответ, возвращяемый bash-скриптом *install* и в случае ошибки очищает временную папку за собой:
```bash
if [ $? -ne 0 ] ; then
  echo "INSTALLATOR REPORTED NON-ZERO STATUS. OK FOR UPDATE, ERROR OTHERWISE! CLEAN UP AND EXIT..."
  rm -rf /tmp/%{name}-%{version}.%{release}/
  exit 1
fi
```

Окончательный вид содержимого файла LinuxInstaller-2.15.2.4.spec:
```shell
Name:           LinuxInstaller
Version:        2.15.2
Release:        4
Summary:        LinuxInstaller

License:        GPLv3+
URL:            https://example.com/%{name}
Source0:        %{name}-%{version}.%{release}.tar.gz

Group:          System Environment/Daemons
ExclusiveArch:  x86_64

Requires:       bash
Requires:       cups >= 2.2

%description
LinuxInstaller

%prep
%setup -q -n %{name}-%{version}.%{release}

%build

%install
%{__rm} -rf %{buildroot}
mkdir -p %{buildroot}/tmp
cp -r %{_builddir}/%{name}-%{version}.%{release} %{buildroot}/tmp/

%post
/bin/bash /tmp/%{name}-%{version}.%{release}/install

if [ $? -ne 0 ] ; then
  echo "INSTALLATOR REPORTED NON-ZERO STATUS. OK FOR UPDATE, ERROR OTHERWISE! CLEAN UP AND EXIT..."
  rm -rf /tmp/%{name}-%{version}.%{release}/
  exit 1
fi

rm -rf /tmp/%{name}-%{version}.%{release}/

%clean
%{__rm} -rf %{buildroot}
%{__rm} -rf %{_builddir}/%{name}-%{version}.%{release} 

%files
/tmp/%{name}-%{version}.%{release}


%changelog
* Tue Nov 30 2021 Artem Osta
- First package release with update option
```
