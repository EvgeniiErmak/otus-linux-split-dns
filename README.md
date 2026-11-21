# Vagrant-стенд с DNS: настраиваем Split-DNS

Домашнее задание курса OTUS «Администратор Linux. Professional».

## Задание

1. Взять стенд https://github.com/erlong15/vagrant-bind
   - добавить ещё один сервер **client2**;
   - завести в зоне **dns.lab** имена: `web1` → client1, `web2` → client2;
   - завести ещё одну зону **newdns.lab** с записью `www` → оба клиента.
2. Настроить **split-dns**:
   - client1 видит обе зоны, но в зоне dns.lab только web1;
   - client2 видит только dns.lab.
3. \* Настроить всё без выключения SELinux.

Формат сдачи: Vagrant + Ansible.

## Структура репозитория

| Путь | Назначение |
|---|---|
| [Vagrantfile](Vagrantfile) | 4 ВМ CentOS 7: ns01, ns02, client, client2 |
| [provisioning/](provisioning/) | Ansible-playbook и все конфигурационные файлы |
| [provisioning/playbook.yml](provisioning/playbook.yml) | настройка всех ВМ (включая SELinux) |
| [provisioning/master-named.conf](provisioning/master-named.conf) | BIND master (ns01) с view |
| [provisioning/slave-named.conf](provisioning/slave-named.conf) | BIND slave (ns02) с view |
| [provisioning/named.dns.lab](provisioning/named.dns.lab) | зона dns.lab (web1 + web2) |
| [provisioning/named.dns.lab.client](provisioning/named.dns.lab.client) | зона dns.lab для client (только web1) |
| [provisioning/named.newdns.lab](provisioning/named.newdns.lab) | зона newdns.lab (www → .15 и .16) |
| [provisioning/servers-resolv.conf.j2](provisioning/servers-resolv.conf.j2) | шаблон resolv.conf для DNS-серверов |
| [provisioning/CentOS-Base.repo](provisioning/CentOS-Base.repo) | архивные репозитории CentOS 7 |

## Схема стенда

| ВМ | IP | Роль |
|---|---|---|
| ns01 | 192.168.50.10 | DNS master |
| ns02 | 192.168.50.11 | DNS slave |
| client | 192.168.50.15 | клиент 1 (web1) |
| client2 | 192.168.50.16 | клиент 2 (web2) |

## Особенности реализации

- **Хост:** Ubuntu 26.04, VirtualBox 7.2, Vagrant 2.4.9.
- **Ansible:** ansible-core **2.16**. Это последняя версия, которая работает с Python 2.7 на CentOS 7. Установка: `uv tool install --python 3.12 "ansible-core<2.17"`.
- **CentOS 7 EOL:** `mirror.centos.org` отключён, поэтому playbook кладёт [CentOS-Base.repo](provisioning/CentOS-Base.repo) с зеркалами `archive.kernel.org/centos-vault` (основное) и `vault.centos.org` (запасное). Установка пакетов повторяется до 5 раз (`retries`).
- **Время:** вместо ntp используется штатный `chronyd`.
- **Split-DNS** реализован через `view` и `acl` с TSIG-ключами `client-key` и `client2-key`:
  - view **client**: `dns.lab` из файла `named.dns.lab.client` (только web1) и `newdns.lab`;
  - view **client2**: полная `dns.lab` и обратная зона;
  - view **default**: все зоны (для остальных).
- **Slave ns02** забирает зоны каждого view, подписывая запрос ключом этого view (`masters { 192.168.50.10 key client-key; }`), поэтому мастер отдаёт ему нужный вариант зоны. У каждой slave-зоны свой файл: BIND не разрешает двум view писать в один файл.
- `named.conf` проверяется `named-checkconf` перед копированием (`validate`).

## \* SELinux (без отключения)

**Проблема.** SELinux работает в режиме `Enforcing`, файлы зон лежат в `/etc/named`. Тип этого каталога (`etc_t` / `named_conf_t`) процесс `named_t` может только читать. В журнале ns02:

~~
named: dumping master file: /etc/named/tmp-XrQXfE2HpX: open: permission denied
~~

Slave-зоны не сохраняются на диск, на мастере не пишутся журналы (`.jnl`) динамической зоны `ddns.lab`.

**Решение:** отдельный play в [playbook.yml](provisioning/playbook.yml) для ns01 и ns02.

~~bash
semanage fcontext -m -t named_zone_t '/etc/named(/.*)?'   # тип для файлов зон
restorecon -Rv /etc/named
setsebool -P named_write_master_zones on                  # разрешить named запись в named_zone_t
~~

**Результат:**
- `getenforce` показывает `Enforcing`;
- ошибок `denied` в журнале нет;
- slave-зоны сохранены на ns02;
- `nsupdate` в `ddns.lab` работает и создаёт `named.ddns.lab.jnl`.

## Запуск и проверка

~~bash
vagrant up
vagrant ssh client  -c 'dig +short web1.dns.lab; dig +short web2.dns.lab; dig +short www.newdns.lab'
vagrant ssh client2 -c 'dig +short web1.dns.lab; dig +short web2.dns.lab; dig +short www.newdns.lab'
vagrant ssh ns02    -c 'getenforce; sudo ls -lZ /etc/named/'
~~

Результат (одинаково через ns01 и ns02):

| | web1.dns.lab | web2.dns.lab | www.newdns.lab |
|---|---|---|---|
| client | 192.168.50.15 | — | 192.168.50.15, 192.168.50.16 |
| client2 | 192.168.50.15 | 192.168.50.16 | — |
