# Настройка и работа с сетью в Linux

*[[linux|<- Назад]]*

*[[index|<- На главную]]*
***

***
## Создание и настройка моста

*Если отсутствуют необходимые утилиты, необходимо установить следующий пакет:*

```bash
apt install bridge-utils
```

**Создание моста**

- *Создаем* мост

```bash
brctl addbr mybr
```

- *Назначаем* мосту ip-адрес

```bash
ip addr add 192.168.1.1/24 brd 192.168.1.255 dev mybr
```

- *Включаем* интерфейс моста

```bash
ip link set mybr up
```

- *Проверяем* созданное устройство

```bash
ip a
```

**Другие полезные команды brctl**

- Вывести все мосты

```bash
brctl show
```

- Показать мак-адреса подключенные к мосту

```bash
brctl showmacs brname
```

- Удаляем мост (сперва его необходимо выключить `ip link set brname down`)

```bash
brctl delbr brname
```

- Добавить интерфейс к мосту

```bash
brctl addif brname interface_name
```

- Отключить интерфейс от моста

```bash
brctl delif brname interface_name
```

***