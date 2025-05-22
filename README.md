Выполнил Зинченко А.С. в рамках 3 домашней работы

# Содержание
1. [Выполнение пунктов задания](#выполнение-пунктов-задания)
2. [Архитектура решения](#архитектура-решения)
3. [Обоснование использования ACL](#обоснование-использования-acl)
4. [Верификация работы](#верификация-работы)
5. [Логи выполнения](#логи-выполнения)

# Выполнение пунктов задания

### 1. Создание пользователя Docker и системных ресурсов
**Реализация:**
- Группа `docker` создается с GID 1001
- Пользователь `docker` (UID 1001) с домашней директорией `/home/docker`
- Директория `/mnt/docker` с правами 0755 и владельцем `docker:docker`

**Ключевые модули:**
```
- name: Create docker group
  ansible.builtin.group:
    name: "{{ docker_user }}"

- name: Create docker storage directory
  ansible.builtin.file:
    path: /mnt/docker
    owner: "{{ docker_user }}"
```

### 2. Работа с файлом table
**Особенности реализации:**
- Файл создается в домашней директории текущего пользователя (`/home/ubuntu/table`)
- Использование многострочного контента с сохранением форматирования
- Поиск строк с буквой 'o' через awk с регистрацией вывода

**Пример вывода:**
```
ok: [lest_host] => {
    "awk_output.stdout_lines": [
        "2 loyal",
        "3 famous",
        "4 absorb",
        "8 about",
        "9 offer",
        "10 topple",
        "14 evoke"
    ]
}
```

### 3. Управление пользователем test
**Особенности реализации:**
- Генерация 16-символьного пароля через `pwgen -s 16 1`
- Создание одноименной группы (GID 1002)
- Хеширование пароля с использованием `sha512_crypt`
- Безопасное удаление файла через временные ACL-права

**Основные задачи:**
```
- name: Set temporary permissions
  ansible.posix.acl:
    path: /home/{{ ansible_user }}
    entity: "{{ temp_user }}"
    permissions: rwx
```

# Структура Playbook

```
.
├── ansible.cfg
├── host_vars
│   └── lest_host
│       └── vault.yml     # Зашифрованные параметры подключения 
├── inventory
│   └── inventory.yaml
├── playbook.yaml
└── roles
    └── main
        ├── defaults
        │   └── main.yml  # docker_user: docker, temp_user: test
        └── tasks
            └── main.yml  # Основные задачи
```

# Обоснование использования ACL

### Проблема
При выполнении задач от имени пользователя `test` возникали ошибки:
```
chmod: invalid mode: ‘A+user:test:rx:allow’
```

### Причина
- Ansible создает временные файлы в `~/.ansible/tmp`
- Пользователь `test` не имел прав на запись в `/home/ubuntu`

### Решение
1. Временное предоставление прав через ACL:
```
setfacl -m u:test:rwx /home/ubuntu
```
2. Последовательное выполнение:
   - Установка прав
   - Удаление файла
   - Отзыв прав

# Итог

### Критерии успеха
1. Все задачи завершены с `ok` или `changed`
2. Отсутствие failed в PLAY RECAP
3. Файл `/home/ubuntu/table` отсутствует

**Результат:**

На основе [лог файла из пайплана #15](https://github.com/astrekoi/lesta-hw-2/actions/runs/15182127783)

```
PLAY RECAP
lest_host : ok=19 changed=7 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

# Логи выполнения

Полный лог выполнения доступен [-> тут](https://github.com/astrekoi/lesta-hw-2/actions/runs/15182127783/artifacts/3175240325)

**Ключевые моменты лога:**
1. Успешное создание пользователей:
   ```
   TASK [main : Provision docker user] - changed
   TASK [main : Provision test user] - changed
   ```
2. Работа с ACL:
   ```
   TASK [main : Set temporary permissions] - changed
   TASK [main : Cleanup permissions] - changed
   ```
3. Корректное удаление файла:
   ```
   TASK [main : Remove target file] - changed
   TASK [main : Show deletion result] - "msg": "File successfully deleted"
   ```