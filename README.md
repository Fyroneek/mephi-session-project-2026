# MEPHI Session Project 2026

> Практический зачётный проект по дисциплине **«Информационная безопасность»**
> на сертифицированном дистрибутиве **РЕД ОС 8** (сертификат ФСТЭК России № 4060 от 12.01.2019 г.)

---

## 📋 Исходные данные

| Параметр                                       | Значение                  |
| ---------------------------------------------- | ------------------------- |
| **Бухгалтерский шифр (номер зачётной книжки)** | `374352`                  |
| **Имя хоста**                                  | `mephi-2026.domain.local` |
| **Дистрибутив**                                | РЕД ОС 8                  |
| **Сертификат ФСТЭК**                           | № 4060 от 12.01.2019      |
| **Основной пользователь стенда**               | `dmitry` (UID 1000)       |
| **Режим SELinux**                              | `Enforcing`               |

> **Важно.** Переустанавливать дистрибутив не потребовалось: все настройки выполнены из уже установленной системы через командную строку, как разрешено текстом задания («Если программа установки не позволяет выполнить данные настройки, то выполните их после установки системы в командной строке»).

---

## 🗂 Структура репозитория

Все файлы артефактов собраны в корне репозитория. Системные файлы из `/etc` загружены под простыми именами (как требует задание 8.1).

| Файл                   | Источник                                       | Пункт    | Что проверяется                                     |
| ---------------------- | ---------------------------------------------- | -------- | --------------------------------------------------- |
| `ping.out`             | `ping -c 4 8.8.8.8 > ping.out`                 | 1.3      | Работоспособность сети (DHCP + связность с 8.8.8.8) |
| `dnf.out`              | `sudo dnf history > dnf.out`                   | 2.1–2.3  | Управление пакетами (update, install, RPM)          |
| `fstab`                | `/etc/fstab`                                   | 3.2      | Автоматическое монтирование `LABEL=MEPHI_WEB`       |
| `journalctl.out`       | `sudo journalctl -u nginx -b > journalctl.out` | 4.2      | Запуск веб-сервера nginx                            |
| `stat.out`             | `stat /data/mephi-2026` и `stat /mephi-web`    | 5.1, 5.3 | Права доступа (DAC) и контекст SELinux              |
| `getcap.out`           | `getcap /usr/sbin/tcpdump > getcap.out`        | 5.2      | Настройка привилегий tcpdump                        |
| `getenforce.out`       | `getenforce > getenforce.out`                  | 5.3      | Режим SELinux (Enforcing)                           |
| `passwd`               | `/etc/passwd`                                  | 5.1, 6.1 | Управление пользователями и shell                   |
| `shadow`               | `/etc/shadow` (с затёртыми хешами)             | 6.2      | Управление паролями (PASS_MAX_DAYS)                 |
| `group`                | `/etc/group`                                   | 5.1      | Управление группами (mephi, curators)               |
| `pwquality.conf`       | `/etc/security/pwquality.conf`                 | 6.2      | Настройка парольной политики (minlen = 12)          |
| `curl.out`             | `curl -s http://localhost/ > curl.out`         | 7.2      | Результат тестирования web-страницы                 |
| `mephi-screenshot.png` | скриншот GUI                                   | 7.2      | Визуальное подтверждение                            |
| `history.out`          | `history > history.out`                        | 8.1      | Выполненные команды                                 |
| `README.md`            | этот файл                                      | 8.1      | Описание соответствия артефактов                    |

---

## 🚀 Выполнение по разделам

### Раздел 1. Установка дистрибутива (5 баллов)

**1.1. Настройка сети (DHCP).** Сетевой интерфейс стенда — `enp0s3`. Переведён в режим DHCP через NetworkManager:

```bash
nmcli connection modify "enp0s3" ipv4.method auto
nmcli connection up "enp0s3"
nmcli connection show "enp0s3" | grep ipv4.method   # → auto
```

**1.2. Настройка имени хоста.**

```bash
sudo hostnamectl set-hostname mephi-2026.domain.local
hostname -f   # → mephi-2026.domain.local
```

**1.3. Проверка сетевой связности.** Результат сохранён в [`ping.out`](ping.out) — 4 пакета передано, 4 получено, `0% packet loss`.

---

### Раздел 2. Управление программным обеспечением (5 баллов)

**2.1. Обновление дистрибутива:**

```bash
sudo dnf clean all && sudo dnf makecache
sudo dnf update -y
```

**2.2. Установка пакетов из репозиториев:**

```bash
sudo dnf install -y nginx libcap-ng-utils libcap
```

**2.3. Установка локального RPM-пакета tcpdump** (через `dnf download` + `rpm -ivh`):

```bash
sudo dnf install -y libpcap          # зависимости заранее
cd /tmp && sudo dnf download tcpdump
sudo rpm -ivh /tmp/tcpdump-*.rpm
```

**2.4. История транзакций dnf** сохранена в [`dnf.out`](dnf.out). Видны транзакции:

- `#4` — install nginx, libcap-ng-utils, libcap;
- `#3` — полное обновление дистрибутива (I, O, U — install/obsolete/update, 395 пакетов);
- `#2` — удаление open-vm-tools, spice-vdagent;
- `#1` — первичная установка системы (1868 пакетов).

---

### Раздел 3. Управление файловыми системами (5 баллов)

**3.1. Создание файловой системы на втором диске `/dev/sdb`:**

```bash
sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary ext4 0% 100%
sudo mkfs.ext4 -L MEPHI_WEB /dev/sdb1
sudo blkid /dev/sdb1   # → LABEL="MEPHI_WEB" TYPE="ext4"
```

**3.2. Монтирование в `/mephi-web` с использованием метки тома.** Запись в [`fstab`](fstab):

```
LABEL=MEPHI_WEB /mephi-web ext4 defaults 0 2
```

Проверка и применение:

```bash
sudo findmnt --verify --verbose
sudo mount -a
findmnt /mephi-web   # → SOURCE=LABEL=MEPHI_WEB
```

---

### Раздел 4. Управление сервисами (5 баллов)

**4.1. Запуск nginx и включение автозагрузки:**

```bash
sudo systemctl enable --now nginx
systemctl is-enabled nginx   # → enabled
systemctl is-active nginx    # → active
```

**4.2. Журнал nginx за текущую загрузку** сохранён в [`journalctl.out`](journalctl.out). Видны:

- успешная проверка конфигурации (`nginx: configuration file /etc/nginx/nginx.conf test is successful`);
- запуск сервиса `Started nginx.service - The nginx HTTP and reverse proxy server`;
- корректные stop/start циклы при перезагрузках конфигурации.

---

### Раздел 5. Управление доступом (15 баллов)

#### 5.1. Дискреционное управление доступом (DAC) — 5 баллов

**Создание групп:**

```bash
sudo groupadd -g 4444 curators   # кураторы, GID 4444
sudo groupadd -g 5500 mephi     # разработчики, общая группа
```

**Создание пользователей** (UID разработчиков строго по заданию — 5501/5502/5503):

```bash
sudo useradd -u 5501 -m -g mephi user1
sudo useradd -u 5502 -m -g mephi user2
sudo useradd -u 5503 -m -g mephi user3
sudo useradd -m -G curators curator1
sudo useradd -m -G curators curator2
```

См. [`passwd`](passwd) и [`group`](group):

```
user1:x:5501:5500::/home/user1:/bin/bash
user2:x:5502:5500::/home/user2:/bin/bash
user3:x:5503:5500::/home/user3:/bin/bash
curator1:x:5504:5504::/home/curator1:/sbin/nologin
curator2:x:5505:5505::/home/curator2:/sbin/nologin
curators:x:4444:curator1,curator2
mephi:x:5500:
```

**Права на директорию проекта `/data/mephi-2026`:**

```bash
sudo mkdir -p /data/mephi-2026
sudo chown root:mephi /data/mephi-2026
sudo chmod 2770 /data/mephi-2026            # setgid + rwxrws---
```

Бит **setgid** (символ `s` в `drwxrws---`) критически важен: новые файлы наследуют группу `mephi`, к которой принадлежат все три разработчика.

**POSIX ACL для кураторов (только чтение) и запрета others:**

```bash
sudo setfacl -m g:curators:rx /data/mephi-2026
sudo setfacl -d -m g:curators:rx /data/mephi-2026
sudo setfacl -d -m g::rwx /data/mephi-2026
sudo setfacl -d -m o::--- /data/mephi-2026
```

**umask 002 для разработчиков** (через `/etc/profile.d/`):

```bash
sudo tee /etc/profile.d/mephi-umask.sh >/dev/null <<'EOF'
if id -nG 2>/dev/null | tr ' ' '\n' | grep -qw mephi; then
    umask 002
fi
EOF
sudo chmod 644 /etc/profile.d/mephi-umask.sh
```

**Проверка** (из [`history.out`](history.out), команды 224–233):

```bash
# user1 создаёт файл
sudo -u user1 bash -c 'umask 002; echo "data from user1" > /data/mephi-2026/u1.txt'
# user2 дописывает (группа mephi имеет rw)
sudo -u user2 bash -c 'umask 002; echo "appended by user2" >> /data/mephi-2026/u1.txt'
# curator1 читает, но НЕ пишет
sudo -u curator1 cat /data/mephi-2026/u1.txt
sudo -u curator1 bash -c 'echo x >> /data/mephi-2026/u1.txt' || echo "OK: curator CANNOT write"
# nobody — отказ
sudo -u nobody ls /data/mephi-2026 2>&1 || echo "OK: others denied"
```

Результат — в [`stat.out`](stat.out):

```
Доступ: (2770/drwxrws---)  Uid: (0/root)  Gid: (5500/mephi)
Контекст: unconfined_u:object_r:default_t:s0
```

#### 5.2. Привилегии tcpdump (снятие set-UID, замена на capabilities) — 5 баллов

```bash
TCPDUMP=$(command -v tcpdump)
sudo setcap cap_net_raw,cap_net_admin+ep "$TCPDUMP"
sudo chmod u-s "$TCPDUMP"          # снять set-UID
getcap /usr/sbin/tcpdump           # → cap_net_admin,cap_net_raw=ep
```

Результат в [`getcap.out`](getcap.out):

```
/usr/sbin/tcpdump cap_net_admin,cap_net_raw=ep
```

Проверка запуска от обычного пользователя (команды 246–247 в `history.out`):

```bash
sudo -u user1 tcpdump -D         # список интерфейсов
sudo -u user1 tcpdump -c 5 -i any # захват 5 пакетов
```

#### 5.3. Мандатное управление доступом (MAC) — SELinux — 5 баллов

**Проверка режима:**

```bash
getenforce                       # → Enforcing
```

Содержимое [`getenforce.out`](getenforce.out): `Enforcing`.

**Назначение контекста `httpd_sys_content_t` для `/mephi-web`:**

```bash
sudo dnf install -y policycoreutils-python-utils
sudo semanage fcontext -a -t httpd_sys_content_t "/mephi-web(/.*)?"
sudo restorecon -Rv /mephi-web
```

Результат — в [`stat.out`](stat.out) для `/mephi-web`:

```
Доступ: (0755/drwxr-xr-x)  Uid: (0/root)  Gid: (0/root)
Контекст: system_u:object_r:httpd_sys_content_t:s0
```

**Перенастройка nginx на root `/mephi-web` и добавление `index index.html;`:**

```bash
# 1. Заменить root на /mephi-web
sudo sed -i 's|root[[:space:]]\+/usr/share/nginx/html;|root         /mephi-web;|' /etc/nginx/nginx.conf
# 2. Добавить директиву index после строки root
sudo sed -i '/root[[:space:]]\+\/mephi-web;/a\    index index.html;' /etc/nginx/nginx.conf
# 3. Проверить конфигурацию и перезапустить
sudo nginx -t && sudo systemctl restart nginx
```

---

### Раздел 6. Аутентификация (5 баллов)

**6.1. Ограничение входа для кураторов** (через `/sbin/nologin` — см. [`passwd`](passwd)):

```bash
sudo usermod -s /sbin/nologin curator1
sudo usermod -s /sbin/nologin curator2
```

```
curator1:x:5504:5504::/home/curator1:/sbin/nologin
curator2:x:5505:5505::/home/curator2:/sbin/nologin
```

Кураторы не могут войти локально через консоль, SSH или `su -`; при этом `sudo -u curator1 ...` продолжает работать (использовалось в проверках Раздела 5.1).

**6.2. Управление паролями.** Срок действия — 90 дней:

```bash
sudo sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   90/' /etc/login.defs
for u in user1 user2 user3 curator1 curator2; do
    sudo chage -M 90 "$u"
done
```

Минимальная длина — 12 символов ([`pwquality.conf`](pwquality.conf)):

```bash
sudo dnf install -y libpwquality
echo "minlen = 12" | sudo tee -a /etc/security/pwquality.conf
```

Файл [`pwquality.conf`](pwquality.conf) заканчивается строкой `minlen = 12` (остальные параметры — закомментированные значения по умолчанию).

---

### Раздел 7. Тестирование (5 баллов)

**7.1. Создание web-страницы** с номером зачётной книжки:

```bash
sudo tee /mephi-web/index.html >/dev/null <<'EOF'
Hello from Student: 374352
EOF
sudo chmod 644 /mephi-web/index.html
sudo chown root:root /mephi-web/index.html
sudo restorecon -v /mephi-web/index.html
```

**7.2. Проверка доступности** — содержимое [`curl.out`](curl.out):

```
Hello from Student: 374352
```

Команда проверки:

```bash
curl http://localhost/
# → Hello from Student: 374352
```

Визуальное подтверждение — [`mephi-screenshot.png`](mephi-screenshot.png): скриншот графического браузера на хост-системе виртуальной машины, на котором чётко виден текст «Hello from Student: 374352».

---

## ✅ Контрольный список перед сдачей

| Пункт        | Команда проверки                           | Ожидаемый результат            | Артефакт                           |
| ------------ | ------------------------------------------ | ------------------------------ | ---------------------------------- |
| 1.1 DHCP     | `nmcli -t -f ipv4.method connection show`  | `auto`                         | —                                  |
| 1.2 Hostname | `hostname -f`                              | `mephi-2026.domain.local`      | —                                  |
| 1.3 Ping     | `cat ping.out`                             | 0% packet loss                 | `ping.out`                         |
| 2.1 Update   | `dnf history`                              | транзакция с I/O/U             | `dnf.out`                          |
| 2.2 Packages | `rpm -q nginx libcap-ng-utils libcap`      | три версии                     | `dnf.out`                          |
| 2.3 tcpdump  | `rpm -q tcpdump`                           | версия                         | `dnf.out`                          |
| 3.1 FS       | `findfs LABEL=MEPHI_WEB`                   | `/dev/sdb1`                    | `fstab`                            |
| 3.2 Mount    | `findmnt /mephi-web`                       | `LABEL=MEPHI_WEB`              | `fstab`                            |
| 4.1 nginx    | `systemctl is-enabled nginx`               | `enabled`                      | `journalctl.out`                   |
| 4.2 Journal  | `journalctl -u nginx -b`                   | `Started nginx.service`        | `journalctl.out`                   |
| 5.1 DAC      | `stat /data/mephi-2026`                    | `2770/drwxrws---` root:mephi   | `stat.out`                         |
| 5.2 cap      | `getcap /usr/sbin/tcpdump`                 | `cap_net_admin,cap_net_raw=ep` | `getcap.out`                       |
| 5.3 MAC      | `getenforce`                               | `Enforcing`                    | `getenforce.out`                   |
| 5.3 ctx      | `stat /mephi-web`                          | `httpd_sys_content_t`          | `stat.out`                         |
| 6.1 shell    | `getent passwd curator1`                   | `...:/sbin/nologin`            | `passwd`                           |
| 6.2 age      | `grep PASS_MAX_DAYS /etc/login.defs`       | `90`                           | `shadow`                           |
| 6.2 len      | `grep minlen /etc/security/pwquality.conf` | `12`                           | `pwquality.conf`                   |
| 7.1 page     | `cat /mephi-web/index.html`                | `Hello from Student: 374352`   | —                                  |
| 7.2 curl     | `curl http://localhost/`                   | `Hello from Student: 374352`   | `curl.out`, `mephi-screenshot.png` |

---

## 🔐 Примечание о безопасности

Файл [`shadow`](shadow) содержит `/etc/shadow` с **затёртыми хешами паролей** (поле пароля заменено на `!`) и сменённым владельцем. Подготовка:

```bash
sudo sed -i -E 's/^(user1|user2|user3|curator1|curator2|dmitry|root):[^:]*:/\1:!:/' ./shadow
sudo chown dmitry:dmitry ./shadow
```

Это соответствует требованиям задания 8.1 (загрузить `/etc/shadow` под именем `shadow`), но снижает риск утечки учётных данных.

---

_Проект выполнен на виртуальной машине с РЕД ОС 8. Все настройки сохраняются после перезагрузки. Бухгалтерский шифр студента — `374352`._
