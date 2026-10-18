# Лабораторная работа №2. Ansible + Caddy

**Выполнили:** Мкртчян Карина, Гриценко Ярослав
**Среда выполнения:** Windows 11 + WSL2 (Ubuntu)

---

## Введение

Цель работы — освоить базовые возможности Ansible (ad-hoc команды и плейбуки) и настроить веб-сервер Caddy на целевом хосте.
Работа выполнялась в среде WSL2 (Ubuntu) на Windows, так как Ansible не поддерживает нативный запуск на Windows 
- управляющая машина (control node) должна быть UNIX-подобной системой.

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

## 3. Задание 2. [заполним после выполнения]


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
```
