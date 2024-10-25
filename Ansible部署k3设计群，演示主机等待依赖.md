你好！

使用 Ansible 自动化部署 k3s 集群是一个高效且可重复的解决方案。考虑到 k3s 的主节点（Master）和子节点（Worker）之间存在先后关系，确保主节点在子节点之前成功部署是至关重要的。下面，我将详细介绍如何使用 Ansible 部署一个包含 1 个主节点和 2 个子节点的 k3s 集群，并处理节点部署的先后顺序。

## 一、部署概述

1. **主节点（Master Node）**：
   - 负责集群的控制平面，包括 API 服务器、调度器等。
   - 子节点需要通过主节点的 Token 或地址加入集群。

2. **子节点（Worker Nodes）**：
   - 负责运行实际的工作负载，如 Pod。
   - 需要在主节点部署完成后，才能正确加入集群。

## 二、项目结构

建议使用以下目录结构，以便于管理和复用角色：

```
k3s_ansible/
├── inventory/
│   └── hosts.ini
├── playbooks/
│   └── deploy_k3s.yml
├── roles/
│   ├── master/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── vars/
│   │   │   └── main.yml
│   │   └── templates/
│   ├── worker/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── vars/
│   │   │   └── main.yml
│   │   └── templates/
│   └── common/
│       ├── tasks/
│       │   └── main.yml
│       ├── vars/
│       │   └── main.yml
│       └── templates/
├── group_vars/
│   └── all.yml
├── host_vars/
│   └── [hostname].yml
└── ansible.cfg
```

## 三、详细步骤

### 1. 准备工作

#### 1.1. 安装 Ansible

确保你的控制节点（运行 Ansible 的机器）已经安装了 Ansible。如果尚未安装，可以使用以下命令安装：

```bash
sudo apt update
sudo apt install ansible -y
```

#### 1.2. 配置 SSH 访问

确保 Ansible 控制节点可以通过 SSH 无密码访问目标主机。可以通过生成 SSH 密钥并将公钥复制到目标主机来实现：

```bash
# 在控制节点上生成 SSH 密钥（如果尚未生成）
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# 将公钥复制到主节点和子节点
ssh-copy-id your_username@master_ip
ssh-copy-id your_username@worker1_ip
ssh-copy-id your_username@worker2_ip
```

### 2. 配置 Ansible Inventory

在 `inventory/hosts.ini` 文件中定义主节点和子节点：

```ini
[masters]
master ansible_host=192.168.1.10 ansible_user=your_username

[workers]
worker1 ansible_host=192.168.1.11 ansible_user=your_username
worker2 ansible_host=192.168.1.12 ansible_user=your_username

[k3s_cluster:children]
masters
workers
```

### 3. 定义变量

在 `group_vars/all.yml` 中定义全局变量，例如：

```yaml
---
# group_vars/all.yml

k3s_version: v1.26.3+k3s1
k3s_install_options: "--disable traefik --disable servicelb --disable local-storage"
```

### 4. 创建角色

#### 4.1. common 角色

用于安装依赖和基础配置。

**roles/common/tasks/main.yml**

```yaml
---
- name: 更新 APT 包缓存
  apt:
    update_cache: yes
    cache_valid_time: 600

- name: 安装必要的软件包
  apt:
    name:
      - curl
      - apt-transport-https
      - ca-certificates
      - software-properties-common
    state: present
```

**roles/common/vars/main.yml**

```yaml
---
# roles/common/vars/main.yml

common_packages:
  - curl
  - apt-transport-https
  - ca-certificates
  - software-properties-common
```

#### 4.2. master 角色

负责在主节点上安装 k3s 并获取 Token。

**roles/master/tasks/main.yml**

```yaml
---
- name: 安装 k3s 主节点
  shell: |
    curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION={{ k3s_version }} INSTALL_K3S_EXEC="{{ k3s_install_options }}" sh -
  args:
    creates: /etc/rancher/k3s/k3s.yaml

- name: 等待 k3s 服务启动
  wait_for:
    port: 6443
    host: "{{ ansible_host }}"
    timeout: 60

- name: 获取 k3s Token
  command: cat /var/lib/rancher/k3s/server/node-token
  register: k3s_token

- name: 设置事实 - k3s_token
  set_fact:
    k3s_token: "{{ k3s_token.stdout }}"
```

**roles/master/vars/main.yml**

```yaml
---
# roles/master/vars/main.yml

k3s_install_options: "--disable traefik --disable servicelb --disable local-storage"
```

#### 4.3. worker 角色

负责在子节点上安装 k3s 并加入集群。

**roles/worker/tasks/main.yml**

```yaml
---
- name: 安装 k3s 子节点
  shell: |
    curl -sfL https://get.k3s.io | K3S_URL=https://{{ hostvars['master']['ansible_host'] }}:6443 K3S_TOKEN={{ hostvars['master']['k3s_token'] }} INSTALL_K3S_VERSION={{ k3s_version }} sh -
  args:
    creates: /etc/rancher/k3s/k3s.yaml

- name: 等待 k3s 服务启动
  wait_for:
    port: 6443
    host: "{{ hostvars['master']['ansible_host'] }}"
    timeout: 60
```

**roles/worker/vars/main.yml**

```yaml
---
# roles/worker/vars/main.yml

k3s_install_options: ""
```

### 5. 创建 Playbook

在 `playbooks/deploy_k3s.yml` 中定义 Playbook，分阶段部署主节点和子节点。

**playbooks/deploy_k3s.yml**

```yaml
---
- name: 部署 k3s 主节点
  hosts: masters
  become: yes
  roles:
    - common
    - master

- name: 部署 k3s 子节点
  hosts: workers
  become: yes
  roles:
    - common
    - worker
  when: hostvars['master']['k3s_token'] is defined
```

### 6. 执行 Playbook

确保主节点和子节点都在 Inventory 中正确配置后，可以执行 Playbook：

```bash
ansible-playbook -i inventory/hosts.ini playbooks/deploy_k3s.yml
```

### 7. 验证集群

部署完成后，可以通过主节点登录并验证集群状态：

```bash
ssh your_username@master_ip

# 查看节点状态
sudo k3s kubectl get nodes

# 示例输出：
# NAME        STATUS   ROLES    AGE   VERSION
# master      Ready    master   10m   v1.26.3+k3s1
# worker1     Ready    <none>   8m    v1.26.3+k3s1
# worker2     Ready    <none>   8m    v1.26.3+k3s1
```

## 四、处理节点部署顺序

在上述 Playbook 中，主节点和子节点的部署被分为两个独立的 Play。这确保了主节点首先被部署和配置，然后子节点在主节点部署完成后加入集群。

### 4.1. Playbook 执行顺序

1. **第一部分**：部署主节点（masters 组）
   - 执行 `common` 和 `master` 角色。
   - 安装 k3s 主节点并获取 Token。

2. **第二部分**：部署子节点（workers 组）
   - 执行 `common` 和 `worker` 角色。
   - 使用主节点的 Token 加入集群。

### 4.2. 使用条件语句确保主节点部署完成

在子节点部署的 Play 中，添加 `when` 条件，确保只有在主节点的 Token 被成功获取后，才开始部署子节点：

```yaml
when: hostvars['master']['k3s_token'] is defined
```

这避免了子节点在主节点尚未准备好时尝试加入集群，从而减少失败的风险。

## 五、进一步优化与最佳实践

### 1. 使用 Ansible Roles

将主节点和子节点的部署逻辑封装到不同的角色中，提高 Playbook 的模块化和可复用性。上述示例已体现这一点。

### 2. 变量管理

- **使用 `group_vars` 和 `host_vars`**：
  - 将不同主机组的变量定义在相应的 `group_vars` 文件中，如 `group_vars/masters.yml` 和 `group_vars/workers.yml`。
  - 将特定主机的变量定义在 `host_vars` 目录下。

### 3. 错误处理与重试机制

在 Playbook 中添加错误处理和重试机制，提高部署的可靠性。例如，使用 `retries` 和 `delay` 选项：

```yaml
- name: 安装 k3s 主节点
  shell: |
    curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION={{ k3s_version }} INSTALL_K3S_EXEC="{{ k3s_install_options }}" sh -
  args:
    creates: /etc/rancher/k3s/k3s.yaml
  register: install_k3s
  retries: 3
  delay: 10
  until: install_k3s is success
```

### 4. 使用 Ansible Vault 管理敏感信息

对于需要传递的敏感信息（如 Token），使用 Ansible Vault 进行加密，确保安全性。

```bash
# 创建加密文件
ansible-vault create group_vars/all/vault.yml

# 在 Playbook 中引用
vars_files:
  - group_vars/all/vault.yml
```

### 5. 并行部署与控制

默认情况下，Ansible 会并行处理不同主机的任务。如果需要控制并发数，可以在 `ansible.cfg` 中配置 `forks`：

```ini
[defaults]
forks = 10
```

### 6. 使用 Tags 进行部分执行

在 Playbook 中使用 Tags，允许你有选择地执行 Playbook 的某些部分。例如，只部署主节点或子节点：

```yaml
- name: 部署 k3s 主节点
  hosts: masters
  become: yes
  roles:
    - common
    - master
  tags: master

- name: 部署 k3s 子节点
  hosts: workers
  become: yes
  roles:
    - common
    - worker
  when: hostvars['master']['k3s_token'] is defined
  tags: workers
```

执行时指定 Tag：

```bash
# 仅部署主节点
ansible-playbook -i inventory/hosts.ini playbooks/deploy_k3s.yml --tags master

# 仅部署子节点
ansible-playbook -i inventory/hosts.ini playbooks/deploy_k3s.yml --tags workers
```

### 7. 集群监控与管理

部署完成后，考虑使用工具如 `kubectl`、`helm` 或其他 Kubernetes 管理工具，进行集群的监控和管理。可以在 Ansible Playbook 中添加任务，自动配置这些工具。

## 六、总结

通过使用 Ansible 部署 k3s 集群，你可以实现高效、可重复和可维护的自动化部署流程。关键点包括：

1. **角色化管理**：将主节点和子节点的部署逻辑分别封装到不同的角色中。
2. **顺序控制**：通过 Playbook 的分阶段部署，确保主节点在子节点之前成功部署。
3. **变量管理**：合理使用 `group_vars` 和 `host_vars`，并结合 Ansible Vault 管理敏感信息。
4. **错误处理与重试**：提高部署过程的鲁棒性，减少失败的风险。
5. **最佳实践**：遵循 Ansible 的最佳实践，如模块化、使用 Tags、控制并发等，提升项目的可维护性和扩展性。

希望上述指南能帮助你顺利使用 Ansible 部署 k3s 集群。如果在实际操作中遇到任何问题或有进一步的疑问，欢迎随时提问！

你好！

在使用 Ansible 进行自动化配置和管理时，了解并遵循其保留关键字、标准文件布局以及约定的规则至关重要。这不仅有助于避免潜在的冲突和错误，还能提升项目的可维护性和可扩展性。以下是对 Ansible 中保留关键字、标准文件布局和约定规则的详细介绍。

## 一、Ansible 的保留关键字

### 1. Playbook 关键字

在 Ansible Playbook 中，有一系列保留的关键字，这些关键字用于定义 Playbook 的结构和行为。以下是常见的保留关键字及其作用：

- **`hosts`**：指定 Play 要作用的主机组或单个主机。
  
  ```yaml
  - name: 示例 Play
    hosts: webservers
    tasks:
      - name: 安装 Nginx
        apt:
          name: nginx
          state: present
  ```

- **`tasks`**：定义一系列任务（Tasks），这些任务将在指定的主机上依次执行。

- **`vars`**：在 Play 级别定义变量，这些变量在整个 Play 中可用。

- **`vars_files`**：引用外部变量文件。

- **`roles`**：指定要应用的角色（Roles）。

- **`become`**：提升权限，如使用 sudo 执行任务。

- **`gather_facts`**：是否收集主机的事实信息（默认为 `true`）。

- **`handlers`**：定义处理程序（Handlers），通常用于在任务改变状态时执行特定操作，如重启服务。

### 2. 角色（Roles）关键字

在 Ansible 角色中，也有一系列保留的关键字，用于定义角色的结构和功能：

- **`tasks`**：角色的主要任务列表，定义角色需要执行的任务。

- **`handlers`**：角色中的处理程序，响应任务中的变化。

- **`vars`**：角色级别的变量，优先级高于 `defaults`。

- **`defaults`**：角色的默认变量，优先级最低，易于被覆盖。

- **`files`**：存放角色需要复制到目标主机的静态文件。

- **`templates`**：存放 Jinja2 模板文件，动态生成配置文件等。

- **`meta`**：定义角色的元数据，如依赖关系。

- **`library`**：存放自定义模块。

- **`lookup_plugins`**：存放自定义查找插件。

- **`vars_plugins`**：存放自定义变量插件。

### 3. 其他保留关键字

除了 Playbook 和角色中的关键字外，还有一些其他关键字在 Ansible 中有特定用途：

- **`include`** 和 **`import`**：用于包含其他 Playbook、任务或角色。

- **`block`**：用于将一组任务组织在一起，并应用共同的属性或处理错误。

- **`when`**：条件语句，用于控制任务的执行条件。

- **`register`**：用于注册任务的输出结果到变量中。

- **`notify`** 和 **`listen`**：用于触发处理程序。

- **`loop`** 和 **`with_items`**：用于循环执行任务。

### 4. 注意事项

- **避免使用保留关键字作为变量名**：为了防止变量覆盖或解析错误，避免在 Playbook、角色和变量文件中使用上述保留关键字作为自定义变量名。

- **遵循命名规范**：使用有意义且唯一的变量名，遵循 Ansible 的命名约定，如使用小写字母和下划线分隔（snake_case）。

## 二、Ansible 的标准文件布局

遵循 Ansible 推荐的目录结构有助于提高项目的可维护性和可复用性。以下是 Ansible 项目和角色的标准文件布局及其作用。

### 1. Ansible 项目的标准目录结构

一个组织良好的 Ansible 项目通常具有以下目录结构：

```
ansible_project/
├── inventory/
│   ├── hosts.ini
│   ├── group_vars/
│   │   ├── all.yml
│   │   └── webservers.yml
│   └── host_vars/
│       ├── host1.yml
│       └── host2.yml
├── playbooks/
│   ├── site.yml
│   ├── webservers.yml
│   └── dbservers.yml
├── roles/
│   ├── common/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── handlers/
│   │   │   └── main.yml
│   │   ├── templates/
│   │   │   └── some_template.j2
│   │   ├── files/
│   │   │   └── some_file.conf
│   │   ├── vars/
│   │   │   └── main.yml
│   │   ├── defaults/
│   │   │   └── main.yml
│   │   ├── meta/
│   │   │   └── main.yml
│   ├── webserver/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── ... (其他子目录同上)
│   └── dbserver/
│       ├── tasks/
│       │   └── main.yml
│       └── ... (其他子目录同上)
├── group_vars/
│   ├── all.yml
│   ├── webservers.yml
│   └── dbservers.yml
├── host_vars/
│   ├── host1.yml
│   └── host2.yml
├── library/
│   └── custom_module.py
├── filter_plugins/
│   └── custom_filters.py
├── files/
│   └── some_global_file.conf
├── templates/
│   └── some_global_template.j2
├── vars/
│   └── main.yml
├── defaults/
│   └── main.yml
├── meta/
│   └── main.yml
├── ansible.cfg
└── requirements.yml
```

### 2. 各目录和文件的作用

#### 2.1. `inventory/`

- **作用**：定义和组织受管理的主机及其组。
- **内容**：
  - **`hosts.ini`**：库存文件，列出所有管理的主机及其分组。
  - **`group_vars/`**：为特定主机组定义变量。
    - **`all.yml`**：为所有主机定义的变量。
    - **`webservers.yml`**：为 `webservers` 组定义的变量。
  - **`host_vars/`**：为特定主机定义变量。
    - **`host1.yml`**：仅适用于 `host1` 的变量。
    - **`host2.yml`**：仅适用于 `host2` 的变量。

#### 2.2. `playbooks/`

- **作用**：存放 Ansible Playbook 文件，定义自动化任务和流程。
- **内容**：
  - **`site.yml`**：主 Playbook，通常包含所有角色和任务。
  - **`webservers.yml`**、**`dbservers.yml`**：针对特定任务或角色的 Playbook。

#### 2.3. `roles/`

- **作用**：存放角色（Roles），每个角色包含一套标准子目录和文件，用于组织相关任务、变量、模板等。
- **内容**：
  - **`common/`**、**`webserver/`**、**`dbserver/`**：不同的角色，每个角色包含 `tasks/`、`handlers/`、`templates/` 等子目录。

#### 2.4. `group_vars/` 和 `host_vars/`

- **作用**：集中管理主机组和特定主机的变量。
- **区别**：
  - **`group_vars/`**：针对主机组定义变量，适用于组内所有主机。
  - **`host_vars/`**：针对特定主机定义变量，仅适用于该主机。

#### 2.5. `library/`

- **作用**：存放自定义 Ansible 模块（Custom Modules）。
- **内容**：自定义的 Python 模块文件，如 `custom_module.py`。

#### 2.6. `filter_plugins/`

- **作用**：存放自定义过滤器插件（Custom Filters）。
- **内容**：自定义的 Jinja2 过滤器文件，如 `custom_filters.py`。

#### 2.7. `files/`

- **作用**：存放需要直接复制到目标主机的静态文件。
- **内容**：如配置文件、脚本等静态文件，如 `some_global_file.conf`。

#### 2.8. `templates/`

- **作用**：存放需要模板化的文件，使用 Jinja2 模板语法。
- **内容**：如动态生成的配置文件模板，如 `some_global_template.j2`。

#### 2.9. `vars/` 和 `defaults/`

- **作用**：
  - **`vars/`**：定义全局变量，优先级高于 `defaults`。
  - **`defaults/`**：定义全局默认变量，优先级最低，易于被覆盖。
- **内容**：
  - **`vars/main.yml`**：全局变量定义。
  - **`defaults/main.yml`**：全局默认变量定义。

#### 2.10. `meta/`

- **作用**：定义角色的元数据，如依赖关系。
- **内容**：
  - **`meta/main.yml`**：角色的元数据文件，定义角色依赖关系等信息。

#### 2.11. `ansible.cfg`

- **作用**：Ansible 的配置文件，定义全局配置选项，如库存文件路径、角色路径、并发数等。

#### 2.12. `requirements.yml`

- **作用**：定义角色和集合的依赖关系，方便使用 `ansible-galaxy` 安装。
- **内容**：列出需要安装的角色和集合。

### 3. 角色（Roles）的标准目录结构

每个角色（如 `common`、`webserver`）应遵循以下标准目录结构：

```
roles/
└── role_name/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── templates/
    │   └── template_file.j2
    ├── files/
    │   └── file.conf
    ├── vars/
    │   └── main.yml
    ├── defaults/
    │   └── main.yml
    ├── meta/
    │   └── main.yml
    ├── library/
    │   └── custom_module.py
    ├── filter_plugins/
    │   └── custom_filters.py
    └── README.md
```

#### 各子目录和文件的作用：

- **`tasks/main.yml`**：定义该角色的主要任务列表。Ansible 会自动加载 `tasks/main.yml` 作为角色的入口任务。

- **`handlers/main.yml`**：定义需要由该角色触发的处理程序（Handlers），如服务的重启等。

- **`templates/`**：存放 Jinja2 模板文件，用于生成配置文件等需要动态内容的文件。

- **`files/`**：存放静态文件，这些文件会被直接复制到目标主机。

- **`vars/main.yml`**：定义该角色的变量，优先级高于 `defaults`。适用于需要固定的、不可覆盖的变量。

- **`defaults/main.yml`**：定义该角色的默认变量，优先级最低，用户可以在调用角色时轻松覆盖。

- **`meta/main.yml`**：定义角色的元数据，如依赖关系（Dependencies）。确保角色的依赖关系在 Playbook 执行前被正确加载。

- **`library/`**：存放自定义模块，供该角色使用。

- **`filter_plugins/`**：存放自定义过滤器插件，供该角色使用。

- **`README.md`**：角色的说明文件，描述角色的用途、变量、依赖等信息，方便团队协作和角色复用。

### 4. 其他约定和规则

#### 4.1. 文件命名和扩展名

- **YAML 文件**：通常使用 `.yml` 或 `.yaml` 扩展名。保持一致性，建议统一使用 `.yml`。
  
  ```yaml
  playbook.yml
  main.yml
  ```

- **Jinja2 模板文件**：使用 `.j2` 扩展名。
  
  ```jinja
  nginx.conf.j2
  ```

#### 4.2. 缩进

- **空格代替制表符（Tabs）**：YAML 对缩进非常敏感，建议始终使用空格（推荐 2 个空格）进行缩进，避免使用制表符。
  
  ```yaml
  tasks:
    - name: 安装 Nginx
      apt:
        name: nginx
        state: present
  ```

#### 4.3. 变量命名规范

- **使用小写字母和下划线分隔（snake_case）**：确保变量名具有可读性且不与保留关键字冲突。
  
  ```yaml
  app_name: "MyApp"
  database_host: "db.example.com"
  ```

- **避免使用特殊字符和空格**：变量名应简洁明了，避免使用特殊字符和空格。

#### 4.4. 角色复用和模块化

- **拆分职责**：将不同功能或职责拆分到不同的角色中，提升模块化和可复用性。
  
  ```yaml
  roles/
  ├── common/
  ├── webserver/
  ├── database/
  └── monitoring/
  ```

- **利用角色依赖**：在 `meta/main.yml` 中定义角色之间的依赖关系，确保角色按正确顺序加载。
  
  ```yaml
  # roles/webserver/meta/main.yml
  ---
  dependencies:
    - { role: common }
    - { role: database }
  ```

#### 4.5. 使用变量文件和模板

- **分离配置和代码**：将配置参数定义在变量文件中，使用模板生成动态配置文件，提升灵活性和可维护性。
  
  ```yaml
  # vars/main.yml
  ---
  nginx_port: 80
  ```

  ```jinja
  # templates/nginx.conf.j2
  server {
      listen {{ nginx_port }};
      server_name localhost;
      ...
  }
  ```

#### 4.6. Ansible 配置文件 (`ansible.cfg`)

- **位置**：项目根目录下，或者用户主目录下的 `.ansible.cfg`，或者系统级别的 `/etc/ansible/ansible.cfg`。
  
- **配置选项**：定义库存文件路径、角色路径、并发数等全局配置。
  
  ```ini
  [defaults]
  inventory = inventory/hosts.ini
  roles_path = roles
  forks = 10
  ```

## 三、Ansible 的约定规则

### 1. 使用角色（Roles）组织任务

- **角色化管理**：将相关的任务、变量、模板等组织到独立的角色中，提升 Playbook 的可复用性和可维护性。
  
  ```yaml
  - name: 部署 Web 服务器
    hosts: webservers
    roles:
      - common
      - webserver
  ```

### 2. 分层管理变量

- **层级覆盖**：利用 Ansible 的变量优先级机制，将变量按层级管理，确保变量值按需覆盖。
  
  ```yaml
  # group_vars/all.yml
  timezone: "UTC"
  
  # group_vars/webservers.yml
  timezone: "Asia/Shanghai"
  
  # host_vars/webserver1.yml
  timezone: "Europe/London"
  ```

  **最终结果**：`webserver1` 的 `timezone` 为 `"Europe/London"`，`webservers` 组的其他主机为 `"Asia/Shanghai"`，所有主机默认为 `"UTC"`。

### 3. 避免硬编码

- **参数化配置**：尽量使用变量和模板，避免在 Playbook 和角色中硬编码值，提升灵活性和可维护性。
  
  ```yaml
  # vars/main.yml
  app_port: 8080
  ```

  ```yaml
  # tasks/main.yml
  - name: 启动应用服务
    systemd:
      name: myapp
      state: started
      enabled: yes
      port: "{{ app_port }}"
  ```

### 4. 使用模板和 Jinja2

- **动态生成配置文件**：使用 Jinja2 模板，根据变量动态生成配置文件，适应不同环境和需求。
  
  ```jinja
  # templates/app.conf.j2
  [server]
  port = {{ app_port }}
  env = {{ environment }}
  ```

### 5. 文档化

- **详细说明**：为 Playbook 和角色编写详细的文档，说明其用途、变量、依赖等信息，提升团队协作和代码可维护性。
  
  ```markdown
  # roles/webserver/README.md
  
  ## 描述
  
  该角色用于安装和配置 Nginx 作为 Web 服务器。
  
  ## 变量
  
  ### 默认变量 (`defaults/main.yml`)
  
  - `nginx_port`：Nginx 监听的端口，默认值为 `80`。
  
  ### 角色变量 (`vars/main.yml`)
  
  - `nginx_user`：Nginx 运行的用户，默认值为 `www-data`。
  
  ## 依赖
  
  - `common` 角色
  
  ## 使用示例
  
  ​```yaml
  - hosts: webservers
    roles:
      - role: webserver
        vars:
          nginx_port: 8080
  ```
  ```
  
  ```

### 6. 使用 Ansible Galaxy

- **分享和复用**：将自定义的角色上传到 Ansible Galaxy，方便在不同项目和团队之间共享和复用。
  
  ```bash
  ansible-galaxy role create my_custom_role
  ```

### 7. 安全管理

- **敏感信息**：使用 Ansible Vault 加密敏感信息，如密码、密钥等，确保配置的安全性。
  
  ```bash
  # 创建加密文件
  ansible-vault create vars/secret.yml
  
  # 在 Playbook 中引用
  vars_files:
    - vars/secret.yml
  ```

### 8. 测试与验证

- **语法检查**：使用 Ansible 的 `--syntax-check` 选项检查 Playbook 语法。
  
  ```bash
  ansible-playbook playbook.yml --syntax-check
  ```

- **干运行（Dry Run）**：使用 `--check` 选项进行干运行，预测 Playbook 的执行结果。
  
  ```bash
  ansible-playbook playbook.yml --check
  ```

- **格式化工具**：使用 `yamllint` 等工具检查 YAML 文件的语法和格式。
  
  ```bash
  yamllint playbook.yml
  ```

### 9. 控制并发与性能

- **配置并发数**：通过 `ansible.cfg` 文件中的 `forks` 参数控制并发执行的任务数量，优化 Playbook 的执行速度。
  
  ```ini
  [defaults]
  forks = 20
  ```

- **分阶段执行**：在大型基础设施中，分阶段执行 Playbook，逐步部署和验证，避免一次性操作导致的资源压力或错误扩散。

### 10. 使用 Tags

- **部分执行**：通过在 Playbook 中使用 Tags，有选择地执行特定的任务或角色，提升 Playbook 的灵活性和执行效率。
  
  ```yaml
  - name: 安装 Web 服务器
    hosts: webservers
    roles:
      - role: webserver
    tags:
      - web
  ```

  **执行带 Tag 的任务**：
  
  ```bash
  ansible-playbook playbook.yml --tags web
  ```

## 四、总结

### 1. 保留关键字的重要性

了解并避免使用 Ansible 的保留关键字作为自定义变量名、任务名等，能够避免变量覆盖和解析错误，确保 Playbook 和角色的正确执行。

### 2. 遵循标准文件布局

遵循 Ansible 推荐的目录结构和文件命名规范，有助于提高项目的可维护性、可复用性和团队协作效率。标准布局使得 Ansible 能够自动加载和解析相关文件，减少手动配置的复杂性。

### 3. 采用约定规则和最佳实践

- **变量管理**：合理分层管理变量，利用 Ansible 的变量优先级机制，确保变量按需覆盖和应用。
- **模块化和角色复用**：通过角色化管理，提升 Playbook 的模块化和可复用性，减少重复代码。
- **文档化**：详细的文档说明提升团队协作效率，帮助新成员快速上手和理解项目结构。
- **安全管理**：使用 Ansible Vault 管理敏感信息，确保配置的安全性。
- **测试与验证**：通过语法检查、干运行和格式化工具，确保 Playbook 的正确性和可维护性。

通过深入理解 Ansible 的保留关键字、标准文件布局和约定规则，并严格遵循最佳实践，你可以构建高效、可维护且安全的自动化配置项目。如果你有更多具体问题或需要进一步的帮助，欢迎随时提问！

当然，以下是基于专业最佳实践重构的 Ansible Playbook，用于部署一个包含 1 个主节点（Master）和 2 个子节点（Worker）的 k3s 集群。重构后的代码结构更模块化、可维护性更强，并且遵循了 Ansible 的标准文件布局和变量管理原则。

## 一、项目结构

一个良好的 Ansible 项目结构有助于提高可维护性、可复用性和可读性。以下是推荐的目录结构：

```
k3s_ansible/
├── ansible.cfg
├── inventory/
│   └── hosts.ini
├── playbooks/
│   └── deploy_k3s.yml
├── roles/
│   ├── common/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── handlers/
│   │   │   └── main.yml
│   │   ├── templates/
│   │   ├── files/
│   │   ├── vars/
│   │   │   └── main.yml
│   │   ├── defaults/
│   │   │   └── main.yml
│   │   ├── meta/
│   │   │   └── main.yml
│   │   └── README.md
│   ├── master/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── handlers/
│   │   │   └── main.yml
│   │   ├── templates/
│   │   ├── files/
│   │   ├── vars/
│   │   │   └── main.yml
│   │   ├── defaults/
│   │   │   └── main.yml
│   │   ├── meta/
│   │   │   └── main.yml
│   │   └── README.md
│   └── worker/
│       ├── tasks/
│       │   └── main.yml
│       ├── handlers/
│       │   └── main.yml
│       ├── templates/
│       ├── files/
│       ├── vars/
│       │   └── main.yml
│       ├── defaults/
│       │   └── main.yml
│       ├── meta/
│       │   └── main.yml
│       └── README.md
├── group_vars/
│   ├── all.yml
│   ├── masters.yml
│   └── workers.yml
├── host_vars/
│   ├── master.yml
│   ├── worker1.yml
│   └── worker2.yml
├── files/
│   └── ... (全局静态文件)
├── templates/
│   └── ... (全局模板文件)
└── vault.yml (使用 Ansible Vault 加密的敏感变量)
```

## 二、配置文件与目录说明

### 1. `ansible.cfg`

配置 Ansible 的行为，例如库存文件路径、角色路径等。放在项目根目录下，确保项目内所有操作都使用该配置。

```ini
[defaults]
inventory = inventory/hosts.ini
roles_path = roles
forks = 10
host_key_checking = False
retry_files_enabled = False
```

### 2. `inventory/hosts.ini`

定义管理的主机及其分组。

```ini
[masters]
master ansible_host=192.168.1.10 ansible_user=your_username

[workers]
worker1 ansible_host=192.168.1.11 ansible_user=your_username
worker2 ansible_host=192.168.1.12 ansible_user=your_username

[k3s_cluster:children]
masters
workers
```

### 3. `group_vars/`

集中管理不同组的变量。

- **`group_vars/all.yml`**：定义所有主机通用的变量。
  
  ```yaml
  ---
  k3s_version: v1.26.3+k3s1
  k3s_install_options: "--disable traefik --disable servicelb --disable local-storage"
  timezone: "UTC"
  ntp_server: "pool.ntp.org"
  ```

- **`group_vars/masters.yml`**：定义主节点特有的变量。
  
  ```yaml
  ---
  # 主节点特有变量
  k3s_role: master
  ```

- **`group_vars/workers.yml`**：定义子节点特有的变量。
  
  ```yaml
  ---
  # 子节点特有变量
  k3s_role: worker
  ```

### 4. `host_vars/`

为特定主机定义变量，适用于个别主机的配置需求。

- **`host_vars/master.yml`**
  
  ```yaml
  ---
  # 主节点特有变量
  k3s_token: "{{ vault_k3s_token }}"
  ```

- **`host_vars/worker1.yml`** 和 **`host_vars/worker2.yml`**
  
  ```yaml
  ---
  # 子节点特有变量
  # 如果有特定的配置，可以在这里定义
  ```

### 5. `roles/`

角色用于封装特定功能的任务、变量、模板等，提升可复用性和模块化。

#### 5.1. `common` 角色

负责安装基础依赖和配置系统。

- **`roles/common/tasks/main.yml`**

  ```yaml
  ---
  - name: 更新 APT 包缓存
    apt:
      update_cache: yes
      cache_valid_time: 600

  - name: 安装必要的软件包
    apt:
      name:
        - curl
        - apt-transport-https
        - ca-certificates
        - software-properties-common
      state: present
  ```

- **`roles/common/vars/main.yml`**

  ```yaml
  ---
  # 可在此定义 common 角色的变量
  ```

- **`roles/common/defaults/main.yml`**

  ```yaml
  ---
  # common 角色的默认变量
  ```

- **`roles/common/meta/main.yml`**

  ```yaml
  ---
  dependencies: []
  ```

- **`roles/common/README.md`**

  ```markdown
  # Common Role

  ## 描述

  该角色用于安装基础依赖和配置系统，适用于所有节点。

  ## 变量

  - 无

  ## 使用示例

  在 Playbook 中引用该角色：
  ​```yaml
  - hosts: all
    roles:
      - common
  ```
  ```
  
  ```
#### 5.2. `master` 角色

负责在主节点上安装 k3s 并生成 Token。

- **`roles/master/tasks/main.yml`**

  ```yaml
  ---
  - name: 安装 k3s 主节点
    shell: |
      curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION={{ k3s_version }} INSTALL_K3S_EXEC="{{ k3s_install_options }}" sh -
    args:
      creates: /etc/rancher/k3s/k3s.yaml
    register: install_k3s
    retries: 3
    delay: 10
    until: install_k3s is succeeded

  - name: 等待 k3s 服务启动
    wait_for:
      port: 6443
      host: "{{ ansible_host }}"
      timeout: 60

  - name: 获取 k3s Token
    command: cat /var/lib/rancher/k3s/server/node-token
    register: k3s_token

  - name: 设置事实 - k3s_token
    set_fact:
      k3s_token: "{{ k3s_token.stdout }}"

  - name: 保存 Token 到 vault
    copy:
      content: "{{ k3s_token.stdout }}"
      dest: /etc/k3s_token
      mode: '0600'
  ```

- **`roles/master/vars/main.yml`**

  ```yaml
  ---
  # master 角色的变量
  ```

- **`roles/master/defaults/main.yml`**

  ```yaml
  ---
  # master 角色的默认变量
  ```

- **`roles/master/meta/main.yml`**

  ```yaml
  ---
  dependencies:
    - common
  ```

- **`roles/master/README.md`**

  ```markdown
  # Master Role

  ## 描述

  该角色用于在主节点上安装和配置 k3s，并生成集群 Token。

  ## 变量

  - `k3s_version`：k3s 的版本号（来自 `group_vars/all.yml`）。
  - `k3s_install_options`：安装选项（来自 `group_vars/all.yml`）。

  ## 使用示例

  在 Playbook 中引用该角色：
  ​```yaml
  - hosts: masters
    roles:
      - master
  ```
  ```
  
  ```
#### 5.3. `worker` 角色

负责在子节点上安装 k3s 并加入集群。

- **`roles/worker/tasks/main.yml`**

  ```yaml
  ---
  - name: 获取主节点的 IP 地址
    set_fact:
      master_ip: "{{ hostvars['master']['ansible_host'] }}"

  - name: 安装 k3s 子节点
    shell: |
      curl -sfL https://get.k3s.io | K3S_URL=https://{{ master_ip }}:6443 K3S_TOKEN={{ hostvars['master']['k3s_token'] }} INSTALL_K3S_VERSION={{ k3s_version }} sh -
    args:
      creates: /etc/rancher/k3s/k3s.yaml
    register: install_k3s_worker
    retries: 3
    delay: 10
    until: install_k3s_worker is succeeded

  - name: 等待 k3s 服务启动
    wait_for:
      port: 6443
      host: "{{ master_ip }}"
      timeout: 60
  ```

- **`roles/worker/vars/main.yml`**

  ```yaml
  ---
  # worker 角色的变量
  ```

- **`roles/worker/defaults/main.yml`**

  ```yaml
  ---
  # worker 角色的默认变量
  ```

- **`roles/worker/meta/main.yml`**

  ```yaml
  ---
  dependencies:
    - common
  ```

- **`roles/worker/README.md`**

  ```markdown
  # Worker Role

  ## 描述

  该角色用于在子节点上安装和配置 k3s，并将其加入到已有的 k3s 集群中。

  ## 变量

  - `k3s_version`：k3s 的版本号（来自 `group_vars/all.yml`）。
  - `k3s_install_options`：安装选项（来自 `group_vars/all.yml`）。
  - `master_ip`：主节点的 IP 地址（自动获取）。
  - `k3s_token`：集群的 Token（来自主节点角色）。

  ## 使用示例

  在 Playbook 中引用该角色：
  ​```yaml
  - hosts: workers
    roles:
      - worker
  ```
  ```
  
  ```
### 6. `files/` 和 `templates/`

用于存放全局静态文件和模板文件。如果有需要复制到所有节点的文件，可以放在这里。

- **`files/`**
  
  ```yaml
  # 例如，一个全局配置文件
  some_global_file.conf
  ```

- **`templates/`**
  
  ```jinja
  # 例如，一个全局模板文件
  some_global_template.j2
  ```

### 7. `vault.yml`

使用 Ansible Vault 加密敏感信息，如 k3s Token。确保该文件在 `.gitignore` 中，避免泄露敏感信息。

```yaml
---
vault_k3s_token: "your_secure_token_here"
```

加密文件：

```bash
ansible-vault encrypt vault.yml
```

在 Playbook 中引用加密文件：

```yaml
vars_files:
  - vault.yml
```

## 三、Playbook 定义

**`playbooks/deploy_k3s.yml`**

```yaml
---
- name: 部署 k3s 主节点
  hosts: masters
  become: yes
  vars_files:
    - ../vault.yml
  roles:
    - common
    - master

- name: 部署 k3s 子节点
  hosts: workers
  become: yes
  roles:
    - common
    - worker
  when: hostvars['master']['k3s_token'] is defined
```

### Playbook 说明

1. **第一个 Play**：部署主节点
   - **主机**：`masters` 组中的主机。
   - **权限提升**：使用 `become: yes` 提升到管理员权限。
   - **变量文件**：加载加密的 `vault.yml` 文件，确保敏感信息安全。
   - **角色**：执行 `common` 和 `master` 角色。

2. **第二个 Play**：部署子节点
   - **主机**：`workers` 组中的主机。
   - **权限提升**：使用 `become: yes` 提升到管理员权限。
   - **角色**：执行 `common` 和 `worker` 角色。
   - **条件**：仅当主节点的 `k3s_token` 被定义时执行，确保主节点已成功部署。

## 四、变量管理与安全

### 1. 使用 Ansible Vault 管理敏感信息

将敏感变量（如 `k3s_token`）存储在加密文件中，确保安全性。

**创建加密文件：**

```bash
ansible-vault create inventory/vault.yml
```

**示例内容：**

```yaml
---
vault_k3s_token: "your_secure_token_here"
```

**在 Playbook 中引用：**

```yaml
vars_files:
  - ../vault.yml
```

### 2. 集中管理变量

利用 `group_vars` 和 `host_vars` 目录，实现变量的分层管理，确保变量的可复用性和可维护性。

- **`group_vars/all.yml`**：定义所有主机共享的全局变量。
- **`group_vars/masters.yml`** 和 **`group_vars/workers.yml`**：定义特定组的变量。
- **`host_vars/master.yml`** 和 **`host_vars/worker1.yml`、`worker2.yml`**：定义特定主机的变量。

## 五、改进与优化

### 1. 使用 Ansible Modules 代替 Shell

尽量使用 Ansible 提供的模块，而不是直接使用 `shell` 或 `command`，以提高 Playbook 的可读性和 idempotency。

**示例改进：安装 k3s 使用 `get_url` 和 `command` 组合**

虽然 k3s 的安装通常通过 `curl` 命令完成，但我们可以优化任务，确保 idempotency 和错误处理。

**`roles/master/tasks/main.yml`**

```yaml
---
- name: 下载 k3s 安装脚本
  get_url:
    url: https://get.k3s.io
    dest: /tmp/k3s_install.sh
    mode: '0755'
  register: download_k3s

- name: 安装 k3s 主节点
  shell: |
    /tmp/k3s_install.sh INSTALL_K3S_VERSION={{ k3s_version }} INSTALL_K3S_EXEC="{{ k3s_install_options }}"
  args:
    creates: /etc/rancher/k3s/k3s.yaml
  when: download_k3s.changed

- name: 等待 k3s 服务启动
  wait_for:
    port: 6443
    host: "{{ ansible_host }}"
    timeout: 60

- name: 获取 k3s Token
  command: cat /var/lib/rancher/k3s/server/node-token
  register: k3s_token

- name: 设置事实 - k3s_token
  set_fact:
    k3s_token: "{{ k3s_token.stdout }}"

- name: 保存 Token 到 vault
  copy:
    content: "{{ k3s_token.stdout }}"
    dest: /etc/k3s_token
    mode: '0600'
```

### 2. 错误处理与重试机制

在安装过程中加入重试机制，提升部署的鲁棒性。

```yaml
- name: 安装 k3s 主节点
  shell: |
    /tmp/k3s_install.sh INSTALL_K3S_VERSION={{ k3s_version }} INSTALL_K3S_EXEC="{{ k3s_install_options }}"
  args:
    creates: /etc/rancher/k3s/k3s.yaml
  register: install_k3s
  retries: 3
  delay: 10
  until: install_k3s is succeeded
```

### 3. 使用 Tags 进行部分执行

通过标签（Tags）对 Playbook 进行部分执行，提高灵活性和效率。

**Playbook 示例：**

```yaml
---
- name: 部署 k3s 主节点
  hosts: masters
  become: yes
  vars_files:
    - ../vault.yml
  roles:
    - common
    - master
  tags: master

- name: 部署 k3s 子节点
  hosts: workers
  become: yes
  roles:
    - common
    - worker
  when: hostvars['master']['k3s_token'] is defined
  tags: worker
```

**执行时指定 Tag：**

```bash
# 仅部署主节点
ansible-playbook -i inventory/hosts.ini playbooks/deploy_k3s.yml --tags master

# 仅部署子节点
ansible-playbook -i inventory/hosts.ini playbooks/deploy_k3s.yml --tags worker
```

### 4. 使用 Handlers 触发服务重启

在角色中定义 Handlers，确保在配置变化时自动重启服务。

**`roles/master/tasks/main.yml`**

```yaml
---
- name: 下载 k3s 安装脚本
  get_url:
    url: https://get.k3s.io
    dest: /tmp/k3s_install.sh
    mode: '0755'
  register: download_k3s

- name: 安装 k3s 主节点
  shell: |
    /tmp/k3s_install.sh INSTALL_K3S_VERSION={{ k3s_version }} INSTALL_K3S_EXEC="{{ k3s_install_options }}"
  args:
    creates: /etc/rancher/k3s/k3s.yaml
  when: download_k3s.changed
  notify: Restart k3s

- name: 等待 k3s 服务启动
  wait_for:
    port: 6443
    host: "{{ ansible_host }}"
    timeout: 60

- name: 获取 k3s Token
  command: cat /var/lib/rancher/k3s/server/node-token
  register: k3s_token

- name: 设置事实 - k3s_token
  set_fact:
    k3s_token: "{{ k3s_token.stdout }}"

- name: 保存 Token 到 vault
  copy:
    content: "{{ k3s_token.stdout }}"
    dest: /etc/k3s_token
    mode: '0600'

handlers:
  - name: Restart k3s
    systemd:
      name: k3s
      state: restarted
```

### 5. 文档化角色与 Playbook

为每个角色和 Playbook 编写详细的 README 文件，说明其用途、变量、依赖等，提升团队协作和代码可维护性。

**`roles/master/README.md`**

```markdown
# Master Role

## 描述

该角色用于在主节点上安装和配置 k3s，并生成集群 Token。

## 变量

- `k3s_version`：k3s 的版本号（来自 `group_vars/all.yml`）。
- `k3s_install_options`：安装选项（来自 `group_vars/all.yml`）。

## 依赖

- `common` 角色

## 使用示例

在 Playbook 中引用该角色：
​```yaml
- hosts: masters
  roles:
    - master
```
```

## 六、安全管理

### 1. 使用 Ansible Vault 加密敏感变量

将敏感信息（如 `k3s_token`）存储在加密文件中，确保安全性。

**创建加密文件：**

​```bash
ansible-vault create inventory/vault.yml
```

**示例内容：**

```yaml
---
vault_k3s_token: "your_secure_token_here"
```

**在 Playbook 中引用加密文件：**

```yaml
vars_files:
  - ../vault.yml
```

### 2. 限制敏感文件访问权限

确保敏感文件（如 `vault.yml` 和 `/etc/k3s_token`）具有严格的权限控制，避免未授权访问。

**示例任务：**

```yaml
- name: 设置 /etc/k3s_token 权限
  file:
    path: /etc/k3s_token
    owner: root
    group: root
    mode: '0600'
```

## 七、执行 Playbook

确保主节点和子节点都在 Inventory 中正确配置，并且控制节点已安装 Ansible 和必要的依赖。

**执行 Playbook：**

```bash
ansible-playbook -i inventory/hosts.ini playbooks/deploy_k3s.yml
```

**部分执行主节点或子节点：**

```bash
# 仅部署主节点
ansible-playbook -i inventory/hosts.ini playbooks/deploy_k3s.yml --tags master

# 仅部署子节点
ansible-playbook -i inventory/hosts.ini playbooks/deploy_k3s.yml --tags worker
```

## 八、验证集群

部署完成后，通过主节点验证集群状态。

**登录主节点：**

```bash
ssh your_username@master_ip
```

**查看节点状态：**

```bash
sudo k3s kubectl get nodes
```

**示例输出：**

```plaintext
NAME        STATUS   ROLES    AGE   VERSION
master      Ready    master   10m   v1.26.3+k3s1
worker1     Ready    <none>   8m    v1.26.3+k3s1
worker2     Ready    <none>   8m    v1.26.3+k3s1
```

## 九、进一步优化与最佳实践

### 1. 使用 Ansible Galaxy 管理角色

将自定义角色上传到 Ansible Galaxy，便于在不同项目中复用和共享。

**上传角色到 Ansible Galaxy：**

```bash
ansible-galaxy role import your_github_username/master
```

### 2. 使用变量验证

利用 Ansible 的 `assert` 模块，确保变量符合预期，避免配置错误。

**示例任务：**

```yaml
- name: 确保 k3s_token 已定义
  assert:
    that:
      - k3s_token is defined
      - k3s_token != ""
    fail_msg: "k3s_token 未定义或为空！"
```

### 3. 集群监控与管理

在部署完成后，配置监控工具（如 Prometheus、Grafana）来监控 k3s 集群的健康状态和性能。

### 4. 使用 CI/CD 流水线自动化部署

将 Ansible Playbook 集成到 CI/CD 流水线中，实现自动化部署和持续集成。

## 十、总结

通过上述重构，你的 Ansible 项目结构更加清晰、模块化，变量管理更为合理，并且遵循了安全最佳实践。以下是关键改进点的总结：

1. **标准化项目结构**：遵循 Ansible 的推荐目录布局，提升可维护性和可复用性。
2. **变量分层管理**：利用 `group_vars` 和 `host_vars` 实现变量的分层管理，确保变量覆盖的正确性。
3. **安全管理**：使用 Ansible Vault 加密敏感信息，并设置文件权限，确保配置的安全性。
4. **模块化角色**：将主节点和子节点的部署逻辑封装到独立角色中，提升 Playbook 的可复用性。
5. **错误处理与重试**：在关键任务中加入重试机制，提升部署的鲁棒性。
6. **使用 Tags**：通过标签实现部分 Playbook 的执行，提高灵活性。
7. **文档化**：为每个角色编写详细的 README 文件，提升团队协作和项目可维护性。
8. **使用 Ansible Modules**：尽量使用 Ansible 提供的模块，替代直接使用 `shell` 或 `command`，提高 Playbook 的可读性和 idempotency。

通过这些改进，你可以更高效地使用 Ansible 自动化部署和管理 k3s 集群。如果在实际操作中遇到任何问题或需要进一步的帮助，欢迎随时提问！