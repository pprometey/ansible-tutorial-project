# Шаги по изучению Ansibled

Ansible - is Automation Configuration Management Tools (ACMT) - Инструмент автоматизации управления конфигурацией.

Бывает 2 вида ACMT:

- Pull - на управляемых машинах устанавливается Agent который забирает настройки от Master (Chef, puppet, slatstack)
- Push - на управляемых не надо устанавливать агента, Master делает установку настроек удаленно. (Ansible)

## Минимальные требования

**Сервер управления (Master сервер):**

- Только Linux
- Python версии 2.6 и выше или Python 3.5 и выше

**Управляемые серверы (Managed серверы):**

- Linux: имя пользователя и пароль администратора или SSH-ключ, Python 2.6 и выше
- Windows: имя пользователя и пароль администратора, PowerShell версии 3.0 и выполнение скрипта /ConfigureRemotingForAnsible.ps1

## Установка Ansible

Инструкция по установке с официального сайта: https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

### Debian

```
Sudo install gpupg
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
echo "deb [http://ppa.launchpad.net/ansible/ansible/ubuntu](http://ppa.launchpad.net/ansible/ansible/ubuntu) trusty main" | sudo tee -a /etc/apt/sources.list
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 93C4A3FD7BB9C367
sudo apt update
sudo apt install ansible
```

## Шаг 1. Подключение к клиентам и инвентори-файл

В первую очередь необходимо создать файл инвентаризации (inventory file) - это файл, в котором описываются целевые хосты и группы хостов для управления ими.

Какие имена он может иметь:  

- По умолчанию в Ansible:  
  - /etc/ansible/hosts - системный дефолтный инвентори-файл  
  - Можно использовать любой другой файл, указав его в параметре -i (inventory) при запуске ansible/ansible-playbook.  
- Варианты имён:
  - hosts - часто называют просто так
  - inventory - часто встречается
  - production / staging / dev - имена для разных окружений
  - inventory.ini - когда файл в INI-формате
  - inventory.yaml или inventory.yml - если инвентори в YAML-формате
  - inventory.json - в JSON-формате
- Типы инвентарей:
  - Статический инвентори: обычный файл (ini, yaml, json)
  - Динамический инвентори: исполняемый скрипт или плагин, который на лету возвращает список хостов

В корне проекта создаем файл `hosts`, и перечислим имена хостов, которыми будет управлять, и установим переменные для подключения к ним по ssh:

```ini
[minions]
minion1 anisble_username=developer ansible_ssh_private_key_file=/home/developer/.ssh/id_rsa ansible_python_interpreter=/usr/bin/python3
minion2 anisble_username=developer ansible_ssh_private_key_file=/home/developer/.ssh/id_rsa ansible_python_interpreter=/usr/bin/python3
```

Теперь можжно выполнить пинг сервера:

```bash
ansible -i hosts all -m ping 
```

где параметры:

- \-`i` указывем расположение файла инвентаризации (`all` - применить ко всем группам)
- \-`m` - какой модуль применяем (`ping` в данном случае).

результат пинга:

```bash
minion2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
minion1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Во время первого подключюения будет сделан запрос на добавления хоста в known_hosts - это механизм безопасности SSH.

```
The authenticity of host 'minion2 (172.18.0.2)' can't be established.
ED25519 key fingerprint is SHA256:WW96QnhRxdH6bes4Jb7XizPiJxmICoQUu/dDokoRm/M.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? 
```

Что это значит?

- При первом подключении к новому серверу по SSH клиент не знает его ключа (уникального идентификатора).
- SSH показывает отпечаток (fingerprint) этого ключа и спрашивает, доверяете ли вы этому серверу.
- Если вы соглашаетесь, ключ сервера сохраняется в файл ~/.ssh/known_hosts на вашем компьютере.
- При следующих подключениях SSH проверяет, что ключ сервера совпадает с сохранённым - чтобы убедиться, что вы подключаетесь именно к тому же серверу, а не к злоумышленнику (защита от MITM-атак).

Мы можем настроить чтобы ансибл автоматически принимал ключи новых хостов, но в производственной среде так делать крайне не рекомендуется. 

Для этого создадим файл конфигурации для ансибл `ansible.cfg` в корне проекта с содержимым:

```ini
[defaults]
host_key_checking = false 
inventory         = ./hosts #
```

- `host_key_checking = false` - автоматически приняите ключей для новых хостов,
- `inventory = ./hosts` - путь к файлу инвентаризации и теперь при выполнении команд ансибл, не обязательно указывать каждый раз путь к файлу инвентаризации, как в этом примере `ansible all -m ping`

### Правила создания файла инвентаризациии

В минимальном исполнении достаточно просто перечислить IP хостов

```ini
10.0.15.12
10.0.15.13
```

Можно дать хосту алиас

```ini
ansible-debian ansible_host=10.0.15.12
```

Можно включить сервер в группу, для этого название группы надо обрамить квадратными скобками

```ini
[group_name]
hostname ansible_host=10.0.15.12
```

В команде ansible по умолчанию доступны изначально два параметра, для указания к каким хостам применять инструкции:

-  **`all`** - в которую включены все хосты перечисленные в файле инвентаризации
- **`ungrouped`** - хосты которые не включены ни в одну группу

Чтобы посмотреть список групп в файле инвентаризации нужно выполнить команду `ansible-inventory --list`, которая покажет все группы и переменные доступные этим группам, либо команду `ansible-inventory --graph`:

```bash
@all:
  |--@ungrouped:
  |  |--10.0.15.11
  |--@cluster_servers:
  |  |--@master_servers:
  |  |  |--control1
  |  |--@worker_servers:
  |  |  |--worker1
  |  |  |--worker2
```

Для создания родительской группы, которая будет содержать в себе названия других групп надо после названия группы указать `:children`

```ini
10.0.15.11

[master_servers]
control1 ansible_host=10.0.15.12

[worker_servers]
worker1 ansible_host=10.0.15.13
worker2 ansible_host=10.0.15.14

[cluster_servers:children]
master_servers
worker_servers
```

Родительские группы можно включать как элементы в другие родительские группы, например

```ini
[cluster_servers:children]
master_servers
worker_servers

[all_cluster_servers:children]
cluster_servers
cluster2_servers
```

Для групп можно указать общие переменные, которые актуальны для всех хостов в этой группе. Для этого надо добавить новый новую секцию конфигурации, с названием группы, после после которого указать постфикс`:vars`, например:

```ini
[cluster_servers:children]
master_servers
worker_servers

[cluster_servers:vars]
ansible_user=developer
ansible_ssh_private_key_file=/home/developer/.ssh/id_rsa
```

В итоге мы можем привести наш файл hosts к такому виду

```ini
[master_servers]
minion1

[worker_servers]
minion2

[cluster_servers:children]
master_servers
worker_servers

[cluster_servers:vars]
anisble_username=developer 
ansible_ssh_private_key_file=/home/developer/.ssh/id_rsa
ansible_python_interpreter=/usr/bin/python3
```

## Шаг 2. Выполнение ad-hoc команд и основные модули ansible

ad-hoc команды, это команды которые выполняются напрямую, как параметры ansible в командной строке. Любые действия ансибл выполянет посредством модулей. 

Простейший пример `ansible all -m ping`, где мы говорим ансибл выполнить на всех серверах в указанных в инвентори файле модуль ping, котоырй выполнит соответствущу команду.

для получения служебной информации о выполнении команд можно использовать параметр `-v`, либо `-vv`, либо `-vvv`, и в каждом случае будет выводиться все более подробная служебная информация о процессе выполнении ансибл команд, пример `ansible cluster_servers -m shell -a "ls -lpa /home/developer" -vvv`

Для получения списка команд и модулей, которые поддерживает ансибл, можно использовать команду `ansible-doc -l`

### Основные модули ansible

#### setup

Модуль `setup` - `ansible master_servers -m setup` выдает много информации о хосте и его настройках. 

#### shell

Модуль `shell` позволяет выполнить любую команду оболочки,

```ini
ansible all -m shell -a "uptime"

minion2 | CHANGED | rc=0 >>
 13:35:43 up 32 min,  0 users,  load average: 0.02, 0.07, 0.09
minion1 | CHANGED | rc=0 >>
 13:35:43 up 32 min,  0 users,  load average: 0.02, 0.07, 0.09
```

#### shell, command

Можно выполнить любу команду оболочки чрез модуль `shell`, например `ansible all -m shell -a "ls /home/developer"`
Есть еще модуль `command`, он тоже может запускать команды, но не через оболочку, поэтому недоступны переменные окружения оболочки, перенаправления потоков и т.д. (`"*"`, `"<"`, `">"`, `"|"`, `";"` и `"&"`). Пример выполнения через `command`  `ansible all -m command -a "ls /home/developer"`

#### copy

Модуль `copy` позволяет копировать файлы на удаленные машины. Предварительно необходимо создать в корневой папке проекта файл hi.txt, который мы будем копировать на другие машины (с любым содержимым внутри).

```bash
ansible all -m copy -a "src=hi.txt dest=/home/developer mode=777"

minion1 | CHANGED => {
    "changed": true,
    "checksum": "aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d",
    "dest": "/home/developer/hi.txt",
    "gid": 999,
    "group": "developer",
    "md5sum": "5d41402abc4b2a76b9719d911017c592",
    "mode": "0777",
    "owner": "developer",
    "size": 5,
    "src": "/home/developer/.ansible/tmp/ansible-tmp-1751377524.576038-3751-160554860394367/.source.txt",
    "state": "file",
    "uid": 999
}
minion2 | CHANGED => {
    "changed": true,
    "checksum": "aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d",
    "dest": "/home/developer/hi.txt",
    "gid": 999,
    "group": "developer",
    "md5sum": "5d41402abc4b2a76b9719d911017c592",
    "mode": "0777",
    "owner": "developer",
    "size": 5,
    "src": "/home/developer/.ansible/tmp/ansible-tmp-1751377524.5859642-3752-264340817273336/.source.txt",
    "state": "file",
    "uid": 999
}

```

Если мы хотим чтобы команда вызвалась от имени sudo, то надо к команде добавить параметр `-b`

```bash
ansible all -m copy -a "src=hi.txt dest=/home mode=777" -b

minion1 | CHANGED => {
    "changed": true,
    "checksum": "aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d",
    "dest": "/home/hi.txt",
    "gid": 0,
    "group": "root",
    "md5sum": "5d41402abc4b2a76b9719d911017c592",
    "mode": "0777",
    "owner": "root",
    "size": 5,
    "src": "/home/developer/.ansible/tmp/ansible-tmp-1751377557.9988103-4608-156469420249661/.source.txt",
    "state": "file",
    "uid": 0
}
minion2 | CHANGED => {
    "changed": true,
    "checksum": "aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d",
    "dest": "/home/hi.txt",
    "gid": 0,
    "group": "root",
    "md5sum": "5d41402abc4b2a76b9719d911017c592",
    "mode": "0777",
    "owner": "root",
    "size": 5,
    "src": "/home/developer/.ansible/tmp/ansible-tmp-1751377558.0059295-4609-255462204213465/.source.txt",
    "state": "file",
    "uid": 0
}
```

#### file

Для удаления файла можно использовать модуль `file`, который служит для совершения операций над файлами

```bash
ansible all -m file -a "path=/home/hi.txt state=absent" -b

minion1 | CHANGED => {
    "changed": true,
    "path": "/home/hi.txt",
    "state": "absent"
}
minion2 | CHANGED => {
    "changed": true,
    "path": "/home/hi.txt",
    "state": "absent"
}
```

Можно повторно запустить эту же команду, и она будет успешно исполнятся, если допустим копируем файл, и такой файл есть, команда выполниться, но ансибл скажет что изменений не было, например запустим повторно предыдущую команду

```bash
ansible all -m file -a "path=/home/hi.txt state=absent" -b

minion1 | SUCCESS => {
    "changed": false,
    "path": "/home/hi.txt",
    "state": "absent"
}
minion2 | SUCCESS => {
    "changed": false,
    "path": "/home/hi.txt",
    "state": "abssent"
}
```

Мы видим что команда успешно выполнилась, но "changed": false, что говорит нам, что изменения не были применены. 

#### get_url

Для скачивания файлов с интернета можно использовать модуль get_url

```bash
ansible all -m get_url -a "url=https://raw.githubusercontent.com/pprometey/ansible-learning-template/refs/heads/main/README.md dest=/home/developer"
```

#### yum, apt

Для установки и удалении приложений на redhat подобные системы, можно использовать модуль yum

- Установить приложение stress `ansible all -m yum -a "name=stress state=latest" -b`
- Удалить `ansible all -m yum -a "name=stress state=removed" -b`
Для debian подобных систем
- Установить приложение stress  `ansible all -m apt -a "name=stress state=latest" -b`
- Удалить приложение - `ansible all -m apt -a "name=stress state=removed" -b`

#### uri

Модуль для выполнения запросов по http и https. `ansible all -m uri -a "url=https://iit.kz"`, для получения содержимого, нужно указать `return_conent=yes` пример:`ansible all -m uri -a "url=https://iit.kz return_content=yes"`

#### service

Модуль для управления сервисами, например запустить веб-сервер apache и чтобы он автоматически загружался при запуске (при условии что он был ранее установлен) `ansible all -m service -a "name=httpd state=started enabled=yes" -b`, или удалить этот сервис `ansible all -m service -a "name=httpd state=removed" -b

## Шаг 3. Yaml, group_vars, host_vars

### Основы Yaml

YAML - это текстовый формат для хранения данных в структурированном виде. Файлы в формате yaml имеюют расширение либо `.yaml`, либо `.yml`.

Правила оформления YAML

1. Используй отступы для вложенности. Главное - одинаковое количество пробелов на одном уровне.
2. Списки начинаются с `-` и пробела.
3. Пары пишутся в виде `ключ: значение`, обязательно с двоеточием и пробелом.
4. Имя можно указывать как поле (`name: Alexey`) или как ключ (`Alexey:`).  
    Если имя - это ключ, после него обязательно должно быть двоеточие.
5. Можно использовать краткую запись с фигурными скобками:  
    `{name: ..., age: ..., skills: [...]}` или `Имя {age: ..., skills: [...]}`.
6. Для краткой записи списков можно использовать квадратные скобки:  `skills: [Python, C#, YAML]` . Это эквивалентно многострочной записи через `-`.
7. Если в значении есть двоеточие (`:`), ставь кавычки (двойные или одинарные):  `nikname: "Anton: god"` - иначе будет ошибка.

Пример, все 4 записи одной и той же информации валидны. 

```yaml
---
- name: Alexey
  nikname: Alex
  age: 35
  skills:
    - Python
    - C#
    - Yaml

- Anton:
    nikname: "Anton: thebest"
    age: 30
    skills:
      - Продажи
      - Маркетинг

- {name: Sasha, age: 60, skills: ['FoxPro', 'Работа по дереву']}

- Nina {age: 56, skills: ['Готовка', 'Садоводство: Кабачки']}
...
```

Одинарные кавычки `'...'`

- Всё внутри воспринимается буквально - никаких спецсимволов или экранирования.
- Чтобы вставить саму одинарную кавычку, её нужно удвоить:  
    Пример: `'Это ''пример'' текста'` -> `Это 'пример' текста`.
- Нельзя делать переносы строк внутри одинарных кавычек.
Двойные кавычки `"..."`
- Позволяют использовать escape-последовательности (как в языках программирования):
  - `\n` - перенос строки
  - `\t` - табуляция
  - `\"` - двойная кавычка внутри строки
- Можно делать многострочные строки с помощью специальных приёмов.
Итог
- Если нужна простая строка без специальных символов - лучше одинарные кавычки.
- Если нужно вставить перенос строки или спецсимволы - двойные.
- Обе кавычки позволяют защищать двоеточия и другие спецсимволы в значениях.

### Переменные в group_vars

В больших проектах,  переменные, относящиеся к хостам рекомендуются хранить в папке `group_vars`. В ней необходимо создать файлы с именем группы, которая указана в файле инвентаризации.

Вот изначальный файл инвентаризации до переноса переменных в `group_vars`

```ini
[master_servers]
minion1

[worker_servers]
minion2

[cluster_servers:children]
master_servers
worker_servers

[cluster_servers:vars]
anisble_username=developer 
ansible_ssh_private_key_file=/home/developer/.ssh/id_rsa
ansible_python_interpreter=/usr/bin/python3
```

Необходимо создать в корне проекта каталог `group_vars`. В нем содаем файл `cluster_servers.yml` переносим из него строки с переменными, заменив =(равно) на :(двоеточие), приведя к yaml формату:

```yaml
---
anisble_username: developer 
ansible_ssh_private_key_file: /home/developer/.ssh/id_rsa
ansible_python_interpreter: /usr/bin/python3
```

в итоге файл инвентаризации будет таким, без секции `[cluster_servers:vars]`:

```yaml
[master_servers]
minion1

[worker_servers]
minion2

[cluster_servers:children]
master_servers
worker_servers
```

### Переменные в host_vars

Подобно тому, как общие для групп переменные рекомендуется хранить в папке `group_vars`, уникальные для отдельных хостов переменные рекомендуется хранить в каталоге `host_vars`, чтобы не "засорять" файл инвентаризации, который по мере роста проекта может стать сложным для чтения и поиска в нем информации. Ansible автоматически умеет находить переменные определенные в этих двух каталогах. 
В каталоге `host_vars` необходимо создать файл с именем хоста, и в нем в формате yml определить переменные уникальные именно для этого хоста.

## Шаг 4. Плейбуки (Playbooks)

Playbook, это yaml файл, с определенной структурой, в котором указано несколько ansible команд, выполняющие определенную цель. Как правило файлы с плейбуками хранятся каталоге `playbooks` которая находится в корне проекта.

### Первый Playbook

Напишем первый плейбук, который будет выполнять проверку соединения с хостами, выволнив команду ping, для этого создадим в каталоге `playbooks` файл `ping.yml` со следующим содержимым:

```yaml
---
- name: Test connection to my servers # Имя плейбука
  hosts: all # к каким хостам применять
  become: yes # использовать sudo

  tasks: # массив задач плейбука
  - name: Ping my servers # имя задачи
    ping: # модуль используемый в задаче
```

Запустим плейбук: `ansible-playbook playbooks/ping.yml`

```bash
PLAY [Test connection to my servers] 

TASK [Gathering Facts] 
ok: [minion1]
ok: [minion2]

TASK [Ping my servers] 
ok: [minion2]
ok: [minion1]

PLAY RECAP 
minion1 : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
minion2 : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

### Второй плейбук - установка веб-сервера nginx

Второй плейбук будет выполнять установку веб-сервера на debian совместимые системы. Для этого в каталоге с плейбуками создадим файл с именем `install_nginx.yml` со следующим содержимым:

```yaml
---
- name: Install Nginx on Debian-like systems
  hosts: all
  become: true

  tasks:
    - name: Update apt cache
      apt: # Обновить кэш apt
        update_cache: yes # Обновить кэш пакетов, чтобы получить актуальную информацию

    - name: Install nginx 
      apt: # Установить пакет nginx
        name: nginx # Указать имя пакета
        state: present # Убедиться, что пакет установлен

    - name: Ensure nginx is running and enabled
      service: # Убедиться, что сервис nginx запущен и включён
        name: nginx # Указать имя сервиса
        state: started # Убедиться, что сервис запущен
        enabled: yes # Включить автозапуск при загрузке
```

Затем выполним плейбук `ansible-playbook playbooks/install_nginx.yml`:

```bash
PLAY [Install Nginx on all servers] 

TASK [Gathering Facts] 
ok: [minion2]
ok: [minion1]

TASK [Update apt cache] 
changed: [minion2]
changed: [minion1]

TASK [Install nginx] 
changed: [minion2]
changed: [minion1]

TASK [Ensure nginx is running and enabled] 
changed: [minion1]
changed: [minion2]
 
```

Можем проверить установку, запросив страничку с вебсервера, как будто бы мы ее открыли в браузере, выполнив для этого команду `curl -l minion1`, и видим ответ, стандартная HTML страница по умолчанию для веб-сервера nginx. Значит nginx установлен и работает.

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### Индепонентность в ansible

Идемпотентность в Ansible означает, что повторный запуск плейбука не изменит систему, если она уже находится в нужном состоянии.
Коротко:

- Один и тот же плейбук можно запускать много раз - результат останется одинаковым.
- Ansible сам проверяет, нужно ли что-то менять (например, не переустанавливает nginx, если он уже установлен).
- Это делает автоматизацию надёжной и безопасной.

Мы можем повторно запустить плейбук установки веб-сервера nginx  `ansible-playbook playbooks/install_nginx.yml` и увидим:

```bash

PLAY [Install Nginx on all servers] 

TASK [Gathering Facts] 
ok: [minion2]
ok: [minion1]

TASK [Update apt cache] 
ok: [minion2]
ok: [minion1]

TASK [Install nginx] 
ok: [minion1]
ok: [minion2]

TASK [Ensure nginx is running and enabled] 
ok: [minion2]
ok: [minion1]

PLAY RECAP 
minion1                    : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
minion2                    : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Если мы сравним этот лог с логом первого запуском плейбука, то заметим разницу, при первом запуске изменения применялись (`changed: [имя_хоста]`), а при повторном уже нет. Вот часть лога первого запуска, где показано что изменения применились.

```bash
TASK [Update apt cache] 
changed: [minion2]
changed: [minion1]

TASK [Install nginx] 
changed: [minion2]
changed: [minion1]

TASK [Ensure nginx is running and enabled] 
changed: [minion1]
changed: [minion2]
```

Индепонентность даем такие преимущества:

- Экономит время: Ansible пропускает уже выполненные задачи - плейбук работает быстрее.
- Безопасно для производственной среды: не перезапустит сервис без причины, не удалит и не перезапишет лишнего.
- Поддержка состояния: Ansible описывает желаемое состояние, а не пошаговые инструкции что нужно сделать.
- Можно использовать в cron или CI/CD: повторные запуски не сломают систему.

### Третий плейбук - установка веб-сервера nginx с копированием контента сайта

Третий плейбук будет так же устанавливать веб-сервер nginx, но дефолтную страницу сайта мы заменим на свою. 

В корне проекта создадим каталог `website_content`, в нем создадим файл `index.html`, который будет содержать нашу html страницу с таким содержанием:

```html
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <title>Hello World</title>
</head>
<body>
  <h1>Hello World</h1>
</body>
</html>
```

Так же в папке c плейбуками создадим еще один плейбук с именем `install_nginx_content.yml`:

```yaml
---
- name: Install Nginx with content
  hosts: all
  become: true
  
  vars:
    source_file: "../website_content/index.html" # Путь к исходному файлу index.html, который будет скопирован на сервер
    dest_file: "/var/www/html" # Путь назначения для файла index.html на сервере

  tasks:
    - name: Update apt cache
      apt: update_cache=yes

    - name: Install nginx 
      apt: name=nginx state=present

    - name: Copy custom index.html to Nginx web root
      copy: # Копировать файл index.html
        src: "{{ source_file }}" # Указать путь к исходному файлу на локальной машине
        dest: "{{ dest_file }}"  # Указать путь назначения на сервере
        mode: '0644' # Установить права доступа к файлу
      notify: Restart Nginx # Уведомить обработчик о необходимости перезапуска Nginx после копирования файла

    - name: Ensure nginx is running and enabled
      service: name=nginx state=started enabled=yes

  handlers: # Обработчики выполняются только при вызове notify
    - name: Restart Nginx # Обработчик для перезапуска Nginx
      service: # Модуль для управления сервисами
        name: nginx # Указать имя сервиса
        state: restarted # Перезапустить сервис
```

В этом плейбуке мы добавили задачу копирования файла с нашей html страницей, которая заменяет дефолтную страницу nginx. В таске копирования мы используем переменные, которые заранее определили, и в которых задаем откуда и куда копировать страницу. Затем в таске копирования нашей html страницы мы использовали эти переменные. Также, в демонстрационных целях добавили обработчик в секцию `handlers`, который перезапускает веб-сервер, после смены дефолтной страницы. Для этого задача копирования должна вызвать этот обработчик через инструкцию `notify: Имя_обработчика`.  
Чтобы не "захламлять" плейбук, обработчики рекомендуется сохранять в предназначенную для этого папку `handlers`, которую размещают в корне проекта.

Запустим плейбук  `ansible-playbook playbooks/install_nginx_content.yml`:

```bash
PLAY [Install Nginx with content] 

TASK [Gathering Facts] 
ok: [minion2]
ok: [minion1]

TASK [Update apt cache] 
ok: [minion1]
ok: [minion2]

TASK [Install nginx] 
ok: [minion1]
ok: [minion2]

TASK [Copy custom index.html to Nginx web root] 
changed: [minion2]
changed: [minion1]

TASK [Ensure nginx is running and enabled] 
ok: [minion2]
ok: [minion1]

PLAY RECAP 
minion1                    : ok=5    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
minion2                    : ok=5    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Теперь проверим, запросив нашу страницу с веб-сервера `curl -l minion1`, и мы видим что нам вернулась наша скопированная ранее на веб-сервер страница:

```bash
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <title>Hello World</title>
</head>
<body>
  <h1>Hello World</h1>
</body>
```

## Шаг 5. Переменные, Debug, Set_fact, Register

При использовании переменных, ее имя обрамляется двойными фигурными скобками `{{ var_name }}`, что дает ансибл понять, что сюда надо подставить значение переменной, ранее определенной с таким именем.

### Переопределение переменных

Переменные можно определять в различных частях проекта, и в том числе переопределять значение одно и той же переменной.
Нииже показано, в какой очередности будет переопределятся значение переменной, если мы переопределим ее в разных местах. 

```yaml
# roles/myrole/defaults/main.yml
# --------------------------------
# Значения по умолчанию для роли.
# Наименьший приоритет - легко переопределяются в любом другом месте.
myvar: "from_role_defaults"

# roles/myrole/vars/main.yml
# ---------------------------
# Жёстко зашитые переменные внутри роли.
# Переопределяются только set_fact и extra_vars.
myvar: "from_role_vars"

# vars_files/vars.yml
# --------------------
# Файл с переменными, подключённый через `vars_files` в playbook'е.
# Приоритет выше, чем у group_vars и host_vars.
myvar: "from_vars_file"

# gathered facts (gather_facts)
# ------------------------------
# Автоматически собираемые данные о хосте.
# Примеры: ansible_user, ansible_os_family, ansible_hostname и т.д.
myvar: "{{ ansible_distribution }}"  # например: "Debian"

# inventory file (hosts или к примеру inventory.ini)
# ------------------------------------
# Переменные, заданные прямо в инвентори.
# Часто используются для задания IP, путей и значений по умолчанию.
myvar: "from_inventory"

# group_vars/all.yml
# -------------------
# Переменные, применяемые ко всем хостам группы.
# Выше по приоритету, чем инвентори.
myvar: "from_group_vars"

# host_vars/minion1.yml
# -----------------------
# Переменные, специфичные для одного хоста.
# Приоритет выше, чем у group_vars.
myvar: "from_host_vars"

# playbook.yml - переменная в секции vars
# ---------------------------------------
# Задано прямо внутри playbook'а.
# Переопределяет всё выше (кроме register, set_fact, extra_vars).
vars:
  myvar: "from_playbook_vars"

# Внутри таски с register
# ------------------------
# Сохраняет результат команды или модуля.
# Переменная имеет более высокий приоритет, чем playbook vars.
- name: Set via register
  command: echo "from_register"
  register: myvar

# Внутри таски с set_fact
# ------------------------
# Задаёт переменную во время выполнения.
# Очень высокий приоритет (проигрывает только extra_vars).
- name: Set via set_fact
  set_fact:
    myvar: "from_set_fact"

# CLI (аргументы командной строки)
# --------------------------------
# Самый высокий приоритет.
# Побеждает всё, включая set_fact и register.
ansible-playbook playbook.yml -e "myvar=from_extra_vars"
```

Не все еще компоненты ансибл мы пока изучили (такие как роли, регистры и т.д.), но всегда можно вернуться к этой шпаргалке, чтобы понять в какой очередности переопределяются переменные. Это важно понимать для построения логики работы плейбуков.

В демонстрационных целя добавим несколько переменных в разных местах.
В инвентори файле hosts добавим хостам переменную `owner`:

```ini
[master_servers]
minion1 owner="Alex form inventory"

[worker_servers]
minion2 owner="Maxim form inventory"
...
```

Переопределим эту же переменную, но уже в каталоге `host_vars`  
`minion1.yml`:

```yml
owner: "Alex form host_vars"
```

`minion2.yml`:

```yml
owner: "Maxim form host_vars"
```

### Пример работы с переменными

Для лучшего понимани работы с переменными, создадим новый плейбук, и назовем его `varitables.yml` (далее мы более подробно рассмотрим некоторые элементы этого плейбука, такие как `set_facts`, `register` и т.д., пока на них можно не заострять внимание):

```yaml
---
- name: Playbook to demonstrating working with variables
  hosts: all
  become: true

  vars:
    message1: "Hello" # Переменные можно задавать в самом плейбуке
    message2: "World"
    my_var1: "The my variable"

  tasks:
    - name: Print the value of my_var1 via var
      debug: # Команда debug выводит значение переменной на экран
        var: my_var1 # var позволяет вывести значение переменной на экран

    - name: Print the value of my_var1 via msg
      debug: # Команда debug также позволяет вывести значение переменной в строке
        msg: "The value of my_var1 is: {{ my_var1 }}"  # msg позволяет вывести значение переменной в строке

    - debug:
        msg: "Owner of this server: {{ owner }}" # Переменная owner определена в инвентарном файле, и host_vars и будет подставлена в строку

    - name: "Set a new variable using set_fact"
      set_fact: my_new_varitable="{{ message1 }} {{ message2 }} from {{ owner }}"
      # Команда set_fact позволяет создать новую переменную my_new_varitable, 
      # которая будет содержать значение переменных message1 и message2, а также owner.
      # Эта переменная будет доступна в последующих задачах плейбука.

    - debug:
        var: my_new_varitable # Выводим значение переменной my_new_varitable на экран

    - debug: 
        var: ansible_memfree_mb # Переменная (факт) ansible_memfree_mb содержит значение свободной памяти на сервере в мегабайтах
    
    - name: "Get uptime of the server using shell module and register the result"
      shell: uptime # выполняем команду uptime на сервере через модуль shell
      register: uptime_result # Результат выполнения команды будет сохранен в переменную uptime_result

    - name: "Print the uptime_result of the server"
      debug:
        var: uptime_result # Выводим значение переменной uptime_result на экран

    - name: "Print the uptime_result.stdout" # 
      debug:
        msg: "Uptime of the server is: {{ uptime_result.stdout }}" # Выводим значение переменной uptime_result.stdout на экран
```

Запускаем плейбук `ansible-playbook playbooks/varitables.yml` и смотри вывод:

```bash
PLAY [Playbook to demonstrating working with variables]

TASK [Gathering Facts]
ok: [minion2]
ok: [minion1]

TASK [Print the value of my_var1 via var]
ok: [minion1] => {
    "my_var1": "The my variable"
}
ok: [minion2] => {
    "my_var1": "The my variable"
}

TASK [Print the value of my_var1 via msg]
ok: [minion1] => {
    "msg": "The value of my_var1 is: The my variable"
}
ok: [minion2] => {
    "msg": "The value of my_var1 is: The my variable"
}

TASK [debug]
ok: [minion1] => {
    "msg": "Owner of this server: Alex form host_vars"
}
ok: [minion2] => {
    "msg": "Owner of this server: Maxim form host_vars"
}

TASK [Set a new variable using set_fact]
ok: [minion1]
ok: [minion2]

TASK [debug]
ok: [minion1] => {
    "my_new_varitable": "Hello World from Alex form host_vars"
}
ok: [minion2] => {
    "my_new_varitable": "Hello World from Maxim form host_vars"
}

TASK [debug]
ok: [minion1] => {
    "ansible_memfree_mb": 5310
}
ok: [minion2] => {
    "ansible_memfree_mb": 5310
}

TASK [Get uptime of the server using shell module and register the result]
changed: [minion2]
changed: [minion1]

TASK [Print the uptime_result of the server]
ok: [minion1] => {
    "uptime_result": {
        "changed": true,
        "cmd": "uptime",
        "delta": "0:00:00.005227",
        "end": "2025-07-01 22:13:31.333633",
        "failed": false,
        "msg": "",
        "rc": 0,
        "start": "2025-07-01 22:13:31.328406",
        "stderr": "",
        "stderr_lines": [],
        "stdout": " 22:13:31 up  8:39,  0 users,  load average: 0.11, 0.12, 0.10",
        "stdout_lines": [
            " 22:13:31 up  8:39,  0 users,  load average: 0.11, 0.12, 0.10"
        ]
    }
}
ok: [minion2] => {
    "uptime_result": {
        "changed": true,
        "cmd": "uptime",
        "delta": "0:00:00.003344",
        "end": "2025-07-01 22:13:31.333838",
        "failed": false,
        "msg": "",
        "rc": 0,
        "start": "2025-07-01 22:13:31.330494",
        "stderr": "",
        "stderr_lines": [],
        "stdout": " 22:13:31 up  8:39,  0 users,  load average: 0.11, 0.12, 0.10",
        "stdout_lines": [
            " 22:13:31 up  8:39,  0 users,  load average: 0.11, 0.12, 0.10"
        ]
    }
}

TASK [Print the uptime_result.stdout]
ok: [minion1] => {
    "msg": "Uptime of the server is:  22:13:31 up  8:39,  0 users,  load average: 0.11, 0.12, 0.10"
}
ok: [minion2] => {
    "msg": "Uptime of the server is:  22:13:31 up  8:39,  0 users,  load average: 0.11, 0.12, 0.10"
}
```

Не смотря на то что у нас были определенны переменные в инвентарном файле `hosts`, занчения переменных взяты из каталога `host_vars`:

```bash
ok: [minion1] => {  
    "my_new_varitable": "Hello World from Alex form host_vars"  
}  
ok: [minion2] => {  
    "my_new_varitable": "Hello World from Maxim form host_vars"  
}  
```

### Модуль set_fact

Далее мы используем `set_fact` - это модуль в Ansible, который позволяет создавать переменные во время выполнения (runtime). Эти переменные сохраняются только в текущем контексте (для хоста) и могут использоваться в следующих задачах.
Еще раз, что важно запомнить:

- Переменная, созданная через set_fact, доступна только для текущего хоста, для которого запущенная данная задача.
- Она сохраняется до конца выполнения playbook.
- Можно использовать шаблоны Jinja2 для вычислений (о них мы поговорим позже).
- set_fact работает только на этапе выполнения (runtime), в отличие от переменных, объявленных в vars.

В нашем плейбуке мы создали новую переменнуюю `my_new_varitable`, соединив значение трех переменных (`message1`, `message2`, `owner`) в одну строку, и вывели потом ее на экран.

### Модуль setup (факты)

Модуль `setup` в Ansible собирает информацию о целевом хосте: ОС, сеть, память, CPU, переменные окружения и т.д. и определняет соответствующие переменные (их назвают `факты`), которые мы затем можем использовать в своих плйебуках.
Ансибл автоматически запускает этот модуль, перед запуском плейбука. Но этим поведение можно управлять, для того в плейбуке необходимо отключить автосбор фактов, задав значение `gather_facts: no`

```yaml
- hosts: all
  gather_facts: no
  tasks:
    - debug:
        msg: "{{ ansible_hostname }}" # Переменная будет пустая, если только не определелии вручную, так как мы отключили сбор фактов
```

Можно вручную запустить сбор фактов, и даже ограничить их сбор с помощью фильтра.

```yaml
- setup:
    filter: "ansible_distribution"
```

Команда заполнит только те факты, чьи ключи начинаются с ansible_distribution, например:

- ansible_distribution
- ansible_distribution_version
- ansible_distribution_major_version

Остальные (например, ansible_hostname, ansible_memtotal_mb, ansible_default_ipv4) не будут собраны.

Вот список часто используемых фактов Ansible (setup):

- Хост / ОС:
  - `ansible_hostname` - имя хоста
  - `ansible_fqdn` - полное доменное имя
  - `ansible_distribution` - дистрибутив (например, Ubuntu)
  - `ansible_distribution_version` - версия (например, 20.04)
  - `ansible_architecture` - архитектура (например, x86_64)
  - `ansible_os_family` - семейство ОС (Debian, RedHat)
- Сеть:
  - `ansible_default_ipv4.address` - основной IP-адрес
  - `ansible_all_ipv4_addresses` - все IPv4
  - `ansible_interfaces` - список сетевых интерфейсов
  - `ansible_eth0.ipv4.address` - IP интерфейса eth0
- Процессор / Память:
  - `ansible_processor_cores` - количество ядер
  - `ansible_processor` - список CPU
  - `ansible_memtotal_mb` - всего памяти (в МБ)
  - `ansible_memfree_mb` - свободной памяти
- Диски:
  - `ansible_mounts` - список точек монтирования
  - `ansible_mounts.0.size_total` - размер первого раздела

В нашем примере мы вывели на экран объем свободной памяти в мегабайтах

```sh
TASK [debug]
ok: [minion1] => {
    "ansible_memfree_mb": 5310
}
ok: [minion2] => {
    "ansible_memfree_mb": 5310
}
```

### Директива register

`register` в Ansible - это директива, которая сохраняет результат выполнения задачи (обычно shell-команды) в переменную.
В нашем примере мы выполнили команду `uptime` через модуль `shell`, и сохранили результат в переменную `uptime_result`

```yml
- name: Run uptime and save output
  shell: uptime
  register: uptime_result
```

и как увидели при запуске плейбука, в uptime_result стала доступна структура с полями:

- stdout - стандартный вывод
- stderr - ошибки
- rc - код возврата
- changed - был ли выполнен changed
- stdout_lines - список строк из stdout
То есть `register` - это способ сохранить вывод команды и использовать его позже.

## Шаг 6. Условия (when) и блоки (block)

### Условия when

Добавим новый хост `minion3` которым мы будем управлять на базе redhat-like дистрибутива в инвентори файл `hosts`

```ini
...
[worker_servers]
minion2 owner="Maxim form inventory"
minion3
...
```

Далее напишем новый плейбук устанвоки nginix `install_nginix_when.yml`. Так как minion3 имеет другой linux дистрибутив, прежний наш плейбук выстаст ошибку, при поытке использовать модуль apt, так как он использует другой менеджер пакетов. И здесь на поможет директива `when`, которая используется для условного выполнения задач, хендлеров или ролей.

```yaml
- name: Install Nginx with content to a redHat-like system and debian-like system
  hosts: all
  become: true
  
  vars:
    source_file: "../website_content/index.html" 
    dest_file: "/var/www/html"
    dest_file_redhat: "/usr/share/nginx/html" # Путь назначения для файла index.html на RedHat-подобных системах

  tasks:
    - name: Update apt cache  # Обновить кэш apt перед установкой пакетов
      apt: # Модуль для управления пакетами в Debian-подобных системах
        update_cache: yes # Обновить кэш перед установкой пакетов
      when: ansible_os_family == "Debian" # Выполнить задачу только для Debian-подобных систем

    - name: Install nginx on Debian-like systems # Установить пакет nginx на Debian-подобных системах
      apt: 
        name: nginx # Установить пакет nginx
        state: present # Убедиться, что пакет установлен 
      when: ansible_os_family == "Debian"

    - name: Install EPEL repository # Установить EPEL репозиторий для RedHat-подобных систем
      dnf: # Модуль для управления пакетами в RedHat-подобных системах
        name: epel-release # Установить пакет epel-release, который предоставляет доступ к EPEL репозиторию
        state: present 
      when: ansible_os_family == "RedHat" # Выполнить задачу только для RedHat-подобных систем

    - name: Install nginx on RedHat-like systems # Установить пакет nginx на RedHat-подобных системах
      dnf: 
        name: nginx
        state: present
      when: ansible_os_family == "RedHat" 

    - name: Copy custom index.html to Nginx web root on Debian-like systems
      copy: 
        src: "{{ source_file }}" 
        dest: "{{ dest_file }}" 
        mode: '0644' 
      when: ansible_os_family == "Debian"

    - name: Copy custom index.html to Nginx web root on RedHat-like systems
      copy: 
        src: "{{ source_file }}" 
        dest: "{{ dest_file_redhat }}" 
        mode: '0644' 
      when: ansible_os_family == "RedHat" 

    - name: Ensure nginx is running and enabled 
      service: name=nginx state=started enabled=yes
      when: ansible_os_family == "Debian" 

    - name:  Reload or start nginx in container 
      shell: |
        nginx -t && nginx -s reload || nginx
      ignore_errors: true
      when: ansible_os_family == "RedHat"
```

Запустим плейбук `ansible-playbook playbooks/install_nginix_when.yml`. Анализируя вывод, мы можем заметить что благодаря директиве `when`, задачи преднаначенные для debian-like хостов пропускаются на хосте с redhat-like системой, и наборот. 

```bash
TASK [Update apt cache] 
skipping: [minion3]
ok: [minion1]
ok: [minion2]

...
TASK [Install EPEL repository] 
skipping: [minion1]
skipping: [minion2]
ok: [minion3]
```

Но в нашем плейбуке много лишнего и дублирующего кода. Мы может сократить и огранизовать плейбук более изящно, если будем испольовать директиву `block`.

### Директива block

Директива `block` группирует задачи в единный блок, к которому можно применить when, become, tags и другие, к блокам можно применять директивы `rescue` и `always` для обработки ошибок.

```yaml
- name: Пример использования block/rescue/always
  block:
    - name: Задача 1
      command: /bin/true 
    - name: Задача 2
      command: /bin/false
  rescue:
    - name: Задача обработка ошибки
      debug:
        msg: "Произошла ошибка"
  always:
    - name: Задача всегда выполняется
      debug:
        msg: "Завершение блока"
```

Блоки могут быть вложенными друг в друга. Ключевые моменты по вложенности блоков:

- **Иерархия**: Задачи внутри block являются дочерними по отношению к этому блоку. Внутренний block является дочерним по отношению к внешнему block.
- **Применение параметров**: Параметры, примененные к внешнему блоку (например, when), будут влиять на все задачи внутри него, включая задачи во вложенных блоках. Параметры, примененные к внутреннему блоку, будут влиять только на его задачи. Если параметры конфликтуют, параметры внутреннего блока имеют приоритет.
- **Обработка ошибок (rescue/always)**: Секции rescue и always всегда относятся к ближайшему родительскому block. Если задача во внутреннем блоке завершается с ошибкой, будет активирована секция rescue этого внутреннего блока (если она есть). Если её нет, ошибка "пропагерирует" к следующему внешнему блоку, пока не будет найдена секция rescue или пока ошибка не достигнет уровня плейбука.

Перепишем плейбук с использованием директивы `block` - `install_nginx_block.yml`, а также вместо использования специфических модулей установки пакетов (apt, dnf), используем модуль `package` который является универсальным и работает с различными системами управления пакетами (например, apt для Debian/Ubuntu, yum/dnf для CentOS/RHEL, pacman для Arch Linux и т.д.). Ansible сам определит, какой менеджер пакетов используется на целевом сервере, и выполнит соответствующую команду.

```yaml
- name: Install Nginx with content to a redHat-like system and debian-like system
  hosts: all
  become: true
  
  vars:
    source_file: "../website_content/index.html" 
    dest_file: "/var/www/html"
    dest_file_redhat: "/usr/share/nginx/html" # Путь назначения для файла index.html на RedHat-подобных системах

  tasks:
    - name: Ensure the Nginx package is installed # Убедиться, что пакет nginx установлен
      package: # Модуль для управления пакетами
        name: nginx # Имя пакета для установки
        state: present # Убедиться, что пакет установлен
        update_cache: yes # Обновить кэш пакетов перед установкой

    - block: # Копирование файла index.html и запуск Nginx на Debian-подобных системах
      - name: Copy custom index.html to Nginx web root on Debian-like systems
        copy: 
          src: "{{ source_file }}" 
          dest: "{{ dest_file }}" 
          mode: '0644' 

      - name: Ensure nginx is running and enabled 
        service: name=nginx state=started enabled=yes
      when: ansible_os_family == "Debian"

    - block: # Копирование файла index.html и запуск Nginx на RedHat-подобных системах
      - name: Copy custom index.html to Nginx web root on RedHat-like systems
        copy: 
          src: "{{ source_file }}" 
          dest: "{{ dest_file_redhat }}" 
          mode: '0644' 

      - name:  Reload or start nginx in container 
        shell: |
          nginx -t && nginx -s reload || nginx
        ignore_errors: true
      when: ansible_os_family == "RedHat"
```

Код плейбука получился более комактным и выразительным.

## Шаг 7. Циклы Loop, Until, With_*

Циклы используют, чтобы повторить какое-либо действие несколько раз. В ansible есть несколько способов, как органиовать цикл. 

### Цикл loop (with_items)

Создадим новый плейбук `playbook/loop.yaml`:

```yml
---
- name: Looops Playbooks
  hosts: minion1 # ограничим выполенение одним хостом
  becom: yes

  tasks:
  - name: Say Hello
    debug: msg="Hello {{ item }}" # распечатываем элементы цикла, предопреленное имя переменной цикла - item
    #with_items: //использовалась до верси ansible 2.5
    loop:  # определение цикла
      - "Alex" # элемент цикла
      - "Max"
      - "Katya"
      - "Mery"  
```

запустим, и смотрим вывод

```sh
TASK [Say Hello]
ok: [minion1] => (item=Alex) => {
    "msg": "Hello Alex"
}
ok: [minion1] => (item=Max) => {
    "msg": "Hello Max"
}
ok: [minion1] => (item=Katya) => {
    "msg": "Hello Katya"
}
ok: [minion1] => (item=Mery) => {
    "msg": "Hello Mery"
}
```
Как и ожидалось, директива `debug: msg="Hello {{ item }}"` выполнилась для каждого элемента цикла, заданного с помощью `loop`.

Вот пример уже более полезного и реального кейса, длл установки приложений через цикл loop. Создадим новый плейбук `install_packages.yml`:
```yaml
---
- name: Install packages via loop playbook
  hosts: all
  become: yes

  tasks:
  # Обновляем кеш пакетов один раз, чтобы не выполнять при каждом шаге цикла
  - name: Update package cache on Debian-based systems
    apt:
      update_cache: yes
    when: ansible_os_family == "Debian"

  - name: Update package cache on RedHat-based systems
    yum:
      update_cache: yes
    when: ansible_os_family == "RedHat"

  # Выполняем установку пакетов в цикле
  - name: Install many packages
    package: name={{ item }} state=present update_cache=no
    loop: 
      - tree
      - tmux
      - tcpdump
      - dnsutils 
      - nmap

```
Запускаем плейбук и при желании можем убедиться, что на всех серверах приложения успешно установились. 

### Цикл until

Цикл `until` выполняется до тех пор, пока не выполниться определенное нами условие. Ключевые моменты:
- Выполняется минимум один раз.
- Если условие не истинно, повторяется до `retries` раз.
- Между попытками пауза `delay` секунд.
- Условие проверяется по зарегистрированной переменной (через директиву `register`).
- Если условие не выполнено после всех попыток - задача считается неудачной.

Создадим новый плейбук `playbook/until.yaml`:

```yml
---
- name: Until Playbook
  hosts: minion1
  become: yes

  tasks:
    - name: until loop example                              
      shell: echo -n A >> until_demo.txt && cat until_demo.txt  # Добавляет 'A' без символа окончания строки и выводит содержимое файла
      register: result                                          # Сохраняем результат
      retries: 10                                               # До 10 повторов
      delay: 1                                                  # Пауза 1 секунда
      until: result.stdout.find('AAAAA') == false               # Повторять, пока 'AAAAA' не найдено

    - name: Show current counter value # Отображаем конечный результат
      debug:
        msg: "Current value: {{ result.stdout }}"
```

Логика работы здесь такая. Добавили в файл букву А и выводим содержжмое файла. Сохраняем результат в переменную `result`. Проверяем, найдено ли в переменной result пять букв А (в первый раз естественно нет, там только одна А). Далее ждем одну секунду (а это отвечает delay: 1), и повторяем снова, и так до 10 раз (за это отвечает retries: 10). 
Есил бы мы искали не 5 букв А (ААААА), а к примеру 100, то цикл бы выполнился 11 раз (перый, и еще 10 повторений), и задача завершилась бы неудачей. 

Запустим плейбук с параметром -v (говорит ансибл делать более подробный вывод хода выполнения), чтобы отобразить все повторы цикла (по умолчанию, чтобы не перегруать информацией вывод, ансибл покает только первые 2) `ansible-playbook playbooks/until.yml -v`

```sh
TASK [until loop example]
FAILED - RETRYING: [minion1]: until loop example (10 retries left).
FAILED - RETRYING: [minion1]: until loop example (9 retries left).
FAILED - RETRYING: [minion1]: until loop example (8 retries left).
FAILED - RETRYING: [minion1]: until loop example (7 retries left).
changed: [minion1] => {"attempts": 5, "changed": true, "cmd": "echo -n A >> until_demo.txt && cat until_demo.txt", "delta": "0:00:00.002889", "end": "2025-07-04 23:06:41.871835", "msg": "", "rc": 0, "start": "2025-07-04 23:06:41.868946", "stderr": "", "stderr_lines": [], "stdout": "AAAAA", "stdout_lines": ["AAAAA"]}

TASK [Show current counter value]
ok: [minion1] => {
    "msg": "Current value: AAAAA"
}
```

Цикл `until` применяется, где нужно ждать, пока что-то изменится или станет доступно, с контролем количества попыток и интервалом.

Например в этой задаче мы ждем 50 секунд (10 попыток по 5 секунд), пока хост не начнет пинговаться:
```yaml
- name: Wait until 192.168.1.100 responds to ping
  shell: ping -c 1 -W 1 192.168.1.100 # Отправляет один ping (таймаут 1 секунда)
  register: ping_result        # Сохраняем результат
  ignore_errors: yes           # Не прерывать выполнение при ошибке
  retries: 10                  # Повторить до 10 раз
  delay: 5                     # Пауза 5 секунд между попытками
  until: ping_result.rc == 0   # Условие: успешный код возврата
```

примерный вывод:

```sh
TASK [Wait until 192.168.1.100 responds to ping]
FAILED - RETRYING: Wait until 192.168.1.100 responds to ping (9 retries left).
FAILED - RETRYING: Wait until 192.168.1.100 responds to ping (8 retries left).
ok: [localhost]
```


### Цикл With_fileglob и glob

Ранее мы написали плейбук `install_nginx_block.yml`, который устанавилвает вебсервер nginx, и заменяет страницу по умолчаниюю на наш вариант. Для дальнейшего изучения, я добавил папку `website_content2` куда положил два  файла изображений (`image1.jpeg` и `image2.png`) и отредактировнный `index.html` из первой папки, добавив в него отображжение этих изображений. Эти файлы мы и будем копировать, используя циклы. За основу мы возьмем плейбук `install_nginx_block.yml` изменив его для копирование файлов в цикле. 
Создадим новый плейбук `install_nginx_loop.yml` со следующим содержанием:

```yaml
- name: Install Nginx with content to a redHat-like system and debian-like system
  hosts: all
  become: true
  
  vars:
    source_folder: "../website_content2"  # устанавливаем директорию, откуда будем копировать
    dest_folder_debian: "/var/www/html" # Путь назначения для дефолтной страницы на Debian-подобных системах
    dest_folder_redhat: "/usr/share/nginx/html" # Путь назначения для дефолтной страницы на RedHat-подобных системах
    web_files: # Определяем элементы цикла в виде переменной web_files
      - index.html
      - image1.jpeg
      - image2.png

  tasks:
    - name: Ensure the Nginx package is installed 
      package: 
        name: nginx
        state: present
        update_cache: yes 

    - block: # Замена дефолтной страницы и запуск Nginx на Debian-подобных системах
      - name: 
        copy: 
          src: "{{ source_folder }}/{{ item }}"  #  откуда брать файл
          dest: "{{ dest_folder_debian }}"   #  каталог куда копировать файл
          mode: '0644' 
        loop: "{{ web_files }}"  # задаем какие элементы копировать в цикле

      - name: Ensure nginx is running and enabled 
        service: name=nginx state=started enabled=yes
      when: ansible_os_family == "Debian"

    - block: # Замена дефолтной страницы и запуск Nginx на RedHat-подобных системах
      - name: Copy custom index.html to Nginx web root on RedHat-like systems
        copy: 
          src: "{{ source_folder }}/{{ item }}"  #  откуда брать файл
          dest: "{{ dest_folder_redhat }}"   #  каталог куда копировать файл
          mode: '0644' 
        loop: "{{ web_files }}" # задаем какие элементы копировать в цикле

      - name:  Reload or start nginx in container 
        shell: |
          nginx -t && nginx -s reload || nginx
        ignore_errors: true
      when: ansible_os_family == "RedHat"
```

Можем запустить плейбук и убедиться что дефолтная страница заменилась на нашу, с картинками, перейдя по адресу http://localhost:8081 для minion1, http://localhost:8082 для minion2, и http://localhost:8083/ для minion3

В этом плейбуке мы использовали цикл, для копирования нескольких файлов в цикле, явно укаывая их имена. Но если файлов много, это будет крайне неудобно, прописывать каждый файл. Для решения этой проблемы можно использовать цикл `with_fileglob`.

 **with_fileglob** - это цикл в Ansible, который перебирает файлы по специальному шаблону (глобу) на локальной машине управления. 
 
Перепишем наш плейбук с учетом использования `with_fileglob`, строки где мы использовали цикл loop я закоментировал:

```yaml
- name: Install Nginx with content and loop on RedHat-like and Debian-like systems 
  hosts: all
  become: true
  
  vars:
    source_folder: "../website_content2"  # устанавливаем директорию, откуда будем копировать
    dest_folder_debian: "/var/www/html" # Путь назначения для дефолтной страницы на Debian-подобных системах
    dest_folder_redhat: "/usr/share/nginx/html" # Путь назначения для дефолтной страницы на RedHat-подобных системах
    # web_files: # Определяем элементы цикла в виде переменной web_files для цикла loop
    #   - index.html
    #   - image1.jpeg
    #   - image2.png

  tasks:
    - name: Ensure the Nginx package is installed 
      package: 
        name: nginx
        state: present
        update_cache: yes 

    - block: 
      - name: 
        copy: 
          # src: "{{ source_folder }}/{{ item }}"  # откуда будем копировать файл
          src: "{{ item }}" # откуда будем копировать файл
          dest: "{{ dest_folder_debian }}"  
          mode: '0644' 
        # loop: "{{ web_files }}"
        with_fileglob: "{{ source_folder }}/*.*"   # вовращаем в цикле по одному все файлы в каталоге

      - name: Ensure nginx is running and enabled 
        service: name=nginx state=started enabled=yes
      when: ansible_os_family == "Debian"

    - block: 
      - name: Copy custom index.html to Nginx web root on RedHat-like systems
        copy: 
          # src: "{{ source_folder }}/{{ item }}"   # откуда будем копировать файл
          src: "{{ item }}"  # откуда будем копировать файл
          dest: "{{ dest_folder_redhat }}"  
          mode: '0644' 
        # loop: "{{ web_files }}" 
        with_fileglob: "{{ source_folder }}/*.*"  # вовращаем в цикле по одному все файлы в каталоге

      - name:  Reload or start nginx in container 
        shell: |
          nginx -t && nginx -s reload || nginx
        ignore_errors: true
      when: ansible_os_family == "RedHat"
```
Так же я поправили параметр `src: "{{ item }}`, так как with_fileglob возвращает полный путь к файлу. 

#### Glob
Глоб (glob) - это шаблон для поиска файлов по маске с использованием специальных символов.

Можно использовать разлиные шаблоны глоб, и возвращать файлы по маске, допустим только те, чьи имена начинаются со слова `image` (маска `image*`).

Поддерживаются такие маски:
Маска	| Значение
--- | ---
`*`	| Любая последовательность символов (в том числе пустая)
`?`	| Один любой символ
`[abc]`	| Один символ из перечисленных (например, `a`, `b` или `c`)
`[a-z]`	| Один символ из диапазона
`**` 	| Рекурсивный поиск в подкаталогах

Примеры:

Шаблон | Что найдёт
 --- | --- 
`*.txt` | Все `.txt` файлы в каталоге
`file?.log` | `file1.log`, `fileA.log`, но не `file12.log`
`config[12].yml`| `config1.yml`, `config2.yml`
`**/*.conf` | Все `.conf` во всех подкаталогах

### Циклы with_* и эквиваленты с loop

В ансибл есть несколько директив вида `with_*`, с помощь которых можно организовать цикл для различных целей. Мы уже рассмотрели ранее `with_items` и `with_fileglob`, но их гораздо больше.

#### Основные with_* конструкции:

Директива | Что делает | Пример применения
--- | --- | ---
with_items | Перебор произвольного списка | Пройти по списку имён или чисел
with_fileglob | Перебор файлов по маске (глоб) на **контроллере** | Найти все `.conf` файлы в папке
with_file | Перебор файлов, но не имен, а содержимое каждого файла читается и передаётся как переменная | Чтение содержимого файлов в цикле
with_sequence | Генерация числовой последовательности | Создать список чисел от 1 до 10
with_dict | Перебор словаря | Пройти по ключам и значениям словаря `item.key + item.value`
with_together | Одновременный перебор **двух списков** по индексам | Сопоставить имена и IP-адреса по индексам `name[i] + ip[i]`
with_nested | Перебор всех комбинаций из нескольких списков | Декартово произведение. Если в одном списке цвета [красный, синий], а в другом размеры [S,M], переберет все их возможные комбинации - красный S, красный M, синий S, синий M и т.д.  
with_subelements | Для вложенных структур (списки в объектах) | Перебрать пользователей и их адреса
with_lines | Перебор строк из вывода команды или файла | Перебрать строки вывода `ls`
with_random_choice | Выбор одного случайного элемента из списка | Выбрать случайное имя из списка

Не будем в рамках вводного курса подробно останавливаться на примерах использования каждой конструкции, их можно посмотреть в документации. 

> Важно знать, что `with_*` - начиная с Ansible 2.5 это устаревший стиль, и разработчики ансибл рекомендуют использовать `loop` + `lookup()` вместо `with_*`. Но `with_*` всё ещё широко используется.

#### lookup
`lookup` - это специальная функция Ansible, которая позволяет получить данные из внешних источников до выполнения задачи (таска). `lookup` работает только на контроллере ансибл (т.е локально, а не на целевых хостах).

`lookup` использует **плагины**, которые определяют внешний источник данных. Например с помощью плагина `pipe` можно выполнить команду на контроллере: `lookup('pipe', 'date +%Y')` - выведет текущий год с контроллера.

Источник данных | Пример lookup 
--- | --- 
файл | `lookup('file', 'path/to/file.txt')` 
список файлов | `lookup('fileglob', '*.conf', wantlist=True)`
выполнение команды | `lookup('pipe', 'whoami')`
переменная по имени | `lookup('vars', 'var_name')` 
случайный элемент | `lookup('random_choice', ['a', 'b', 'c'])`


#### Эквиваленты with_* -> loop + lookup

with_* | Эквивалент loop
--- | ---
with_items: [...] | loop: [...]
with_fileglob: '*.conf' | loop: "{{ lookup('fileglob', '*.conf', wantlist=True) }}"
with_file: [...] | loop: "{{ lookup('file', [...], wantlist=True) }}"
with_sequence: start=1 end=5 | loop: "{{ range(1, 6) }}"
with_dict: {a: 1, b: 2} | loop: "{{ dict2items(my_dict) }}"
with_together: [list1, list2] | loop: "{{ zip(list1, list2) }}"
with_nested: [[1, 2], ['a', 'b']] | loop: "{{ product([ [1, 2], ['a', 'b'] ]) }}"
with_subelements | loop: "{{ lookup('subelements', ...) }}"
with_lines: some command | loop: "{{ lookup('pipe', 'some command') | splitlines }}"
with_random_choice: [...] | loop: "{{ [ lookup('random_choice', [...]) ] }}"

## Шаг 8. Шаблонизатор Jinja2 и модуль template

### Шаблонизатор Jinja2

Jinja2 - это шаблонизатор, используемый в Ansible для вставки переменных, условий и циклов в YAML-файлы.
Используется для:

- Определения переменных:
  - `{{ variable_name }}`
- Фильтров:
  - `{{ list | length }}` - возвращает длину списка list
  - `{{ name | upper }}` - возвращает строку name в верхнем регистре
- Условий:
  - `{% if var == "value" %}`...`{% endif %}` - выполняет блок кода, если условие `var == "value"` истинно
- Циклов:
  - `{% for item in list %}`...`{% endfor %}` - выполняет блок кода для каждого элемента из списка `list`, переменная `item` - текущий элемент
- Тестов:
  - `{{ var is defined }}` - проверяет, что переменная `var` существует (определена).
  - `{{ var is not none }}` - проверяет, что переменная `var` не равна `None` (не пустая).

Работает внутри `{{ ... }}` или `{% ... %}` - шаблонов.

Вот список популярных Jinja2-фильтров с краткими примерами:

Фильтр | Описание и пример
--- | ---
`length` | Кол-во элементов. `['a', 'b'] \| length` -> `2`
`upper` | В верхний регистр. `'abc' \| upper` -> `'ABC'`
`lower` | В нижний регистр. `'ABC' \| lower` -> `'abc'`
`replace('a','o')` | Замена символов. `'data' \| replace('a','o')` -> `'doto'`
`default('x')` | Значение по умолчанию. `none_var \| default('x')` -> `'x'`
`join(', ')` | Объединение списка. `['a','b'] \| join(', ')` -> `'a, b'`
`split(',')` | Разделение строки на список. `'a,b,c' \| split(',')` -> `['a','b','c']`
`sort` | Сортировка. `[3,1,2] \| sort` -> `[1,2,3]`
`unique` | Уникальные элементы. `[1,2,2] \| unique` -> `[1,2]`
`reverse` | Обратный порядок. `[1,2,3] \| reverse` -> `[3,2,1]`
`int` | Преобразование в число. `'42' \| int` -> `42`
`float` | Вещественное число. `'3.14' \| float` -> `3.14`
`trim` | Удаление пробелов. `'  abc  ' \| trim` -> `'abc'`
`capitalize` | Переводит первый символ строки в заглавную букву, а все остальные - в строчные. `'HELLO' \| capitalize` -> `'Hello'`

Фильтры можно комбинировать `' HELLO world ' | trim | lower | replace('world', 'there') | capitalize` -> 'Hello there'.

### Модуль template

Модуль `template` в Ansible используется для генерации файлов на основе Jinja2-шаблонов (как правило файлов с расширением `*.j2`). Он берет шаблонный файл, подставляет переменные и сохраняет результат в указанном пути на целевой машине.

**Ключевые моменты**:
- Источник: `src` - путь к шаблону.
- Назначение: `dest` - куда сохранить сгенерированный файл.
- Автоматическая подстановка переменных из окружения или playbook.
- Можно задавать права доступа (`mode`), владельца (`owner`) и группу (`group`).
- Используется для конфигов, скриптов и любых текстовых файлов с динамическим содержим

Простой пример использования:
```yaml
- name: Render config file
  template:
    src: config.j2
    dest: /etc/myapp/config.conf
    mode: '0644'
```

Для демонстрации. я создал каталог website_content3, куда поместил шаблон j2 - это по сути файл index.html из каталога website_content, в который я добавил следующие строки с переменными:

```j2
...
  <h1>Hello World</h1>
  <p>Server owner: {{ owner }}</p>
  <p>Server hostname: {{ ansible_hostname }}</p>
  <p>Server OS famely: {{ ansible_os_family }}</p>
  <p>Server uptime: {{ (ansible_uptime_seconds // 3600) }} h {{ (ansible_uptime_seconds % 3600) // 60 }} min</p>
</body>
...
```
Во время генерации дефолтной страницы модуль `template` автоматически подставить значения переменных в шаблон. 

На основе плейбука `install_nginx_block.yml` создадим новый - `install_nginx_template.yml`. В нём мы укажем путь к шаблону, путь и имя выходного файла и заменим команду `copy` на `template`. Вместо простого копирования файл будет сгенерирован заново: в шаблоне будут автоматически подставлены указанные нами переменные.
```yaml
- name: Install Nginx with content on RedHat-like and Debian-like systems 
  hosts: all
  become: true
  
  vars:
    source_file: "../website_content3/index.j2" # Путь к шаблону 
    dest_file: "/var/www/html"
    dest_file_redhat: "/usr/share/nginx/html" 

  tasks:
    - name: Ensure the Nginx package is installed 
      package: name=nginx state=present update_cache=yes 

    - block:
      - name: Generate index.html to Nginx web root on Debian-like systems
        template: # генерируем файл по шаблону, перменные автоматически подставятся
          src: "{{ source_file }}" 
          dest: "{{ dest_file }}/index.html" # Путь к сгенерированному файлу
          mode: '0644' 

      - name: Ensure nginx is running and enabled 
        service: name=nginx state=started enabled=yes
      when: ansible_os_family == "Debian"

    - block: 
      - name:  Generate index.html to Nginx web root on RedHat-like systems
        template:  # генерируем файл по шаблону, перменные автоматически подставятся
          src: "{{ source_file }}" 
          dest: "{{ dest_file_redhat }}/index.html" # Путь к сгенерированному файлу
          mode: '0644' 

      - name:  Reload or start nginx in container 
        shell: |
          nginx -t && nginx -s reload || nginx
        ignore_errors: true
      when: ansible_os_family == "RedHat"
```
Запустим плейбук `ansible-playbook playbooks/install_nginx_template.yml` и можем убедиться, что все сработало, сгенерированная новая html страница с данными сервера:
```
Hello World
Server owner: Alex form host_vars
Server hostname: c342ef95bc19
Server OS famely: Debian
Server uptime: 12 h 20 min
```