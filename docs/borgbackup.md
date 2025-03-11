# Borg

*[[apps|<- Назад]]*

*[[index|<- На главную]]*
***
## Основы Borg

`Borg` - система резервного копирования написанная на Python (требовательный к производительности код: сжатие, шифрование и т.д., реализован на C), работающая через SSH.

[Документация](https://borgbackup.readthedocs.io/en/stable/index.html)

*Установка:*

```bash
# Debian & Ubuntu
apt install borgbackup

# Fedora
dnf install borgbackup

# Arch
pacman -S borg
```

*Основные команды:*

```bash
# Вывести справочную информацию
borg help
# Вывести справочную информацию по команде create
borg help create

# Инициализация нового репозитория TestRepo
# флаг -e для шифрования репозитория паролем
borg init -e none borg@192.168.0.10:TestRepo

# Создание бэкапа TestBackUp в репозитории TestRepo с содержимым папки /home
borg create borg@192.168.0.10:TestRepo::TestBackUp /home

# Вывести информацию о репозитории TestRepo
borg info borg@192.168.0.10:TestRepo
# Вывести информацию о бэкапе TestBackUp из репозитория TestRepo
borg info borg@192.168.0.10:TestRepo::TestBackUp

# Вывести список бэкапов из репозитория TestRepo
borg list borg@192.168.0.10:TestRepo
# Вывести список файлов из бэкапа TestBackUp репозитория TestRepo
borg list borg@192.168.0.10:TestRepo::TestBackUp

# Проверка репозитория TestRepo на целостность
# флаг -v для вывода информации в терминал
borg check -v borg@192.168.0.10:TestRepo
# Проверка на целостность бэкапа TestBackUp из репозитория TestRepo
borg check -v borg@192.168.0.10:TestRepo::TestBackUp

# Удалить все бэкапы кроме последних двух из репозитория TestRepo
# флаг --list для вывода информации
# флаг --keep-last для указания сколько последних копий оставить
# флаг --dry-run служит для вывода информации об удаляемых бэкапах
# без их удаления (чтобы удалить полноценно, запустить без флага)
borg prune --list --keep-last 2 --dry-run borg@192.168.0.10:TestRepo
# Удалить все бэкапы кроме тех, что были сделаны за последние 2 часа
# флаг --keep-within для указания различных интервалов: часы, дни и т.д.
borg prune --list --keep-within 2H borg@192.168.0.10:TestRepo
# Удалить все бэкапы кроме последнего бэкапа в каждом из 7 дней
# (дни без бэкапов не учитываются)
# флаг --keep-daily для указания количества дневных бэкапов
# существуют также флаги для недельных, месячных и т.д.
borg prune --list --keep-daily 7 borg@192.168.0.10:TestRepo
# После удаления ненужных бэкапов, нужно обязательно применинить
# compact, иначе дисковое пространство не освободится
borg compact -v borg@192.168.0.10:TestRepo

# Показать разницу между бэкапом TestBackUp и TestBackUp2
borg diff borg@192.168.0.10:TestRepo::TestBackUp TestBackUp2

# Примонтировать репозиторий TestRepo к папке /mnt/borg
borg mount borg@192.168.0.10:TestRepo /mnt/borg
# Примонтировать бэкап TestBackUp к папке /mnt/borg
borg mount borg@192.168.0.10:TestRepo::TestBackUp /mnt/borg
# Размонтировать директорию или бэкап из папки /mnt/borg
borg umount /mnt/borg

# Извлечь файл backup.sql из бэкапа TestBackUp
borg extract borg@192.168.0.10:TestRepo::TestBackUp backup.sql

# Переименовать бэкап TestBackUp в BackUp
borg rename borg@192.168.0.10:TestRepo::TestBackUp BackUp

# Удаление бэкапа BackUp
borg delete borg@192.168.0.10:TestRepo::BackUp
```

> В скриптах можно использовать переменные для *borg*, в которых можно указать репозиторий или ключ от репозитория (если было включено шифрование). Полный список можно посмотреть на [странице](https://borgbackup.readthedocs.io/en/stable/usage/general.html#environment-variables) документации.
> 
> ```bash
> export BORG_REPO='ssh://borg@192.168.1.10/./TestRepo'
> export BORG_PASSPHRASE='test'
> 
> borg create ::TestBackUp2 /home
> borg list
> borg list ::TestBackUp2
> ```

***
## Базовая настройка сервера и клиента

- Устанавливаем *borg* на сервере и клиенте (через репозиторий дистрибутива или со [страницы](https://github.com/borgbackup/borg) официального репозитория на GitHub).
- 
- На сервере создаем пользователя для *borg* (необходимо создать пользователя с домашней директорией).

```bash
useradd -m borg
```

- На клиенте генерируем SSH-ключ.

- На сервере, созданному пользователю, добавляем сгенерированный ключ.

```bash
mkdir ~borg/.ssh
echo 'command="/usr/bin/borg serve" ssh-rsa ...' > ~borg/.ssh/authorized_keys
chown -R borg:borg ~borg/.ssh
```

- Инициализируем *borg* репозиторий на бэкап-сервере с клиента.

```bash
borg init -e none borg@192.168.0.10:TestRepo
```

- Запускаем первый бэкап для проверки.

```bash
borg create --stats --list --progress borg@192.168.0.10:TestRepo::'Backup_{hostname}_{now:%Y%m%d_%H%M%S}' /etc
# Флаги --stats и --list выводят статистику по бэкапу и попавшим в него файлам
# --progress отображает прогресс выполнения
```

> Borg может принимать файлы из стандартного ввода, так же можно заменить `borg@192.168.0.10` на алиас, используя `.ssh/config`.
> Пример с дампом базы данных mariadb:
> ```bash
> mariadb-dump --all-databases | borg create --compression zlib --stdin-name backup.sql backup_server:TestRepo::'Backup_{hostname}_{now:%Y%m%d_%H%M%S}' -
> ```

- Проверяем список архивов внутри репозитория.

```bash
borg list borg@192.168.0.10:TestRepo
```

***
## Скрипт для автоматического бэкапа

> Данный скрипт взят со [страницы](https://borgbackup.readthedocs.io/en/stable/quickstart.html#automating-backups) официальной документации.

```bash
#!/bin/sh

# Устанавливаем репозиторий, чтобы не прописывать его в командах
export BORG_REPO='ssh://borg@192.168.1.10/./TestRepo'

# Указываем пароль, если репозиторий инициализировался с шифрованием
export BORG_PASSPHRASE='test'

# Помошники и обработка ошибок
info() { printf "\n%s %s\n\n" "$( date )" "$*" >&2; }
trap 'echo $( date ) Backup interrupted >&2; exit 2' INT TERM

info "Starting backup"

# Создаем бэкап (подробно про ключи можно прочитать - borg help create)

borg create                         \
    --verbose                       \
    --filter AME                    \
    --list                          \
    --stats                         \
    --show-rc                       \
    --compression zlib              \
    --exclude-caches                \
    --exclude 'home/*/.cache/*'     \
                                    \
    ::'{hostname}-{now}'            \
    /home

backup_exit=$?

info "Pruning repository"

# Используем prune с флагами, чтобы сохранить 7 дневных, 4 недельных
# и 6 месячных бэкапов для этой машины. Флаг --glob-archive с указанием
# '{hostname}-*', очень важен для очистки бэкапов именно этой машины,
# иначе будут удалены бэкапы с других серверов в этом репозитории.

borg prune                          \
    --list                          \
    --glob-archives '{hostname}-*'  \
    --show-rc                       \
    --keep-daily    7               \
    --keep-weekly   4               \
    --keep-monthly  6

prune_exit=$?

# Актуализируем свободное пространство диска, путем уплотнения сегментов

info "Compacting repository"

borg compact

compact_exit=$?

# Используем самый большой код выхода, как глобальный код выхода
# (проверка на успешное выполнение операций)
global_exit=$(( backup_exit > prune_exit ? backup_exit : prune_exit ))
global_exit=$(( compact_exit > global_exit ? compact_exit : global_exit ))

if [ ${global_exit} -eq 0 ]; then
    info "Backup, Prune, and Compact finished successfully"
elif [ ${global_exit} -eq 1 ]; then
    info "Backup, Prune, and/or Compact finished with warnings"
else
    info "Backup, Prune, and/or Compact finished with errors"
fi

exit ${global_exit}
```

***