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

В первую очередь необходимо создать файл инвентаризации (inventory file) — это файл, в котором описываются целевые хосты и группы хостов для управления ими.

Какие имена он может иметь:  
- По умолчанию в Ansible:  
  - /etc/ansible/hosts — системный дефолтный инвентори-файл  
  - Можно использовать любой другой файл, указав его в параметре -i (inventory) при запуске ansible/ansible-playbook.  
- Варианты имён:
  - hosts — часто называют просто так
  - inventory — часто встречается
  - production / staging / dev — имена для разных окружений
  - inventory.ini — когда файл в INI-формате
  - inventory.yaml или inventory.yml — если инвентори в YAML-формате
  - inventory.json — в JSON-формате
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

Во время первого подключюения будет сделан запрос на добавления хоста в known_hosts — это механизм безопасности SSH.

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
- При следующих подключениях SSH проверяет, что ключ сервера совпадает с сохранённым — чтобы убедиться, что вы подключаетесь именно к тому же серверу, а не к злоумышленнику (защита от MITM-атак).

Мы можем настроить чтобы ансибл автоматически принимал ключи новых хостов, но в производственной среде так делать крайне не рекомендуется. 

Для этого создадим файл конфигурации для ансибл `ansible.cfg` в корне проекта с содержимым:
```ini
[defaults]
host_key_checking = false 
inventory         = ./hosts #
```
- `host_key_checking = false` - автоматически приняите ключей для новых хостов, 
- `inventory = ./hosts` - путь к файлу инвентаризации и теперь при выполнении команд ансибл, не обязательно указывать каждый раз путь к файлу инвентаризации, как в этом примере `ansible all -m ping `

### Правила создания файла инвентаризациии
В минимальном исполнении достаточно просто перечислить IP хостов
```
10.0.15.12
10.0.15.13
```

Можно дать хосту алиас
```
ansible-debian ansible_host=10.0.15.12
```

Можно включить сервер в группу, для этого название группы надо обрамить квадратными скобками
```
[group_name]
hostname ansible_host=10.0.15.12
```

В команде ansible по умолчанию доступны изначально два параметра, для указания к каким хостам применять инструкции:
-  **`all`** - в которую включены все хосты перечисленные в файле инвентаризации
- **`ungrouped`** - хосты которые не включены ни в одну группу

Чтобы посмотреть список групп в файле инвентаризации нужно выполнить команду `ansible-inventory --list`, которая покажет все группы и переменные доступные этим группам, либо команду `ansible-inventory --graph`:
```
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
```
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

```
[cluster_servers:children]
master_servers
worker_servers

[all_cluster_servers:children]
cluster_servers
cluster2_servers
```

Для групп можно указать общие переменные, которые актуальны для всех хостов в этой группе. Для этого надо добавить новый новую секцию конфигурации, с названием группы, после после которого указать постфикс`:vars`, например:
```
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
```
ansible all -m shell -a "uptime"

minion2 | CHANGED | rc=0 >>
 13:35:43 up 32 min,  0 users,  load average: 0.02, 0.07, 0.09
minion1 | CHANGED | rc=0 >>
 13:35:43 up 32 min,  0 users,  load average: 0.02, 0.07, 0.09
```

#### shell, command

Можно выполнить любу команду оболочки чрез модуль `shell`, например `ansible all -m shell -a "ls /home/developer"`
Есть еще модуль `command`, он тоже может запускать команды, но не через оболочку, поэтому недоступны переменные окружения оболочки, перенаправления потоков и т.д. (`"*"`, `"<"`, `">"`, `"|"`, `";"` и `"&"`). Пример выплненяи через `command`  `ansible all -m command -a "ls /home/developer"`

#### copy

Модуль `copy` позволяет копировать файлы на удаленные машины. Предварительно необходимо создать в корневой папке проекта файл hi.txt, который мы будем копировать на другие машины (с любым содержжимым внутри).

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
Модуль для управления сервисами, например запустить вебсервер apache и чтобы он автоматически загружался при запуске (при условии что он был ранее установлен) `ansible all -m service -a "name=httpd state=started enabled=yes" -b`, или удалить этот сервис `ansible all -m service -a "name=httpd state=removed" -b

## Шаг 3. Yaml, group_vars, host_vars

### Основы Yaml
YAML — это текстовый формат для хранения данных в структурированном виде. Файлы в формате yaml имеюют расширение либо `.yaml`, либо `.yml`.

Правила оформления YAML
1. Используй отступы для вложенности. Главное — одинаковое количество пробелов на одном уровне.
2. Списки начинаются с `-` и пробела.
3. Пары пишутся в виде `ключ: значение`, обязательно с двоеточием и пробелом.
4. Имя можно указывать как поле (`name: Alexey`) или как ключ (`Alexey:`).  
    Если имя — это ключ, после него обязательно должно быть двоеточие.
5. Можно использовать краткую запись с фигурными скобками:  
    `{name: ..., age: ..., skills: [...]}` или `Имя {age: ..., skills: [...]}`.
6. Для краткой записи списков можно использовать квадратные скобки:  `skills: [Python, C#, YAML]` . Это эквивалентно многострочной записи через `-`.
7. Если в значении есть двоеточие (`:`), ставь кавычки (двойные или одинарные):  `nikname: "Anton: god"` — иначе будет ошибка.

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
- Всё внутри воспринимается буквально — никаких спецсимволов или экранирования.
- Чтобы вставить саму одинарную кавычку, её нужно удвоить:  
    Пример: `'Это ''пример'' текста'` → `Это 'пример' текста`.
- Нельзя делать переносы строк внутри одинарных кавычек.
Двойные кавычки `"..."`
- Позволяют использовать escape-последовательности (как в языках программирования):
    - `\n` — перенос строки
    - `\t` — табуляция
    - `\"` — двойная кавычка внутри строки
- Можно делать многострочные строки с помощью специальных приёмов.
Итог
- Если нужна простая строка без специальных символов — лучше одинарные кавычки.
- Если нужно вставить перенос строки или спецсимволы — двойные.
- Обе кавычки позволяют защищать двоеточия и другие спецсимволы в значениях.

### Переменные в group_vars
В больших проектах,  переменные, относящиеся к хостам рекомендуются хранить в папке `group_vars`. В ней необходимо создать файлы с именем группы, которая указана в файле инвентаризации. 

Вот изначальный файл инвентаризации до переноса переменных в group_vars
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
Подобно тому, как общие для групп переменные рекомендуется хранить в папке `group_vars`, уникальные для отдельных хостов переменные рекомендуется хранить в каталоге `host_vars`, чтобы не "засорять" файл инвентаризации, который по мере роста проекта может стать сложным для чтения и посиска в нем информации. 
Ансибл автоматически умеет находить переменные определенные в этих двух каталогах. 
В каталоге `host_vars` небоходимо содать файл с именем хоста, и в нем в формате yml определить переменные уникальные именно для этого хоста.

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
    ping: # модуль искользуемый в задаче
```

Запустим плейбук: `ansible-playbook playbooks/ping.yml`
```bash
PLAY [Test connection to my servers] ******************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************
ok: [minion1]
ok: [minion2]

TASK [Ping my servers] ********************************************************************************************************************************************************************************
ok: [minion2]
ok: [minion1]

PLAY RECAP ********************************************************************************************************************************************************************************************
minion1                    : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
minion2                    : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

### Второй плейбук - установка веб-сервера nginx

Второй плейбук будет выполнять установку веб-сервера на debian совместимые системы. Для этого в каталоге с плейбуками создаим файл с именем `install_nginx.yml` со следующим содержимым:
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
PLAY [Install Nginx on all servers] ******************************

TASK [Gathering Facts] *************************
ok: [minion2]
ok: [minion1]

TASK [Update apt cache] *****************************
changed: [minion2]
changed: [minion1]

TASK [Install nginx] **********************
changed: [minion2]
changed: [minion1]

TASK [Ensure nginx is running and enabled] ***********************
changed: [minion1]
changed: [minion2]

PLAY RECAP ***************
minion1                    : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
minion2                    : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Можем проверить установку, запросив страничку с вебсервера, как будто бы мы ее открыли в браузере, выполнив для этого команду `curl -l minion1`, и видим ответ, стандартная HTML страница по умолчанию для веб-сервера nginx. Значит nginx установлен и работает. 
```
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
- Один и тот же плейбук можно запускать много раз — результат останется одинаковым.
- Ansible сам проверяет, нужно ли что-то менять (например, не переустанавливает nginx, если он уже установлен).
- Это делает автоматизацию надёжной и безопасной.

Мы можем повторно запустить плейбук установки веб-сервера nginx  `ansible-playbook playbooks/install_nginx.yml` и увидим:
```bash

PLAY [Install Nginx on all servers] ***********************************

TASK [Gathering Facts] **********************
ok: [minion2]
ok: [minion1]

TASK [Update apt cache] *********************
ok: [minion2]
ok: [minion1]

TASK [Install nginx] *********************
ok: [minion1]
ok: [minion2]

TASK [Ensure nginx is running and enabled] *******************************************
ok: [minion2]
ok: [minion1]

PLAY RECAP ************
minion1                    : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
minion2                    : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Если мы сравним этот лог с логом первого запуском плейбука, то заметим разницу, при первом запуске именения применялись (`changed: [имя_хоста]`), а при повтороном уже нет. Вот часть лога первого запуска, где покаано что изменения применились. 
```bash
TASK [Update apt cache] *****************************
changed: [minion2]
changed: [minion1]

TASK [Install nginx] **********************
changed: [minion2]
changed: [minion1]

TASK [Ensure nginx is running and enabled] ***********************
changed: [minion1]
changed: [minion2]
```

Индепонентность даем такие приемущества:
- Экономит время: Ansible пропускает уже выполненные задачи — плейбук работает быстрее.
- Безопасно для продакшена: не перезапустит сервис без причины, не удалит и не перезапишет лишнего.
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

Так же в папке c плейбуками создадим еще один плейбук с имененм `install_nginx_content.yml`:
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

В этом плейбуке мы добавили задачу копирования файла с нашей html страницей, которая заменяет дефолтну страницу nginx. В таске копирования мы используем переменные, которые заранее определили, и в которых задаем откуда и куда копировать страницу. Затем в таске копирования нашей html страницы мы использовали эти переменные. Также, в демонстрационных целях добавили обработчик в секцию `handlers`, который перезапускает веб-сервер, после смены дефолтной страницы. Для этого таск копирвоания должен вызвать этот обработчик через инструкцию `notify: Имя_обработчика`.  
Чтобы не "захламлять" плейбук, обработчики рекомендуется сохранять в предназначенную для этого папку `handlers`, которую рамещают в корне проекта. 

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
# Наименьший приоритет — легко переопределяются в любом другом месте.
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

# playbook.yml — переменная в секции vars
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

Не все еще компоненты ансибл мы пока изучили (такие как роли, регистры и т.д.), но всегда можжно вернуться к этой шпаргалке, чтобы понять в какой очередности переопределяются переменные. Это важно понимать для построения логики работы плейбуков. 

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

Для лучшего понимани работы с переменными, создадим новый плейбук, и назоваем его `varitables.yml` (далее мы более подробно расмотрим некотоыре элементы этого плейбука, такие как `set_facts`, `register` и т.д., пока на них можно не заострять внимание):
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

PLAY RECAP
minion1                    : ok=10   changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
minion2                    : ok=10   changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Не смотря на то что у нас были определенны переменные в инвентарном файле `hosts`, занчения переменных взяты из каталога `host_vars`:
```
ok: [minion1] => {  
    "my_new_varitable": "Hello World from Alex form host_vars"  
}  
ok: [minion2] => {  
    "my_new_varitable": "Hello World from Maxim form host_vars"  
}  
```
### Модуль set_fact

Далее мы используем `set_fact` — это модуль в Ansible, который позволяет создавать переменные во время выполнения (runtime). Эти переменные сохраняются только в текущем контексте (для хоста) и могут использоваться в следующих задачах.
Еще раз, что важно запомить :
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
```
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
  - `ansible_hostname` — имя хоста
  - `ansible_fqdn` — полное доменное имя
  - `ansible_distribution` — дистрибутив (например, Ubuntu)
  - `ansible_distribution_version` — версия (например, 20.04)
  - `ansible_architecture` — архитектура (например, x86_64)
  - `ansible_os_family` — семейство ОС (Debian, RedHat)
- Сеть:
  - `ansible_default_ipv4.address` — основной IP-адрес
  - `ansible_all_ipv4_addresses` — все IPv4
  - `ansible_interfaces` — список сетевых интерфейсов
  - `ansible_eth0.ipv4.address` — IP интерфейса eth0
- Процессор / Память:
  - `ansible_processor_cores` — количество ядер
  - `ansible_processor` — список CPU
  - `ansible_memtotal_mb` — всего памяти (в МБ)
  - `ansible_memfree_mb` — свободной памяти
- Диски:
  - `ansible_mounts` — список точек монтирования
  - `ansible_mounts.0.size_total` — размер первого раздела

В нашем примере мы вывели на экран объем свободной памяти в мегабайтах
```
TASK [debug]
ok: [minion1] => {
    "ansible_memfree_mb": 5310
}
ok: [minion2] => {
    "ansible_memfree_mb": 5310
}
```

### Директива register

`register` в Ansible — это директива, которая сохраняет результат выполнения задачи (обычно shell-команды) в переменную.
В нашем примере мы выполнили команду `uptime` через модуль `shell`, и сохранили результат в переменную `uptime_result`
```
- name: Run uptime and save output
  shell: uptime
  register: uptime_result
```
и как увидели при запуске плейбука, в uptime_result стала доступна структура с полями:
- stdout — стандартный вывод
- stderr — ошибки
- rc — код возврата
- changed — был ли выполнен changed
- stdout_lines — список строк из stdout
То есть `register` — это способ сохранить вывод команды и использовать его позже.

## Шаг 6. Шаблонизатор Jinja2, условия (when) и блоки (block)

### Шаблонизатор Jinja2

Jinja2 — это шаблонизатор, используемый в Ansible для вставки переменных, условий и циклов в YAML-файлы.
Используется для:
- Определения переменных:
  - {{ variable_name }}
- Фильтров:
  - {{ list | length }}
  - {{ name | upper }}
- Условий:
  - {% if var == "value" %}...{% endif %}
- Циклов:
  - {% for item in list %}...{% endfor %}
- Тестов:
  - {{ var is defined }}
  - {{ var is not none }}

Наиболее частые примеры использования
```yaml
when: my_var == "value"  # Выполнить задачу только если переменная my_var равна "value"
with_items: "{{ my_list }}"  # Перебрать элементы из списка my_list (устаревший способ, но до сих пор используется в старом коде)
loop: "{{ my_list }}"  # Современный способ организации цикла по элементам списка my_list (предпочтительно использовать вместо with_items)
```
Работает внутри {{ ... }} или {% ... %} — шаблонов.


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

TASK [Install nginx on Debian-like systems] 
skipping: [minion3]
changed: [minion2]
changed: [minion1]

TASK [Install EPEL repository] 
skipping: [minion1]
skipping: [minion2]
ok: [minion3]

TASK [Install nginx on RedHat-like systems] 
skipping: [minion1]
skipping: [minion2]
changed: [minion3]

TASK [Copy custom index.html to Nginx web root on Debian-like systems] 
skipping: [minion3]
ok: [minion1]
ok: [minion2]

TASK [Copy custom index.html to Nginx web root on RedHat-like systems] 
skipping: [minion1]
skipping: [minion2]
changed: [minion3]
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
        when: ansible_os_family == "RedHat" 

      - name:  Reload or start nginx in container 
        shell: |
          nginx -t && nginx -s reload || nginx
        ignore_errors: true
      when: ansible_os_family == "RedHat"
```
Код плейбука получился более комактным и выразительным. 

