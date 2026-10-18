# Лабораторная работа №2. Ansible + Caddy

**Выполнили:** Мкртчян Карина, Гриценко Ярослав


**Среда выполнения:** Windows 11 + WSL2 (Ubuntu)

---

## Введение

Цель работы — освоить базовые возможности Ansible (ad-hoc команды и плейбуки) и настроить веб-сервер Caddy на целевом хосте.
Работа выполнялась в среде WSL2 (Ubuntu) на Windows, так как Ansible не поддерживает нативный запуск на Windows - управляющая машина (control node) должна быть UNIX-подобной системой.

---

## 1. Подготовка окружения

### 1.1. Установка WSL и Ubuntu

В Windows была установлена подсистема WSL2 с дистрибутивом Ubuntu. Проверка:

```bash
wsl --list --verbose
```

```text
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

Затем внутри Ubuntu были установлены базовые пакеты:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-pip python3-venv curl git pipx
```

### 1.2. Установка Ansible

Ansible установлен через pipx (pipx автоматически создает изолированное виртуальное окружение для программы,
избавляя от конфликтов зависимостей и ограничений современных ОС)

```bash
pipx ensurepath
pipx install --include-deps ansible
ansible --version
```

```text
ansible [core 2.21.x]
  config file = None
  ...
  python version = 3.14.x
```

<img width="1467" height="281" alt="image" src="https://github.com/user-attachments/assets/25b76462-d8ba-4486-b200-745ba0251c14" />

### 1.3. Создание рабочей директории и конфига

```bash
mkdir -p ~/ansible/inventory
cd ~/ansible
```

Файл `ansible.cfg`:

```ini
[defaults]
host_key_checking = false
inventory = inventory/hosts
```

Файл `inventory/hosts`:

```ini
[my_servers]
local_server ansible_host=localhost ansible_connection=local
```

Проверка связи:

```bash
ansible my_servers -m ping
```

```text
local_server | SUCCESS => {
    "ansible_facts": {...},
    "changed": false,
    "ping": "ping"
}
```

<img width="1102" height="291" alt="image" src="https://github.com/user-attachments/assets/50ed180a-58f5-4820-91f3-0c2c1e552e98" />

---

## 2. Задание 1. Плейбук для операций с файлом

### 2.1. Постановка задачи

Требовалось переписать пример из шага 5 Части 1 (ad-hoc команды создания и удаления файла) в формат плейбука, а также **добавить четвёртый шаг — перед удалением изменить содержимое файла** на любое другое.

### 2.2. Текст плейбука (`task1_file_ops.yml`)

```yaml
---
- name: File operations demo (create, modify, delete)
  hosts: my_servers
  connection: local
  tasks:

    - name: 1. Create test file with initial content
      shell: echo "test_file_content" > $HOME/test.txt
      args:
        creates: /home/karina/test.txt

    - name: 2. Show current content
      command: cat /home/karina/test.txt
      register: content_before
      changed_when: false

    - name: Print content before modification
      debug:
        var: content_before.stdout

    - name: 3. Modify file content
      shell: echo "modified_content_by_ansible" > $HOME/test.txt

    - name: 4. Show modified content
      command: cat /home/karina/test.txt
      register: content_after
      changed_when: false

    - name: Print content after modification
      debug:
        var: content_after.stdout

    - name: 5. Remove the test file
      file:
        path: /home/karina/test.txt
        state: absent
```

### 2.3. Что делает плейбук

| Шаг | Модуль | Действие |
|-----|--------|----------|
| 1 | `shell` | Создаёт `~/test.txt` с содержимым `test_file_content` |
| 2 | `command` + `register` | Читает содержимое файла в переменную `content_before` |
| 3 | `debug` | Выводит содержимое до изменения |
| 4 | `shell` | Перезаписывает файл содержимым `modified_content_by_ansible` (четвёртый шаг из задания) |
| 5 | `command` + `register` | Читает новое содержимое в `content_after` |
| 6 | `debug` | Выводит содержимое после изменения |
| 7 | `file` | Удаляет файл (`state: absent`) |

### 2.4. Запуск и результат

```bash
ansible-playbook task1_file_ops.yml
```

Вывод:

```text
PLAY [File operations demo (create, modify, delete)] ***

TASK [Gathering Facts] ***
ok: [local_server]

TASK [1. Create test file with initial content] ***
changed: [local_server]

TASK [2. Show current content] ***
ok: [local_server]

TASK [Print content before modification] ***
ok: [local_server] => {
    "content_before.stdout": "test_file_content"
}

TASK [3. Modify file content] ***
changed: [local_server]

TASK [4. Show modified content] ***
ok: [local_server]

TASK [Print content after modification] ***
ok: [local_server] => {
    "content_after.stdout": "modified_content_by_ansible"
}

TASK [5. Remove the test file] ***
changed: [local_server]

PLAY RECAP ***
local_server : ok=8 changed=3 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

<img width="1120" height="991" alt="image" src="https://github.com/user-attachments/assets/45848e89-5034-4a9b-bb1c-b87bcd105fa6" />


<img width="1103" height="200" alt="image" src="https://github.com/user-attachments/assets/e6ff93c5-0825-42b5-b2ee-1c48adf34a27" />

Проверка, что файл удалён:

```bash
ls ~/test.txt
```

```text
ls: cannot access '/home/karina/test.txt': No such file or directory
```

<img width="835" height="60" alt="image" src="https://github.com/user-attachments/assets/3255fd62-89bb-4c2f-92fc-4cb1d1347f13" />


### 2.5. Вывод по Заданию 1

Плейбук успешно выполняет все требуемые операции: создание файла, чтение исходного содержимого, изменение содержимого (добавленный четвёртый шаг), чтение нового содержимого и удаление файла. Все шаги идемпотентны (повторный запуск не сломает систему), а вывод `PLAY RECAP` подтверждает отсутствие ошибок (`failed=0`).

---

## 3. Задание 2.

## 3. Задание 2 (вариант a). Расширение Caddy: своя index.html

### 3.1. Постановка задачи

Требовалось расширить конфигурацию веб-сервера Caddy любым функционалом по желанию. Был выбран вариант **замены дефолтной страницы Caddy на собственную `index.html` с Hello world**, развёртываемую через Ansible.

### 3.2. Структура роли

Роль создана командой `ansible-galaxy init caddy_deploy` и имеет вид:

```text
roles/caddy_deploy/
├── tasks/main.yml
├── handlers/main.yml
├── templates/Caddyfile.j2
└── files/index.html
```

### 3.3. Шаблон Caddyfile (`templates/Caddyfile.j2`)

```caddy
:80 {
    root * /var/www/html
    file_server
}
```

### 3.4. Своя страница (`files/index.html`)

```html
<!DOCTYPE html>
<html>
<head>
    <title>My Caddy Page</title>
</head>
<body>
    <h1>Hello world from Karina!</h1>
    <p>This page was deployed by Ansible.</p>
</body>
</html>
```

### 3.5. Задачи роли (`tasks/main.yml`, ключевые шаги)

Помимо установки Caddy (добавление GPG-ключа и репозитория, `apt install caddy`), роль содержит шаги настройки:

```yaml
- name: Create config file from template
  template:
    src: Caddyfile.j2
    dest: /etc/caddy/Caddyfile
  become: true
  notify: Reload caddy

- name: Ensure web root directory exists
  file:
    path: /var/www/html
    state: directory
    mode: '0755'
  become: true

- name: Copy custom index.html
  copy:
    src: index.html
    dest: /var/www/html/index.html
  become: true

- name: Ensure caddy is started and enabled
  service:
    name: caddy
    state: started
    enabled: yes
  become: true
```

Handler (`handlers/main.yml`):

```yaml
- name: Reload caddy
  service:
    name: caddy
    state: reloaded
  become: true
```

### 3.6. Основной плейбук (`caddy_deploy.yml`)

```yaml
---
- name: Install and configure Caddy webserver
  hosts: my_servers
  connection: local
  become: true
  become_method: sudo
  become_exe: /usr/bin/sudo.ws
  roles:
    - caddy_deploy
```

### 3.7. Запуск и результат

```bash
ansible-playbook caddy_deploy.yml --ask-become-pass
```

```text
PLAY RECAP ***
local_server : ok=10 changed=2 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

<img width="1122" height="1011" alt="image" src="https://github.com/user-attachments/assets/2b6cc72e-0116-4131-af96-f74183b726a5" />


Проверка конфига и файла:

```bash
cat /etc/caddy/Caddyfile
ls -la /var/www/html/
```

```text
:80 {
    root * /var/www/html
    file_server
}

-rw-r--r-- 1 root root 178 Oct 19 00:48 index.html
```

<img width="760" height="275" alt="image" src="https://github.com/user-attachments/assets/35082ea8-1cce-4eee-a765-b645da2edb7a" />


Проверка отдачи страницы:

```bash
curl http://localhost
```

```html
<!DOCTYPE html>
<html>
<head>
    <title>My Caddy Page</title>
</head>
<body>
    <h1>Hello world from Karina!</h1>
    <p>This page was deployed by Ansible.</p>
</body>
</html>
```

<img width="701" height="265" alt="image" src="https://github.com/user-attachments/assets/4e19ddd4-446a-4369-8e86-2baf938865a6" />


Также страница открывается в браузере Windows по адресу `http://localhost` (WSL пробрасывает порт 80 на хост):

<img width="585" height="325" alt="image" src="https://github.com/user-attachments/assets/f2b0ca51-15d5-4c9f-a1d1-30465638ea8c" />


### 3.8. Вывод по Заданию 2

Caddy успешно установлен и настроен через Ansible. Дефолтная страница заменена на собственную `index.html`, которая раздаётся из `/var/www/html` согласно шаблону `Caddyfile`. Конфигурация идемпотентна: повторный запуск плейбука не вносит изменений (`changed=0`), что подтверждает корректность роли.

---

## 4. Часть 3 (адаптация). Локальный HTTPS и доменное имя

### 4.1. Постановка задачи и ограничения

В оригинальной части 3 предлагается зарегистрировать домен (например, на DuckDNS) и настроить Caddy так, чтобы он автоматически получил HTTPS-сертификат от Let's Encrypt. Однако лабораторная работа выполнялась в среде **WSL2 на локальном компьютере**, которая не имеет публичного IP-адреса. Let's Encrypt не может провести проверку домена (HTTP-01 challenge), поэтому получить настоящий сертификат в таких условиях невозможно.

В качестве **локальной адаптации** части 3 было решено:
1. Использовать доменное имя `karina.localhost`, которое браузеры автоматически резолвят в `127.0.0.1` без настройки DNS или файла `hosts`.
2. Настроить Caddy на выпуск сертификата через **внутренний CA (Certificate Authority)** самого Caddy с помощью директивы `tls internal`.
3. Установить корневой сертификат Caddy в доверенное хранилище Windows, чтобы браузер не показывал предупреждение о небезопасном соединении.

### 4.2. Изменение шаблона Caddyfile

Файл `roles/caddy_deploy/templates/Caddyfile.j2` был приведён к виду:

```caddy
karina.localhost {
    root * /var/www/html
    file_server
    tls internal
}
```

Директива `tls internal` указывает Caddy использовать собственный внутренний CA вместо обращения к Let's Encrypt.

### 4.3. Доверие к сертификату

После запуска плейбука и генерации сертификата была выполнена команда:

```bash
sudo caddy trust
```

Она устанавливает корневой сертификат Caddy в доверенное хранилище **Linux (WSL)**.

Однако браузер работает в **Windows** и использует хранилище сертификатов Windows. Поэтому корневой сертификат Caddy был скопирован из WSL в Windows:

```bash
sudo cp /var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt /mnt/c/Users/User/Desktop/caddy-root.crt
```

Затем файл `caddy-root.crt` был импортирован в хранилище **«Доверенные корневые центры сертификации»** Windows (двойной клик → «Установить сертификат» → «Локальный компьютер» → «Поместить все сертификаты в следующее хранилище» → «Доверенные корневые центры сертификации»).

### 4.4. Результат

После импорта сертификата страница открывается в браузере Windows по адресу:

```
https://karina.localhost
```

Отображается собственная страница `Hello world from Karina!`, а соединение помечено как защищённое.

<img width="680" height="295" alt="image" src="https://github.com/user-attachments/assets/e5973650-fbf0-4215-bc08-9b8bfa41204b" />

### 4.5. Вывод по части 3

В условиях локальной среды WSL невозможно получить сертификат Let's Encrypt из-за отсутствия публичного IP. Однако удалось **воспроизвести полный цикл настройки HTTPS**: доменное имя, автоматический выпуск сертификата Caddy через внутренний CA, установка доверия к сертификату в браузере. Это демонстрирует понимание механизмов работы Caddy с TLS и позволяет в дальнейшем без изменений перенести конфигурацию на публичный сервер (заменив `tls internal` на автоматический HTTPS от Let's Encrypt).

---


## Приложение. Полный список файлов проекта

```text
~/ansible/
├── ansible.cfg
├── inventory/
│   └── hosts
├── task1_file_ops.yml
├── caddy_deploy.yml
└── roles/
    └── caddy_deploy/
        ├── tasks/main.yml
        ├── handlers/main.yml
        ├── templates/Caddyfile.j2
        └── files/index.html
```
