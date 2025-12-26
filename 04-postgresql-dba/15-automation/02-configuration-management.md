# Configuration Management для PostgreSQL

## Введение

Configuration Management (управление конфигурацией) - это практика автоматизации настройки и поддержания состояния серверов и приложений. Для PostgreSQL это означает автоматическую установку, настройку, и управление кластерами баз данных в согласованном и воспроизводимом состоянии.

## Основные инструменты

### Сравнение инструментов

| Характеристика | Ansible | Puppet | Chef | SaltStack |
|----------------|---------|--------|------|-----------|
| Архитектура | Agentless | Agent-based | Agent-based | Agent/Agentless |
| Язык | YAML | DSL (Ruby) | Ruby | YAML/Python |
| Кривая обучения | Низкая | Средняя | Высокая | Средняя |
| Push/Pull | Push | Pull | Pull | Push/Pull |
| Масштабируемость | Средняя | Высокая | Высокая | Высокая |

## Ansible для PostgreSQL

### Установка Ansible

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install ansible

# macOS
brew install ansible

# pip
pip install ansible

# Проверка версии
ansible --version
```

### Структура Ansible проекта

```
postgresql-automation/
├── ansible.cfg
├── inventory/
│   ├── production
│   └── staging
├── group_vars/
│   ├── all.yml
│   ├── postgresql_primary.yml
│   └── postgresql_replica.yml
├── host_vars/
│   └── db1.example.com.yml
├── roles/
│   └── postgresql/
│       ├── tasks/
│       ├── handlers/
│       ├── templates/
│       ├── files/
│       ├── vars/
│       └── defaults/
└── playbooks/
    ├── install_postgresql.yml
    ├── configure_postgresql.yml
    └── setup_replication.yml
```

### Inventory файл

```ini
# inventory/production

[postgresql_primary]
db-primary.example.com ansible_host=10.0.1.10

[postgresql_replica]
db-replica1.example.com ansible_host=10.0.1.11
db-replica2.example.com ansible_host=10.0.1.12

[postgresql:children]
postgresql_primary
postgresql_replica

[postgresql:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/db_key
postgresql_version=15
```

### Роль установки PostgreSQL

```yaml
# roles/postgresql/tasks/main.yml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- name: Install PostgreSQL repository
  include_tasks: install_repo.yml
  when: postgresql_install_repo | default(true)

- name: Install PostgreSQL packages
  package:
    name: "{{ postgresql_packages }}"
    state: present
  notify: Restart PostgreSQL

- name: Ensure PostgreSQL data directory exists
  file:
    path: "{{ postgresql_data_dir }}"
    state: directory
    owner: postgres
    group: postgres
    mode: '0700'

- name: Initialize PostgreSQL database
  command: "{{ postgresql_bin_dir }}/initdb -D {{ postgresql_data_dir }}"
  args:
    creates: "{{ postgresql_data_dir }}/PG_VERSION"
  become: true
  become_user: postgres
  when: postgresql_init_db | default(true)

- name: Configure PostgreSQL
  template:
    src: postgresql.conf.j2
    dest: "{{ postgresql_config_dir }}/postgresql.conf"
    owner: postgres
    group: postgres
    mode: '0644'
  notify: Reload PostgreSQL

- name: Configure pg_hba.conf
  template:
    src: pg_hba.conf.j2
    dest: "{{ postgresql_config_dir }}/pg_hba.conf"
    owner: postgres
    group: postgres
    mode: '0640'
  notify: Reload PostgreSQL

- name: Ensure PostgreSQL is started and enabled
  service:
    name: "{{ postgresql_service_name }}"
    state: started
    enabled: true
```

### Шаблон postgresql.conf

```jinja2
# roles/postgresql/templates/postgresql.conf.j2
# PostgreSQL configuration file
# Managed by Ansible - DO NOT EDIT MANUALLY

#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

listen_addresses = '{{ postgresql_listen_addresses | default("localhost") }}'
port = {{ postgresql_port | default(5432) }}
max_connections = {{ postgresql_max_connections | default(100) }}

#------------------------------------------------------------------------------
# RESOURCE USAGE (except WAL)
#------------------------------------------------------------------------------

shared_buffers = {{ postgresql_shared_buffers | default('128MB') }}
effective_cache_size = {{ postgresql_effective_cache_size | default('4GB') }}
maintenance_work_mem = {{ postgresql_maintenance_work_mem | default('64MB') }}
work_mem = {{ postgresql_work_mem | default('4MB') }}

#------------------------------------------------------------------------------
# WRITE-AHEAD LOG
#------------------------------------------------------------------------------

wal_level = {{ postgresql_wal_level | default('replica') }}
max_wal_size = {{ postgresql_max_wal_size | default('1GB') }}
min_wal_size = {{ postgresql_min_wal_size | default('80MB') }}

{% if postgresql_archive_mode | default(false) %}
archive_mode = on
archive_command = '{{ postgresql_archive_command }}'
{% endif %}

#------------------------------------------------------------------------------
# REPLICATION
#------------------------------------------------------------------------------

{% if postgresql_role == 'primary' %}
max_wal_senders = {{ postgresql_max_wal_senders | default(10) }}
wal_keep_size = {{ postgresql_wal_keep_size | default('1GB') }}
{% endif %}

{% if postgresql_role == 'replica' %}
hot_standby = on
{% endif %}

#------------------------------------------------------------------------------
# LOGGING
#------------------------------------------------------------------------------

logging_collector = on
log_directory = '{{ postgresql_log_directory | default("log") }}'
log_filename = '{{ postgresql_log_filename | default("postgresql-%Y-%m-%d_%H%M%S.log") }}'
log_statement = '{{ postgresql_log_statement | default("none") }}'
log_min_duration_statement = {{ postgresql_log_min_duration_statement | default(-1) }}

#------------------------------------------------------------------------------
# CUSTOM SETTINGS
#------------------------------------------------------------------------------

{% for key, value in postgresql_custom_settings.items() | default({}) %}
{{ key }} = {{ value }}
{% endfor %}
```

### Шаблон pg_hba.conf

```jinja2
# roles/postgresql/templates/pg_hba.conf.j2
# PostgreSQL Client Authentication Configuration File
# Managed by Ansible - DO NOT EDIT MANUALLY

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections
local   all             postgres                                peer
local   all             all                                     peer

# IPv4 local connections
host    all             all             127.0.0.1/32            scram-sha-256

# IPv6 local connections
host    all             all             ::1/128                 scram-sha-256

{% if postgresql_hba_entries is defined %}
# Custom entries
{% for entry in postgresql_hba_entries %}
{{ entry.type }}    {{ entry.database }}    {{ entry.user }}    {{ entry.address }}    {{ entry.method }}
{% endfor %}
{% endif %}

{% if postgresql_role == 'primary' %}
# Replication connections
host    replication     {{ postgresql_replication_user | default('replicator') }}    {{ postgresql_replication_network | default('10.0.0.0/8') }}    scram-sha-256
{% endif %}
```

### Handlers

```yaml
# roles/postgresql/handlers/main.yml
---
- name: Restart PostgreSQL
  service:
    name: "{{ postgresql_service_name }}"
    state: restarted

- name: Reload PostgreSQL
  service:
    name: "{{ postgresql_service_name }}"
    state: reloaded
```

### Playbook установки

```yaml
# playbooks/install_postgresql.yml
---
- name: Install and configure PostgreSQL
  hosts: postgresql
  become: true

  vars_files:
    - ../group_vars/all.yml

  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"

  roles:
    - role: postgresql
      tags: postgresql

  post_tasks:
    - name: Create application database
      postgresql_db:
        name: "{{ item.name }}"
        encoding: UTF-8
        lc_collate: en_US.UTF-8
        lc_ctype: en_US.UTF-8
        template: template0
        state: present
      become_user: postgres
      loop: "{{ postgresql_databases | default([]) }}"

    - name: Create application users
      postgresql_user:
        name: "{{ item.name }}"
        password: "{{ item.password }}"
        role_attr_flags: "{{ item.role_attr_flags | default('') }}"
        state: present
      become_user: postgres
      loop: "{{ postgresql_users | default([]) }}"
      no_log: true

    - name: Grant privileges
      postgresql_privs:
        database: "{{ item.database }}"
        roles: "{{ item.user }}"
        privs: "{{ item.privs }}"
        objs: "{{ item.objs | default('ALL_IN_SCHEMA') }}"
        type: "{{ item.type | default('table') }}"
        state: present
      become_user: postgres
      loop: "{{ postgresql_privileges | default([]) }}"
```

### Настройка репликации

```yaml
# playbooks/setup_replication.yml
---
- name: Configure Primary for replication
  hosts: postgresql_primary
  become: true

  tasks:
    - name: Create replication user
      postgresql_user:
        name: "{{ postgresql_replication_user }}"
        password: "{{ postgresql_replication_password }}"
        role_attr_flags: REPLICATION,LOGIN
        state: present
      become_user: postgres

    - name: Ensure replication slot exists
      postgresql_slot:
        name: "{{ item }}"
        slot_type: physical
        state: present
      become_user: postgres
      loop: "{{ groups['postgresql_replica'] | map('replace', '.', '_') | list }}"

- name: Configure Replicas
  hosts: postgresql_replica
  become: true
  serial: 1

  tasks:
    - name: Stop PostgreSQL on replica
      service:
        name: "{{ postgresql_service_name }}"
        state: stopped

    - name: Remove existing data directory
      file:
        path: "{{ postgresql_data_dir }}"
        state: absent

    - name: Create base backup from primary
      command: >
        pg_basebackup -h {{ hostvars[groups['postgresql_primary'][0]]['ansible_host'] }}
        -D {{ postgresql_data_dir }}
        -U {{ postgresql_replication_user }}
        -P -R -X stream -C -S {{ inventory_hostname | replace('.', '_') }}
      become_user: postgres
      environment:
        PGPASSWORD: "{{ postgresql_replication_password }}"

    - name: Set correct permissions on data directory
      file:
        path: "{{ postgresql_data_dir }}"
        owner: postgres
        group: postgres
        mode: '0700'
        recurse: yes

    - name: Start PostgreSQL on replica
      service:
        name: "{{ postgresql_service_name }}"
        state: started
```

### Переменные

```yaml
# group_vars/all.yml
---
postgresql_version: 15
postgresql_service_name: postgresql
postgresql_port: 5432
postgresql_listen_addresses: '*'

postgresql_shared_buffers: '256MB'
postgresql_effective_cache_size: '1GB'
postgresql_work_mem: '8MB'
postgresql_maintenance_work_mem: '128MB'

postgresql_max_connections: 200
postgresql_wal_level: replica

postgresql_databases:
  - name: production_db
  - name: staging_db

postgresql_users:
  - name: app_user
    password: "{{ vault_app_user_password }}"
  - name: readonly_user
    password: "{{ vault_readonly_password }}"
    role_attr_flags: "NOSUPERUSER,NOCREATEDB"

postgresql_privileges:
  - database: production_db
    user: app_user
    privs: ALL
    type: database
  - database: production_db
    user: readonly_user
    privs: SELECT
    objs: ALL_IN_SCHEMA

postgresql_hba_entries:
  - type: host
    database: all
    user: all
    address: 10.0.0.0/8
    method: scram-sha-256

# group_vars/postgresql_primary.yml
---
postgresql_role: primary
postgresql_replication_user: replicator
postgresql_replication_password: "{{ vault_replication_password }}"
postgresql_replication_network: 10.0.0.0/8

# group_vars/postgresql_replica.yml
---
postgresql_role: replica
```

## Puppet для PostgreSQL

### Установка Puppet

```bash
# Ubuntu/Debian
wget https://apt.puppet.com/puppet8-release-focal.deb
sudo dpkg -i puppet8-release-focal.deb
sudo apt update
sudo apt install puppet-agent

# Puppet module для PostgreSQL
puppet module install puppetlabs-postgresql
```

### Манифест установки PostgreSQL

```puppet
# manifests/postgresql.pp

class profile::postgresql (
  String $version = '15',
  String $listen_address = '*',
  Integer $port = 5432,
  String $encoding = 'UTF8',
  Hash $databases = {},
  Hash $users = {},
) {

  # Установка PostgreSQL
  class { 'postgresql::globals':
    version             => $version,
    manage_package_repo => true,
    encoding            => $encoding,
  }

  class { 'postgresql::server':
    listen_addresses => $listen_address,
    port             => $port,
    ipv4acls         => [
      'host all all 10.0.0.0/8 scram-sha-256',
    ],
  }

  # Создание баз данных
  $databases.each |String $db_name, Hash $db_params| {
    postgresql::server::db { $db_name:
      user     => $db_params['user'],
      password => postgresql::postgresql_password($db_params['user'], $db_params['password']),
      encoding => $db_params['encoding'],
      locale   => $db_params['locale'],
    }
  }

  # Создание пользователей
  $users.each |String $user_name, Hash $user_params| {
    postgresql::server::role { $user_name:
      password_hash => postgresql::postgresql_password($user_name, $user_params['password']),
      superuser     => $user_params['superuser'],
      createdb      => $user_params['createdb'],
      createrole    => $user_params['createrole'],
      login         => $user_params['login'],
    }
  }

  # Настройка параметров
  postgresql::server::config_entry { 'shared_buffers':
    value => '256MB',
  }

  postgresql::server::config_entry { 'effective_cache_size':
    value => '1GB',
  }

  postgresql::server::config_entry { 'work_mem':
    value => '8MB',
  }

  postgresql::server::config_entry { 'max_connections':
    value => '200',
  }
}
```

### Hiera данные

```yaml
# data/common.yaml
---
profile::postgresql::version: '15'
profile::postgresql::listen_address: '*'
profile::postgresql::port: 5432

profile::postgresql::databases:
  production_db:
    user: app_user
    password: ENC[PKCS7,encrypted_password]
    encoding: UTF8
    locale: en_US.UTF-8

profile::postgresql::users:
  app_user:
    password: ENC[PKCS7,encrypted_password]
    superuser: false
    createdb: false
    createrole: false
    login: true
  admin_user:
    password: ENC[PKCS7,encrypted_password]
    superuser: true
    createdb: true
    createrole: true
    login: true
```

### Манифест репликации

```puppet
# manifests/postgresql_replication.pp

class profile::postgresql::primary (
  String $replication_user = 'replicator',
  String $replication_password,
  String $replication_network = '10.0.0.0/8',
) {

  include profile::postgresql

  # Создание пользователя репликации
  postgresql::server::role { $replication_user:
    password_hash => postgresql::postgresql_password($replication_user, $replication_password),
    replication   => true,
  }

  # Настройка pg_hba для репликации
  postgresql::server::pg_hba_rule { 'allow replication connections':
    type        => 'host',
    database    => 'replication',
    user        => $replication_user,
    address     => $replication_network,
    auth_method => 'scram-sha-256',
    order       => 50,
  }

  # Настройка WAL
  postgresql::server::config_entry { 'wal_level':
    value => 'replica',
  }

  postgresql::server::config_entry { 'max_wal_senders':
    value => '10',
  }

  postgresql::server::config_entry { 'wal_keep_size':
    value => '1GB',
  }
}

class profile::postgresql::replica (
  String $primary_host,
  String $replication_user = 'replicator',
  String $replication_password,
) {

  class { 'postgresql::server':
    manage_recovery_conf => true,
  }

  postgresql::server::config_entry { 'hot_standby':
    value => 'on',
  }

  # Recovery configuration
  postgresql::server::recovery { 'replica':
    standby_mode     => 'on',
    primary_conninfo => "host=${primary_host} user=${replication_user} password=${replication_password}",
    trigger_file     => '/tmp/postgresql.trigger',
  }
}
```

## Chef для PostgreSQL

### Cookbook структура

```
postgresql-cookbook/
├── Berksfile
├── metadata.rb
├── recipes/
│   ├── default.rb
│   ├── server.rb
│   ├── client.rb
│   └── replication.rb
├── attributes/
│   └── default.rb
├── templates/
│   ├── postgresql.conf.erb
│   └── pg_hba.conf.erb
└── libraries/
    └── helpers.rb
```

### Recipe установки

```ruby
# recipes/server.rb

# Установка PostgreSQL
apt_repository 'postgresql' do
  uri 'http://apt.postgresql.org/pub/repos/apt'
  distribution "#{node['lsb']['codename']}-pgdg"
  components ['main']
  key 'https://www.postgresql.org/media/keys/ACCC4CF8.asc'
end

package "postgresql-#{node['postgresql']['version']}" do
  action :install
end

# Конфигурация
template "#{node['postgresql']['config_dir']}/postgresql.conf" do
  source 'postgresql.conf.erb'
  owner 'postgres'
  group 'postgres'
  mode '0644'
  variables(
    listen_addresses: node['postgresql']['listen_addresses'],
    port: node['postgresql']['port'],
    max_connections: node['postgresql']['max_connections'],
    shared_buffers: node['postgresql']['shared_buffers'],
    effective_cache_size: node['postgresql']['effective_cache_size']
  )
  notifies :reload, 'service[postgresql]'
end

template "#{node['postgresql']['config_dir']}/pg_hba.conf" do
  source 'pg_hba.conf.erb'
  owner 'postgres'
  group 'postgres'
  mode '0640'
  variables(
    hba_entries: node['postgresql']['hba_entries']
  )
  notifies :reload, 'service[postgresql]'
end

service 'postgresql' do
  action [:enable, :start]
end

# Создание баз данных
node['postgresql']['databases'].each do |db_name, db_config|
  postgresql_database db_name do
    connection(
      host: '127.0.0.1',
      port: node['postgresql']['port'],
      username: 'postgres'
    )
    encoding db_config['encoding'] || 'UTF8'
    action :create
  end
end

# Создание пользователей
node['postgresql']['users'].each do |user_name, user_config|
  postgresql_user user_name do
    connection(
      host: '127.0.0.1',
      port: node['postgresql']['port'],
      username: 'postgres'
    )
    password user_config['password']
    superuser user_config['superuser'] || false
    createdb user_config['createdb'] || false
    action :create
  end
end
```

### Attributes

```ruby
# attributes/default.rb

default['postgresql']['version'] = '15'
default['postgresql']['config_dir'] = '/etc/postgresql/15/main'
default['postgresql']['data_dir'] = '/var/lib/postgresql/15/main'

default['postgresql']['listen_addresses'] = 'localhost'
default['postgresql']['port'] = 5432
default['postgresql']['max_connections'] = 100

default['postgresql']['shared_buffers'] = '128MB'
default['postgresql']['effective_cache_size'] = '4GB'
default['postgresql']['work_mem'] = '4MB'
default['postgresql']['maintenance_work_mem'] = '64MB'

default['postgresql']['databases'] = {}
default['postgresql']['users'] = {}

default['postgresql']['hba_entries'] = [
  { type: 'local', database: 'all', user: 'postgres', method: 'peer' },
  { type: 'local', database: 'all', user: 'all', method: 'peer' },
  { type: 'host', database: 'all', user: 'all', address: '127.0.0.1/32', method: 'scram-sha-256' }
]
```

## Terraform для PostgreSQL

### Provider и ресурсы

```hcl
# main.tf

terraform {
  required_providers {
    postgresql = {
      source  = "cyrilgdn/postgresql"
      version = "~> 1.21"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# AWS RDS PostgreSQL
resource "aws_db_instance" "postgresql" {
  identifier           = "production-postgres"
  engine               = "postgres"
  engine_version       = "15.4"
  instance_class       = "db.t3.medium"
  allocated_storage    = 100
  storage_type         = "gp3"

  db_name  = "production_db"
  username = var.db_username
  password = var.db_password

  vpc_security_group_ids = [aws_security_group.postgresql.id]
  db_subnet_group_name   = aws_db_subnet_group.postgresql.name

  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "Mon:04:00-Mon:05:00"

  multi_az               = true
  storage_encrypted      = true

  parameter_group_name = aws_db_parameter_group.postgresql.name

  tags = {
    Environment = "production"
  }
}

resource "aws_db_parameter_group" "postgresql" {
  family = "postgres15"
  name   = "production-postgres-params"

  parameter {
    name  = "shared_buffers"
    value = "{DBInstanceClassMemory/4}"
  }

  parameter {
    name  = "max_connections"
    value = "200"
  }

  parameter {
    name  = "log_statement"
    value = "ddl"
  }
}

# PostgreSQL Provider для управления объектами БД
provider "postgresql" {
  host     = aws_db_instance.postgresql.address
  port     = aws_db_instance.postgresql.port
  database = "postgres"
  username = var.db_username
  password = var.db_password
  sslmode  = "require"
}

resource "postgresql_database" "app_db" {
  name              = "application_db"
  owner             = postgresql_role.app_user.name
  encoding          = "UTF8"
  lc_collate        = "en_US.UTF-8"
  lc_ctype          = "en_US.UTF-8"
  connection_limit  = -1
  allow_connections = true
}

resource "postgresql_role" "app_user" {
  name     = "app_user"
  login    = true
  password = var.app_user_password
}

resource "postgresql_role" "readonly_user" {
  name     = "readonly_user"
  login    = true
  password = var.readonly_password
}

resource "postgresql_grant" "app_user_all" {
  database    = postgresql_database.app_db.name
  role        = postgresql_role.app_user.name
  schema      = "public"
  object_type = "table"
  privileges  = ["SELECT", "INSERT", "UPDATE", "DELETE"]
}

resource "postgresql_grant" "readonly_select" {
  database    = postgresql_database.app_db.name
  role        = postgresql_role.readonly_user.name
  schema      = "public"
  object_type = "table"
  privileges  = ["SELECT"]
}
```

## Best Practices

### 1. Идемпотентность

```yaml
# Ansible - всегда идемпотентные операции
- name: Create user (idempotent)
  postgresql_user:
    name: app_user
    password: "{{ password }}"
    state: present  # Не создаст если уже существует
```

### 2. Секреты и пароли

```yaml
# Используйте Ansible Vault
ansible-vault create group_vars/vault.yml

# В vault.yml
vault_db_password: "super_secret_password"

# В playbook
password: "{{ vault_db_password }}"
```

### 3. Тестирование конфигурации

```yaml
# Проверка синтаксиса
ansible-playbook --syntax-check playbook.yml

# Dry run
ansible-playbook --check --diff playbook.yml

# Тестирование с Molecule
molecule test
```

### 4. Версионирование

```yaml
# Используйте Git для хранения конфигураций
# .gitignore
*.retry
.vault_pass
group_vars/vault.yml
```

### 5. Документация

```yaml
# Документируйте переменные
# defaults/main.yml

# PostgreSQL version to install
# Supported: 13, 14, 15, 16
postgresql_version: "15"

# Listen addresses
# Use '*' for all interfaces, 'localhost' for local only
postgresql_listen_addresses: "localhost"
```

## Мониторинг конфигурации

### Ansible для мониторинга

```yaml
# playbooks/check_config.yml
---
- name: Check PostgreSQL configuration
  hosts: postgresql
  become: true

  tasks:
    - name: Get current configuration
      postgresql_query:
        db: postgres
        query: "SELECT name, setting FROM pg_settings WHERE name IN ('shared_buffers', 'work_mem', 'max_connections')"
      register: current_settings
      become_user: postgres

    - name: Display current settings
      debug:
        var: current_settings.query_result

    - name: Check if configuration matches expected
      assert:
        that:
          - item.setting == expected_settings[item.name]
        fail_msg: "Setting {{ item.name }} is {{ item.setting }}, expected {{ expected_settings[item.name] }}"
      loop: "{{ current_settings.query_result }}"
      vars:
        expected_settings:
          shared_buffers: "{{ postgresql_shared_buffers }}"
          work_mem: "{{ postgresql_work_mem }}"
          max_connections: "{{ postgresql_max_connections }}"
```

## Заключение

Configuration Management для PostgreSQL позволяет:

1. **Воспроизводимость** - одинаковая конфигурация на всех серверах
2. **Версионирование** - отслеживание изменений в Git
3. **Автоматизация** - быстрое развертывание новых серверов
4. **Compliance** - соответствие стандартам безопасности
5. **Документация** - код как документация инфраструктуры

Рекомендуемый выбор инструмента:
- **Ansible** - для небольших и средних инфраструктур, простота использования
- **Puppet/Chef** - для крупных enterprise инфраструктур
- **Terraform** - для облачной инфраструктуры (RDS, Cloud SQL)

Следующий шаг - изучение миграций схемы базы данных для управления изменениями структуры БД.
