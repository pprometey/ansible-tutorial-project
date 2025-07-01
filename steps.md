# Шаги по изучению

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

### Основынек модули ansible 

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

Модуль `copy` позволяет копировать файлы на удаленные машины. 
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
Модуль для управления сервисами, например запустить вебсервер apache и чтобы он автоматически загружался при запуске (при условии что он был ранее установлен) `ansible all -m service -a "name=httpd state=started enabled=yes" -b`, или удалить этот сервис `ansible all -m service -a "name=httpd state=removed" -b`

