# Отказоустойчивый кластер при помощи Corosync + Pacemaker

*[[linux|<- Назад]]*

*[[index|<- На главную]]*
***
## Общая информация

`Corosync` - программный продукт, который позволяет создавать *единый кластер* из нескольких аппаратных или виртуальных серверов. Corosync *отслеживает и передает* состояние всех участников (нод) в кластере.

*Достоинства*:

- Позволяет *мониторить статус* приложений;
- Оповещает приложения о *смене активной ноды* в кластере;
- *Отправляет* идентичные сообщения процессам на всех нодах;
- *Предоставляет доступ* к общей базе данных с конфигурацией и статистикой;
- Отправляет *уведомления* об изменениях, произведенных в базе.

`Pacemaker` - *менеджер ресурсов* кластера. Он позволяет *использовать* службы и объекты в рамках одного кластера из двух и более нод.

[Документация](https://clusterlabs.org/pacemaker/doc/)

Достоинства:

- Позволяет находить и устранять *сбои* на уровне нод и служб;
- Не зависит от *подсистемы хранения* (не нужен общий накопитель);
- Не зависит от *типов ресурсов* (все, что можно прописать в скрипты, можно кластеризовать);
- Поддерживает *STONITH* (Shoot The Other Node In The Head), то есть умершая нода изолируется и запросы к ней не поступают, пока нода не отправит сообщение о том, что она снова в рабочем состоянии;
- Поддерживает *кворумные и ресурсозависимые* кластеры любого размера;
- Поддерживает практически любую *избыточную конфигурацию*;
- Может автоматически *реплицировать конфиг* на все узлы кластера (не придется править все вручную);
- Можно задать *порядок запуска ресурсов*, а также их *совместимость* на одном узле;
- Поддерживает расширенные типы ресурсов;
- Имеет единую кластерную оболочку CRM с поддержкой скриптов.

> Идея создания *отказоустойчивого кластера* заключается в том, что с помощью Corosync строится кластер, а следить за его состоянием будет Pacemaker.

**Структура Pacemaker**

Первая важная сущность - это *узлы кластера*. Узел (*нода*) кластера представляет собой физический сервер или виртуальную машину с установленным Pacemaker. Узлы, предназначенные для предоставления одинаковых сервисов, должны иметь одинаковую *конфигурацию софта*.

Вторая важная сущность - это *ресурсы кластера*. Для Pacemaker ресурс - это *скрипт*, написанный на любом языке (чаще всего на `bash`, но могут быть и другие языки, например `Python`). Скрипт управляет сервисами в операционной системе, он должен уметь выполнять 3 действия: `start`, `stop` и `monitor`, а так же делиться некоторой *метаинформацией*.

Ресурсы имеют множество *атрибутов*. Рассмотрим *наиболее интересные* из них:

- `priority` - *приоритет* ресурса, который учитывается, если узел исчерпал лимит по количеству активных ресурсов (по умолчанию 0). Если узлы кластера не одинаковы по производительности или доступности, то можно увеличить приоритет одного из узлов, чтобы он был активным всегда, когда работает.
- `resource-stickiness` - "*липкость*" ресурса (по умолчанию 0). "Липкость" указывает на то, насколько ресурс "хочет" оставаться там, где он есть сейчас. Другими словами, "липкость" показывает, насколько желательно или не желательно, чтобы ресурс вернулся на восстановленный после сбоя узел. Может быть любым числом указывающим "вес", который должен быть выше "веса" других операций (перенос на другой узел и т.д.), чтобы ресурс оставался на данном узле. "Вес" подбирается опытным путем.

> Поскольку по умолчанию липкость всех ресурсов 0, то Pacemaker сам располагает ресурсы на узлах "*оптимально*" на свое усмотрение.

- `migration-threshold` - *сколько отказов* должно произойти, чтобы Pacemaker решил, что узел непригоден для данного ресурса и перенес (*мигрировал*) его на другой узел. По умолчанию этот параметр также равен 0, т.е. при любом количестве отказов автоматического переноса ресурсов не будет происходить. Но, с точки зрения *отказоустойчивости*, правильно выставить этот параметр в 1, чтобы при первом же сбое Pacemaker перемещал ресурс на другой узел.
- `failure-timeout` - количество *секунд после сбоя*, до истечения которых Pacemaker считает, что отказа не произошло, и не предпринимает ничего, в частности, не перемещает ресурсы. По умолчанию, равен 0.
- `multiple-active` - дает указание Pacemaker, что делать с ресурсом, если он оказался запущен *более чем на одном* узле. Может принимать следующие значения:
	- `block` - установить ресурсам опцию `unmanaged`, то есть деактивировать
	- `stop_only` - остановить ресурсы на всех узлах и оставить их в таком состоянии
	- `stop_start` - остановить ресурсы на всех узлах и запустить только на одном (по умолчанию)
	- `stop_unexpected` - остановить все активные ресурсы, кроме тех, где ресурс должен быть активным (начиная с версии 2.1.3)

> По умолчанию, кластер не следит после запуска, жив ли ресурс. Чтобы включить отслеживание, нужно при создании ресурса добавить операцию `monitor`, тогда кластер будет следить за состоянием ресурса. Параметр `interval` этой операции - это интервал, с которым необходимо делать проверку.

**Типы сетей используемых в Corosync**

`KNET (Kernel Network)` - это один из *транспортных протоколов* используемых в Corosync для обеспечения связи между узлами кластера. Он основан на механизмах *ядра Linux* и предназначен для обеспечения высокой производительности и надежности в кластерных средах.

`UDPU (Unicast Datagram Protocol)` - это другой *транспортный протокол*, который также используется в Corosync. Он основан на использовании *UDP* для передачи сообщений и может быть более простым в настройке, особенно в средах, где не требуется высокая производительность или сложные сетевые топологии.

> *Конфиг corosync* лежит по пути `/etc/corosync/corosync.conf`.

***
## Создание и настройка кластера

**Кластер из трех нод. Две с nginx, 3-я нода "наблюдатель".**

- Прописываем *адреса* нод в `/etc/hosts` на всех нодах:

```
192.168.1.10 node1
192.168.1.11 node2
192.168.1.12 node3
```

> Некоторые файлы `/etc/hosts` *перезаписываются из шаблона* при перезагрузке, особенно на облачных решениях. Соответственно нужно *редактировать шаблон*, путь до него обычно указан в самом `/etc/hosts`.

- Устанавливаем *основные пакеты* на всех нодах (данную команду нужно исполнять с правами рута, но если использовать `sudo`, то скрипт установщика отработает неверно и не создастся пользователь `hacluster`):

```bash
apt install pacemaker pcs corosync resource-agents
```

- Включаем *автозагрузку* сервисов на всех нодах:

```bash
systemctl enable pcsd corosync pacemaker
```

> Для *управления кластером* используется утилита `pcs`. При установке Pacemaker *автоматически* будет создан пользователь `hacluster`. Для использования `pcs`, а также для доступа в веб-интерфейс нужно задать пароль пользователю `hacluster`

- Меняем *пароль* пользователю `hacluster` на всех нодах:

```bash
passwd hacluster
```

- Запускаем *сервис* на всех нодах:

```bash
systemctl start pcsd
```

> Далее все действия выполняются уже на *одной ноде*.

- Настраиваем *аутентификацию*:

```bash
pcs host auth node1 node2 node3 -u hacluster -p password
```

- Создаем *кластер* (сразу добавляем в автозагрузку и включаем):

```bash
pcs cluster setup --force --enable --start my_cluster node1 node2 node3
```

- Проверяем *статус* кластера:

```bash
pcs status
pcs status corosync
```

- Выключаем *STONITH*:

```bash
pcs property set stonith-enabled=false
```

- Если используется всего 2 ноды, то при сбое на одном из узлов *кворума* не будет и Pacemaker остановит ресурсы, чтобы этого не произошло отключаем эту политику (только для 2-х нод!!!):

```bash
pcs property set no-quorum-policy=ignore
```

- Настройки можно проверить командой:

```bash
pcs property
```

- Создаем ресурс, а именно виртуальный IP-адрес:

```
pcs resource create my_vip ocf:heartbeat:IPaddr2 ip=192.168.1.100 cidr_netmask=24 op monitor interval=30s
```

- Проверяем:

```bash
pcs resource
ip a
```

- Добавляем ресурс nginx (*для примера*):

```bash
# Убираем службу из автозапуска (необходимо выполнить на всех нодах)
systemctl disable --now nginx
# Добавляем ресурс, который будет проверяться каждые 30с
# и в случае минутного тайм-аута ресурс будет включен на другой ноде
pcs resource create my_nginx ocf:heartbeat:nginx op monitor interval=30s timeout=60s
```

- Создаем группу ресурсов, в которую сложим все созданные ресурсы (чтобы при сбое одного из ресурсов, вся группа переместилась на другой узел):

```bash
pcs resource group add my_group my_vip my_nginx
```

- Создаем ограничение локации для необходимых узлов:

```bash
# 2 основные ноды делаем равнозначными
pcs constraint location my_group prefers node1 node2
# Ноду наюлюдатель изолируем от миграции ресурсов
pcs constraint location my_group avoids node3
```

- *Перезагружаем* службу corosync на всех нодах (pacemaker перезагрузится автоматически):

```bash
service corosync restart
```

**Дополнительные команды**

- Вывести больше *информации* о кластере:

```bash
pcs config show
```

- Присвоение *мета информации* ресурсу:

```bash
pcs resource meta my_group resource-stickiness=1
```

- Проверка *конфигурации* кластера:

```bash
pcs cluster verify --full
```

- Если требуется сменить рабочую ноду руками (например для *технических работ*), то переносим ресурсы следующей командой:

```bash
pcs resource move my_group node2
```

> При переносе, создается *ограничение* по локации в `constraint`.

- Вывод списка существующих *ограничений*:

```bash
pcs constraint config
```

- Удаление *ограничения локации*:

```bash
pcs constraint location delete <constraint_id>
```

- Добавление *новой ноды* в кластер:

```bash
pcs host auth node3 -u hacluster -p password
pcs cluster node add --force node3
service corosync restart
```

- Очистка *счетчиков сбоев* (следует выполнять, когда мы устранили причину сбоя и хотим вернуть узел в состав кластера):

```bash
pcs resource cleanup
```

- Посмотреть *доступные ресурсы*, можно командами:

```bash
# Вывести доступные стандарты
pcs resource standards
# Вывести список ресурсов определенных стандартов
pcs resource agents ocf:heartbeat
pcs resource agents systemd
# Вывести все ресурсы доступные в системе
pcs resource list
```

- Посмотреть *описание* определенного ресурса:

```
pcs resource describe ocf:heartbeat:IPaddr2
```

- Отображение текущего "*веса*" размещения:

```bash
crm_simulate -sL
```

- *Включение/отключение* ноды для тестов:

```bash
pcs node standby node1
pcs node unstandby node1
```

***