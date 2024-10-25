

在 Ansible 中，**保留关键字**（Reserved Keywords）指的是那些在 Playbooks、模块、变量等不同上下文中具有特殊意义的名称。这些关键字用于定义任务结构、控制流程、设置变量等。如果在自定义变量或其他配置中不小心使用了这些保留关键字，可能会导致意外的行为或错误。因此，了解和避免使用这些保留关键字对于编写稳定且可维护的 Ansible Playbooks 至关重要。

以下是 Ansible 中常见的保留关键字分类及详细说明：

---

## 1. Playbook 结构关键字

这些关键字用于定义 Playbook 的基本结构和行为，通常位于 Playbook 的顶层或特定层级中。

| 关键字          | 说明                                                   |
| --------------- | ------------------------------------------------------ |
| `hosts`         | 指定 Play 应用于哪些主机或主机组。                     |
| `tasks`         | 定义一系列要在目标主机上执行的任务。                   |
| `vars`          | 定义在 Play 或任务中使用的变量。                       |
| `roles`         | 引用和包含预定义的角色（Roles），以组织 Playbook。     |
| `handlers`      | 定义在任务中触发的处理程序（通常用于服务重启等操作）。 |
| `pre_tasks`     | 在主要任务执行之前运行的任务。                         |
| `post_tasks`    | 在主要任务执行之后运行的任务。                         |
| `vars_files`    | 引用外部变量文件。                                     |
| `gather_facts`  | 指定是否收集目标主机的事实信息（默认 `true`）。        |
| `become`        | 控制是否以提升权限（如 sudo）执行任务。                |
| `strategy`      | 指定任务执行的策略（如 `linear`、`free`）。            |
| `environment`   | 设置任务执行时的环境变量。                             |
| `ignore_errors` | 指定是否忽略任务执行中的错误。                         |

**示例：**
```yaml
---
- name: 部署 Web 应用
  hosts: webservers
  become: yes
  vars:
    app_version: "1.2.3"
  tasks:
    - name: 安装 Nginx
      apt:
        name: nginx
        state: present
```

---

## 2. 模块参数关键字

Ansible 模块本身有一组预定义的参数，这些参数在模块调用时具有特定含义。使用这些参数名称时需要遵循模块的规范。

**常见模块及其关键字示例：**

- **`shell` 和 `command` 模块**
  - `cmd` / `command`
  - `creates`
  - `removes`
  - `chdir`
  - `warn`

- **`copy` 模块**
  - `src`
  - `dest`
  - `mode`
  - `owner`
  - `group`
  - `content`

- **`apt` 模块**
  - `name`
  - `state`
  - `update_cache`
  - `upgrade`

**注意：** 每个模块都有自己特定的参数，使用时应参考官方文档。

---

## 3. 变量命名约定

### a. **`ansible_*` 变量**

以 `ansible_` 开头的变量是 Ansible 内部使用的特殊变量，用于存储事实信息、连接参数等。这些变量由 Ansible 自动生成和管理，用户不应随意覆盖或修改。

**常见 `ansible_` 变量：**

- `ansible_host`：目标主机的实际连接地址。
- `ansible_user`：用于连接目标主机的用户名。
- `ansible_port`：连接目标主机的端口号。
- `ansible_ssh_private_key_file`：用于 SSH 连接的私钥文件路径。
- `ansible_facts`：收集的目标主机事实信息，如操作系统、网络接口等。

**示例：**
```ini
[webservers]
web1 ansible_host=192.168.1.10 ansible_user=deploy_user
web2 ansible_host=192.168.1.11 ansible_user=deploy_user
```

### b. **避免使用保留名称**

除了 `ansible_` 前缀外，还有其他一些名称在 Ansible 中具有特殊含义，用户应避免在自定义变量、任务名称等地方使用这些名称，以防止冲突和不可预期的行为。

**常见应避免的名称示例：**

- `hostvars`
- `groups`
- `inventory_hostname`
- `playbook_dir`
- `inventory_dir`

**示例：**
```yaml
# 不推荐使用
vars:
  hostvars: "自定义值"  # 会覆盖 Ansible 内部的 hostvars 字典，导致错误
```

---

## 4. 特殊关键字和保留功能

### a. **`_meta`**

`_meta` 是一个特殊的关键字，通常在动态 Inventory 脚本的输出中使用，用于存储主机变量信息（hostvars）。用户不应在 Playbook 或静态 Inventory 中使用 `_meta` 关键字。

**示例（动态 Inventory 输出）：**
```json
{
  "webservers": {
    "hosts": ["web1.example.com", "web2.example.com"],
    "vars": {
      "http_port": 80
    }
  },
  "_meta": {
    "hostvars": {
      "web1.example.com": {
        "ansible_host": "192.168.1.10"
      },
      "web2.example.com": {
        "ansible_host": "192.168.1.11"
      }
    }
  }
}
```

### b. **`vars_prompt` 和 `vars_files`**

这些关键字用于在 Playbook 执行时提示用户输入变量或引用外部变量文件。它们具有特定的功能和使用场景，应按照 Ansible 规范使用。

**示例：**
```yaml
---
- name: 需要用户输入的 Playbook
  hosts: all
  vars_prompt:
    - name: "admin_password"
      prompt: "请输入管理员密码"
      private: yes
  tasks:
    - name: 设置管理员密码
      user:
        name: admin
        password: "{{ admin_password }}"
```

---

## 5. Ansible 配置文件关键字

Ansible 的配置文件（如 `ansible.cfg`）中也有一组保留关键字，用于配置 Ansible 的行为和特性。常见的配置部分和关键字包括：

| 配置部分                 | 关键字                                   | 说明                            |
| ------------------------ | ---------------------------------------- | ------------------------------- |
| `[defaults]`             | `inventory`                              | 指定默认的 Inventory 文件路径。 |
|                          | `remote_user`                            | 指定默认的远程连接用户。        |
|                          | `private_key_file`                       | 指定默认的 SSH 私钥文件路径。   |
|                          | `roles_path`                             | 指定角色的搜索路径。            |
| `[privilege_escalation]` | `become`、`become_method`、`become_user` | 配置权限提升的相关选项。        |
| `[ssh_connection]`       | `ssh_args`、`control_path`、`pipelining` | 配置 SSH 连接的相关参数。       |
| `[inventory]`            | `enable_plugins`                         | 启用特定的 Inventory 插件。     |
| `[callback]`             | `callback_whitelist`                     | 指定启用的回调插件。            |

**示例 `ansible.cfg`：**
```ini
[defaults]
inventory = ./inventory/hosts
remote_user = deploy_user
private_key_file = ~/.ssh/id_rsa
roles_path = ./roles

[privilege_escalation]
become = yes
become_method = sudo
become_user = root

[ssh_connection]
ssh_args = -o ForwardAgent=yes
pipelining = True
```

**注意：** 配置文件中的关键字必须遵循 Ansible 的规范，错误的关键字或拼写可能导致配置失效或 Ansible 报错。

---

## 6. 其他保留关键字

除了上述分类，还有一些在特定上下文中具有特殊意义的关键字，用户在编写 Playbook 或自定义模块时应避免使用这些名称，以防止冲突。

**常见示例：**

- **`meta`**：用于定义 Playbook 的元数据，如 `meta: end_play`。
- **`include`** 和 **`import`**：用于包含其他任务或 Playbook。
- **`when`**：用于条件判断。
- **`notify`** 和 **`listen`**：用于触发和监听处理程序。
- **`block`**：用于将一组任务组合在一起，并应用共同的条件或错误处理。

**示例：**
```yaml
- name: 使用 meta 关键字
  hosts: all
  tasks:
    - name: 结束 Play
      meta: end_play
```

---

## 7. 最佳实践

### a. **避免使用保留关键字作为变量名**

确保自定义变量不使用上述保留关键字，以避免覆盖或干扰 Ansible 的内部机制。

**不推荐的做法：**
```yaml
vars:
  tasks: "自定义值"  # 会覆盖 Playbook 中的 tasks 关键字，导致错误
```

### b. **遵循命名约定**

使用有意义且不冲突的变量名称，建议使用前缀或特定格式以减少命名冲突的可能性。

**推荐的做法：**
```yaml
vars:
  app_tasks: "部署任务"
  db_config: "数据库配置"
```

### c. **参考官方文档**

在使用新的模块或功能时，参考 [Ansible 官方文档](https://docs.ansible.com/) 了解相关的保留关键字和命名规范。

---

## 8. 总结

Ansible 中的保留关键字在 Playbooks、模块、变量命名及配置文件中具有特殊意义，正确理解和避免使用这些关键字对于编写稳定、高效的自动化脚本至关重要。通过遵循最佳实践、参考官方文档并保持命名的一致性，可以有效减少潜在的冲突和错误，提升 Ansible 项目的可维护性和可靠性。

---

**参考资料：**
- [Ansible 官方文档 - Playbooks](https://docs.ansible.com/ansible/latest/user_guide/playbooks.html)
- [Ansible 官方文档 - Modules](https://docs.ansible.com/ansible/latest/collections/index.html)
- [Ansible 官方文档 - Configuration Settings](https://docs.ansible.com/ansible/latest/installation_guide/intro_configuration.html)

在 Ansible 中，**“all”** 和 **“main”** 这两个词在特定的上下文中具有特殊的含义和用途。了解它们的作用对于有效地组织和管理 Ansible 项目至关重要。以下是对这两个关键字的详细解释：

---

## 1. `all` 在 Ansible 中的特殊含义

### a. **`all` 组**

- **定义与作用**：
  - `all` 是 Ansible 中的一个内置组，包含 Inventory 中定义的所有主机。无论你有多少个主机或主机组，`all` 组都会自动包含它们。
  - 这个组在变量定义和 Playbook 编写中非常有用，尤其是当你需要为所有主机设置全局变量或执行某些任务时。

- **使用场景**：
  - **全局变量**：在 `group_vars/all.yml` 中定义适用于所有主机的变量。
  - **Playbook 应用范围**：在 Playbook 中使用 `hosts: all` 来针对所有主机执行任务。

- **示例**：

  **Inventory 文件（INI 格式）**：
  ```ini
  [webservers]
  web1.example.com
  web2.example.com

  [dbservers]
  db1.example.com
  db2.example.com
  ```

  **`group_vars/all.yml` 文件**：
  ```yaml
  # 适用于所有主机的变量
  ansible_user: common_user
  ansible_ssh_private_key_file: /path/to/common/key
  ntp_server: time.example.com
  ```

  **Playbook 示例**：
  ```yaml
  ---
  - name: 全局配置
    hosts: all
    tasks:
      - name: 确保 NTP 服务已安装
        apt:
          name: ntp
          state: present
  ```

### b. **`all` 在动态 Inventory 中**

- **定义与作用**：
  - 在动态 Inventory 脚本的输出中，`all` 组也存在，用于包含所有主机及其相关变量。
  - 通过动态 Inventory，`all` 组同样可以用于定义全局变量或组织所有主机。

- **示例（JSON 输出）**：
  ```json
  {
    "all": {
      "vars": {
        "ansible_user": "common_user",
        "ansible_ssh_private_key_file": "/path/to/common/key",
        "ntp_server": "time.example.com"
      }
    },
    "webservers": {
      "hosts": ["web1.example.com", "web2.example.com"]
    },
    "dbservers": {
      "hosts": ["db1.example.com", "db2.example.com"]
    },
    "_meta": {
      "hostvars": {
        "web1.example.com": {
          "ansible_host": "192.168.1.10"
        },
        "web2.example.com": {
          "ansible_host": "192.168.1.11"
        },
        "db1.example.com": {
          "ansible_host": "192.168.1.20"
        },
        "db2.example.com": {
          "ansible_host": "192.168.1.21"
        }
      }
    }
  }
  ```

  在上述示例中，`all` 组下的 `vars` 部分定义了适用于所有主机的变量。

### c. **优先级**

- **变量优先级**：
  - `all` 组中的变量优先级低于特定组（如 `webservers`、`dbservers`）中的变量。
  - 如果在 `group_vars/all.yml` 和某个特定组中定义了相同的变量，特定组中的定义将覆盖 `all` 组中的定义。

---

## 2. `main` 在 Ansible 中的特殊含义

### a. **`main` 作为默认文件名**

- **定义与作用**：
  
  - 在 Ansible 的角色（Roles）和某些模块中，`main.yml` 是一个约定的默认文件名。Ansible 会自动加载这些文件，无需在 Playbook 或配置中显式指定它们。
  
- **使用场景**：
  - **角色目录结构**：每个角色通常包含多个子目录，如 `tasks`、`handlers`、`defaults`、`vars` 等，每个子目录下都应该有一个 `main.yml` 文件作为入口点。
  - **自动加载**：当引用一个角色时，Ansible 会自动查找并执行 `tasks/main.yml` 中定义的任务。

- **示例**：

  **角色目录结构**：
  ```
  roles/
  └── webserver/
      ├── tasks/
      │   └── main.yml
      ├── handlers/
      │   └── main.yml
      ├── templates/
      │   └── nginx.conf.j2
      ├── files/
      │   └── index.html
      └── defaults/
          └── main.yml
  ```

  **`roles/webserver/tasks/main.yml` 文件**：
  ```yaml
  ---
  - name: 安装 Nginx
    apt:
      name: nginx
      state: present

  - name: 启动 Nginx 服务
    service:
      name: nginx
      state: started
      enabled: true
  ```

  **Playbook 中引用角色**：
  ```yaml
  ---
  - name: 部署 Web 服务器
    hosts: webservers
    roles:
      - webserver
  ```

  在上述 Playbook 中，Ansible 会自动执行 `roles/webserver/tasks/main.yml` 中定义的任务。

### b. **`main` 在模块和插件中的使用**

- **自定义插件**：
  
- 在编写自定义模块、回调插件或其他扩展时，`main.py` 或 `main.yml` 通常作为入口文件。例如，自定义回调插件可能包含一个 `main.py` 文件，Ansible 会自动加载它。
  
- **示例**：

  **自定义回调插件目录结构**：
  ```
  callback_plugins/
  └── my_callback/
      └── main.py
  ```

  在 `ansible.cfg` 中启用自定义回调插件：
  ```ini
  [defaults]
  callback_plugins = ./callback_plugins
  callback_whitelist = my_callback
  ```

### c. **命名规范的重要性**

- **一致性**：
  - 使用 `main.yml` 作为默认文件名有助于保持角色和插件的目录结构一致，便于团队协作和代码维护。

- **自动化加载**：
  - 遵循命名规范使得 Ansible 能够自动识别和加载相应的文件，减少了手动配置的复杂性。

---

## 3. 总结对比

| 特性         | `all`                                      | `main`                                          |
| ------------ | ------------------------------------------ | ----------------------------------------------- |
| **类型**     | 内置主机组                                 | 默认文件名（在角色和插件中）                    |
| **作用范围** | 包含 Inventory 中的所有主机                | 作为角色或插件的入口文件                        |
| **使用场景** | 定义全局变量、针对所有主机执行任务         | 组织角色的任务、处理程序、变量等，自动加载任务  |
| **优先级**   | 低于特定组变量                             | 不适用于变量优先级，作为文件命名约定使用        |
| **示例位置** | `group_vars/all.yml` 或动态 Inventory 脚本 | 角色的 `tasks/main.yml`、`handlers/main.yml` 等 |
| **自动加载** | 无（需要在 Playbook 中明确引用）           | 自动加载，无需在 Playbook 中显式指定            |

---

## 4. 最佳实践

### a. **使用 `all` 组进行全局配置**

- **集中管理**：
  
  - 将所有主机通用的变量放在 `group_vars/all.yml` 中，便于统一管理和修改。
  
- **示例**：
  ```yaml
  # group_vars/all.yml
  ansible_user: common_user
  ansible_ssh_private_key_file: /path/to/common/key
  ntp_server: time.example.com
  ```

### b. **遵循 `main` 文件命名约定**

- **角色开发**：
  
  - 在开发角色时，确保每个子目录（如 `tasks`、`handlers`、`defaults` 等）都有一个 `main.yml` 文件，作为该部分的入口。
  
- **示例**：
  ```
  roles/
  └── database/
      ├── tasks/
      │   └── main.yml
      ├── handlers/
      │   └── main.yml
      ├── defaults/
      │   └── main.yml
      └── vars/
          └── main.yml
  ```

### c. **避免在自定义变量中使用 `all` 和 `main` 作为变量名**

- **防止冲突**：
  
- 不要将 `all` 和 `main` 用作自定义变量名，以避免与 Ansible 的内置功能或约定冲突。
  
- **不推荐的做法**：
  ```yaml
  vars:
    all: "自定义值"    # 会覆盖内置的 all 组
    main: "自定义值"   # 可能与角色中的 main.yml 冲突
  ```

- **推荐的做法**：
  ```yaml
  vars:
    app_all: "自定义值"
    main_config: "自定义值"
  ```

---

## 5. 参考资料

- [Ansible 官方文档 - Inventories](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html)
- [Ansible 官方文档 - Group Vars](https://docs.ansible.com/ansible/latest/user_guide/intro_variables.html#organizing-variables)
- [Ansible 官方文档 - Roles](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html)
- [Ansible 官方文档 - Variable Precedence](https://docs.ansible.com/ansible/latest/user_guide/playbook_variables.html#variable-precedence-where-should-i-put-a-variable)

---

通过理解和正确使用 `all` 和 `main`，你可以更高效地组织 Ansible 项目，确保 Playbook 的可维护性和可扩展性。同时，遵循 Ansible 的命名约定和最佳实践，有助于避免潜在的配置冲突和错误，提升自动化运维的稳定性。

在使用 Ansible 构建环境时，如果遇到某些软件在安装后需要重启系统的情况，可以通过以下几种方式来处理重启后继续执行的问题：

### 方法 1: `reboot` 模块

Ansible 提供了专门的 `reboot` 模块用于处理需要重启的情况。你可以在安装完需要重启的软件后，使用该模块重启系统，并设置任务在重启后继续执行。

示例 Playbook：

```yaml
---
- hosts: all
  become: yes
  tasks:
    - name: 安装需要重启的软件
      apt:
        name: some-package
        state: present

    - name: 重启系统
      reboot:
        reboot_timeout: 3600  # 可以根据系统启动时间调整

    - name: 等待系统重启并可达
      wait_for_connection:
        timeout: 300

    - name: 重启后继续执行的任务
      shell: echo "系统已重启，继续执行其他任务"
```

### 方法 2: `meta: end_play` 和 `meta: reset_connection`

如果你希望在重启后，继续执行未完成的部分任务，可以使用 `meta` 模块中的 `end_play` 来终止当前 Playbook 的执行。然后在重启后重新运行 Playbook，并在需要重启的节点恢复任务执行。

#### 1. 终止当前任务：

```yaml
---
- hosts: all
  become: yes
  tasks:
    - name: 安装需要重启的软件
      apt:
        name: some-package
        state: present

    - name: 重启系统
      reboot:
        reboot_timeout: 3600

    - name: 停止执行 Playbook
      meta: end_play
```

#### 2. 重启后运行的 Playbook：

你可以在之后重新运行 Playbook，它会自动跳过已完成的任务，继续从未完成的部分执行。

```yaml
---
- hosts: all
  become: yes
  tasks:
    - name: 重启后继续执行的任务
      shell: echo "系统已重启，继续执行其他任务"
```

### 方法 3: `ignore_errors` 和 `until`

在某些情况下，你可能希望在重启系统后自动重新连接，并在重启期间等待一段时间。可以使用 `ignore_errors` 和 `until` 来确保任务在重启期间不会中断：

```yaml
---
- hosts: all
  become: yes
  tasks:
    - name: 安装需要重启的软件
      apt:
        name: some-package
        state: present

    - name: 重启系统
      reboot:
        reboot_timeout: 3600

    - name: 等待系统重启并可达
      wait_for_connection:
        timeout: 300
      register: result
      until: result is success
      retries: 10
      delay: 60

    - name: 重启后继续执行的任务
      shell: echo "系统已重启，继续执行其他任务"
```

### 方法 4: 使用 `ansible-playbook` 的 `--start-at-task`

如果想手动控制从重启后某个任务开始执行，可以使用 `ansible-playbook` 的 `--start-at-task` 选项。先运行 Playbook 到重启点，重启系统后手动指定从重启后的任务开始：

```bash
ansible-playbook playbook.yml --start-at-task="重启后继续执行的任务"
```

这几种方法可以根据你的具体场景进行调整。如果你需要完全自动化的流程，推荐使用 `reboot` 模块和 `wait_for_connection` 来自动处理重启和继续任务。

Ansible 提供了多种方式定义和使用变量，以提高灵活性和可维护性。以下是 Ansible 中 22 种使用变量的方式：

### 1. **命令行定义 (`--extra-vars`)**
   通过运行 playbook 时使用 `--extra-vars` 选项在命令行上传递变量。
   ```bash
   ansible-playbook playbook.yml --extra-vars "var_name=value"
   ```

### 2. **Playbook 变量**
   在 playbook 中直接定义变量。
   ```yaml
   ---
   - hosts: all
     vars:
       var_name: value
   ```

### 3. **主机变量 (`host_vars`)**
   在 `host_vars` 目录中为特定主机定义的变量。
   ```
   host_vars/
     hostname.yml
   ```
   ```yaml
   # host_vars/hostname.yml
   var_name: value
   ```

### 4. **组变量 (`group_vars`)**
   在 `group_vars` 目录中为一组主机定义变量。
   ```
   group_vars/
     groupname.yml
   ```
   ```yaml
   # group_vars/groupname.yml
   var_name: value
   ```

### 5. **Inventory 文件中定义的变量**
   在 `inventory` 文件中定义主机或组的变量。
   ```ini
   [groupname]
   hostname var_name=value
   ```

### 6. **角色变量 (`roles/role_name/vars/main.yml`)**
   在角色的 `vars/main.yml` 文件中定义变量。
   ```yaml
   # roles/role_name/vars/main.yml
   var_name: value
   ```

### 7. **角色默认变量 (`roles/role_name/defaults/main.yml`)**
   在角色的 `defaults/main.yml` 文件中定义默认变量。
   ```yaml
   # roles/role_name/defaults/main.yml
   var_name: value
   ```

### 8. **任务中的变量 (`vars`)**
   在单个任务中直接定义变量。
   ```yaml
   - name: Example Task
     command: echo {{ var_name }}
     vars:
       var_name: value
   ```

### 9. **任务中的 `set_fact`**
   使用 `set_fact` 动态设置变量。
   ```yaml
   - name: Set variable using set_fact
     set_fact:
       var_name: value
   ```

### 10. **从文件加载变量 (`vars_files`)**
   从外部文件中加载变量。
   ```yaml
   - hosts: all
     vars_files:
       - vars.yml
   ```
   ```yaml
   # vars.yml
   var_name: value
   ```

### 11. **使用 `include_vars` 导入变量**
   在 playbook 或任务中使用 `include_vars` 动态导入变量文件。
   ```yaml
   - name: Load variables from a file
     include_vars: vars.yml
   ```

### 12. **从 YAML 文件动态导入 (`vars_from_file`)**
   使用 Jinja2 模板语法来动态加载 YAML 文件中的变量。
   ```yaml
   vars:
     var_name: "{{ lookup('file', 'vars.yml') }}"
   ```

### 13. **通过 `with_items` 定义变量**
   在循环中定义变量。
   ```yaml
   - name: Loop through items
     debug:
       msg: "{{ item }}"
     with_items:
       - var1
       - var2
   ```

### 14. **注册变量 (`register`)**
   将任务的输出注册为变量。
   ```yaml
   - name: Register command output
     command: echo "Hello"
     register: command_output
   ```

### 15. **Facts 变量**
   使用 `gather_facts` 收集的系统信息。
   ```yaml
   - hosts: all
     gather_facts: yes
   ```

### 16. **Jinja2 模板中的变量**
   在 Jinja2 模板文件中使用 Ansible 变量。
   ```jinja2
   Hello, {{ var_name }}
   ```

### 17. **环境变量**
   通过 `environment` 关键字在任务中使用环境变量。
   ```yaml
   - name: Use environment variable
     command: echo $MY_VAR
     environment:
       MY_VAR: "value"
   ```

### 18. **角色中的 `include_vars`**
   在角色中使用 `include_vars` 导入变量。
   ```yaml
   - name: Include vars in a role
     include_vars: vars.yml
   ```

### 19. **`vars_prompt` 提示输入**
   运行 playbook 时提示用户输入变量。
   ```yaml
   - hosts: all
     vars_prompt:
       - name: "user_input"
         prompt: "Please enter a value"
         private: no
   ```

### 20. **加密变量 (`ansible-vault`)**
   使用 `ansible-vault` 加密的变量。
   ```bash
   ansible-vault encrypt vars.yml
   ```

### 21. **外部脚本生成的变量**
   使用 `vars_plugins` 调用外部脚本来生成变量。
   ```yaml
   vars:
     var_name: "{{ lookup('pipe', 'my_script.sh') }}"
   ```

### 22. **条件变量 (`when`)**
   在特定条件下根据 `when` 语句使用变量。
   ```yaml
   - name: Conditional task
     command: echo "Running"
     when: var_name == 'value'
   ```

这些方法涵盖了 Ansible 中管理和使用变量的多种途径，可以根据具体需求灵活选择最合适的方式。

Ansible 中插件体系非常灵活，支持许多插件类型。常用的插件类型包括：模块插件（Module Plugins）、回调插件（Callback Plugins）、连接插件（Connection Plugins）、过滤器插件（Filter Plugins）、变量插件（Vars Plugins）等。

以下是常用的 50 种插件，按照使用频率从高到低排序，涵盖了不同的插件类型：

### 1. **Yum**
   用于管理 RedHat 系系统的包管理器。
   ```yaml
   - name: Install a package
     yum:
       name: httpd
       state: present
   ```

### 2. **Apt**
   用于管理 Debian 系系统的包管理器。
   ```yaml
   - name: Install a package
     apt:
       name: nginx
       state: present
   ```

### 3. **Copy**
   用于将文件从控制节点复制到目标节点。
   ```yaml
   - name: Copy a file to the remote server
     copy:
       src: /local/path
       dest: /remote/path
   ```

### 4. **Template**
   用于基于 Jinja2 模板渲染配置文件。
   ```yaml
   - name: Deploy configuration file
     template:
       src: /local/template.j2
       dest: /remote/path
   ```

### 5. **Service**
   用于管理服务的启动、停止和重启。
   ```yaml
   - name: Restart web service
     service:
       name: httpd
       state: restarted
   ```

### 6. **Shell**
   用于运行 Shell 命令。
   ```yaml
   - name: Run a shell command
     shell: "ls -al /tmp"
   ```

### 7. **Command**
   运行命令，但不会通过 shell 解释符。
   ```yaml
   - name: Run a command
     command: /usr/bin/uptime
   ```

### 8. **User**
   用于管理用户和用户组。
   ```yaml
   - name: Create a new user
     user:
       name: john
       state: present
   ```

### 9. **File**
   用于管理文件和目录的属性（权限、所有者、符号链接等）。
   ```yaml
   - name: Ensure the directory is present
     file:
       path: /path/to/directory
       state: directory
   ```

### 10. **Lineinfile**
   在文件中特定位置插入或修改行内容。
   ```yaml
   - name: Ensure a line in a file
     lineinfile:
       path: /etc/sysctl.conf
       line: "net.ipv4.ip_forward=1"
   ```

### 11. **Blockinfile**
   插入多行文本到文件中指定块。
   ```yaml
   - name: Insert a block of text into a file
     blockinfile:
       path: /etc/some_config_file
       block: |
         [some_block]
         setting1=value1
         setting2=value2
   ```

### 12. **Git**
   管理 Git 版本控制系统中的代码库。
   ```yaml
   - name: Clone a Git repository
     git:
       repo: "https://github.com/user/repo.git"
       dest: /path/to/destination
   ```

### 13. **Stat**
   用于检查远程文件的状态（如存在、权限等）。
   ```yaml
   - name: Check if a file exists
     stat:
       path: /path/to/file
     register: file_status
   ```

### 14. **Debug**
   用于调试任务输出。
   ```yaml
   - name: Print debug information
     debug:
       msg: "The value of the variable is {{ some_variable }}"
   ```

### 15. **Setup**
   收集远程系统的 `facts`（系统信息，如内存、磁盘、网络等）。
   ```yaml
   - name: Gather system facts
     setup:
   ```

### 16. **Find**
   查找指定目录下的文件。
   ```yaml
   - name: Find all .log files
     find:
       paths: /var/log
       patterns: "*.log"
   ```

### 17. **Get_url**
   从指定 URL 下载文件。
   ```yaml
   - name: Download a file from a URL
     get_url:
       url: http://example.com/file.tar.gz
       dest: /tmp/file.tar.gz
   ```

### 18. **Unarchive**
   解压缩文件（如 tar、zip）。
   ```yaml
   - name: Unarchive a file
     unarchive:
       src: /local/file.tar.gz
       dest: /remote/destination
   ```

### 19. **Cron**
   管理 crontab 任务调度。
   ```yaml
   - name: Add a cron job
     cron:
       name: "backup job"
       minute: "0"
       hour: "2"
       job: "/usr/local/bin/backup.sh"
   ```

### 20. **Wait_for**
   等待某个端口、文件或超时条件满足。
   ```yaml
   - name: Wait for port 80 to become open
     wait_for:
       port: 80
       host: 127.0.0.1
       timeout: 300
   ```

### 21. **Reboot**
   重新启动远程主机，并等待它再次上线。
   ```yaml
   - name: Reboot the server
     reboot:
   ```

### 22. **Pip**
   用于安装 Python 包。
   ```yaml
   - name: Install a Python package
     pip:
       name: flask
   ```

### 23. **Hosts**
   管理 `/etc/hosts` 文件的条目。
   ```yaml
   - name: Add entry to /etc/hosts
     hosts:
       name: "127.0.0.1 myhostname"
       state: present
   ```

### 24. **Firewalld**
   管理 firewalld 服务。
   ```yaml
   - name: Open port 8080
     firewalld:
       port: 8080/tcp
       permanent: true
       state: enabled
   ```

### 25. **Postgresql_db**
   管理 PostgreSQL 数据库。
   ```yaml
   - name: Create a PostgreSQL database
     postgresql_db:
       name: mydb
   ```

### 26. **Postgresql_user**
   管理 PostgreSQL 用户。
   ```yaml
   - name: Create a PostgreSQL user
     postgresql_user:
       name: myuser
       password: secret
   ```

### 27. **Systemd**
   用于管理 systemd 服务。
   ```yaml
   - name: Restart a systemd service
     systemd:
       name: nginx
       state: restarted
   ```

### 28. **Hostname**
   设置或修改主机名。
   ```yaml
   - name: Set the hostname
     hostname:
       name: myserver
   ```

### 29. **Synchronized**
   同步两个目录的内容。
   ```yaml
   - name: Synchronize files between two hosts
     synchronize:
       src: /source/path/
       dest: /destination/path/
   ```

### 30. **Ufw**
   管理 Uncomplicated Firewall (UFW)。
   ```yaml
   - name: Allow SSH through the firewall
     ufw:
       rule: allow
       name: OpenSSH
   ```

### 31. **Lvol**
   管理 LVM 逻辑卷。
   ```yaml
   - name: Create a logical volume
     lvol:
       vg: myvg
       lv: mylv
       size: 10G
   ```

### 32. **Mount**
   挂载或卸载文件系统。
   ```yaml
   - name: Mount a filesystem
     mount:
       path: /mnt/data
       src: /dev/sdb1
       fstype: ext4
       state: mounted
   ```

### 33. **Docker_container**
   管理 Docker 容器。
   ```yaml
   - name: Start a Docker container
     docker_container:
       name: my_container
       image: nginx
       state: started
   ```

### 34. **Docker_image**
   管理 Docker 镜像。
   ```yaml
   - name: Pull a Docker image
     docker_image:
       name: nginx
       source: pull
   ```

### 35. **Ansible.builtin.include_tasks**
   在 playbook 中动态包含任务文件。
   ```yaml
   - name: Include task list
     include_tasks: tasks.yml
   ```

### 36. **Firewall**
   管理 Linux 防火墙（iptables）。
   ```yaml
   - name: Add firewall rule
     firewall:
       action: insert
       chain: INPUT
       protocol: tcp
       dport: 22
       jump: ACCEPT
   ```

### 37. **Modprobe**
   管理内核模块。
   ```yaml
   - name: Load a kernel module
     modprobe:
       name: nf_conntrack
       state: present
   ```



### 38. **Lxd_container**
   管理 LXD 容器。
   ```yaml
   - name: Create LXD container
     lxd_container:
       name: mycontainer
       state: started
   ```

### 39. **Openvswitch_bridge**
   管理 Open vSwitch 桥接。
   ```yaml
   - name: Create OVS bridge
     openvswitch_bridge:
       bridge: br-int
       state: present
   ```

### 40. **Mysql_db**
   管理 MySQL 数据库。
   ```yaml
   - name: Create a MySQL database
     mysql_db:
       name: mydb
   ```

### 41. **Mysql_user**
   管理 MySQL 用户。
   ```yaml
   - name: Create a MySQL user
     mysql_user:
       name: myuser
       password: secret
   ```

### 42. **Vsftpd**
   管理 vsftpd 服务。
   ```yaml
   - name: Ensure vsftpd is installed and running
     apt:
       name: vsftpd
       state: present
   ```

### 43. **Httpd**
   管理 Apache HTTP Server 服务。
   ```yaml
   - name: Ensure httpd is running
     service:
       name: httpd
       state: started
   ```

### 44. **Terraform**
   集成 Terraform，自动化基础设施管理。
   ```yaml
   - name: Run Terraform
     terraform:
       project_path: /path/to/terraform/project
       state: present
   ```

### 45. **Cpanm**
   管理 Perl 模块安装。
   ```yaml
   - name: Install Perl modules
     cpanm:
       name: Dancer
   ```

### 46. **Zypper**
   管理 openSUSE 系统上的软件包。
   ```yaml
   - name: Install a package on openSUSE
     zypper:
       name: nginx
       state: present
   ```

### 47. **Facter**
   从 Puppet Facter 工具收集 `facts` 信息。
   ```yaml
   - name: Gather facts from Facter
     facter:
   ```

### 48. **Openstack**
   管理 OpenStack 资源。
   ```yaml
   - name: Create OpenStack instance
     os_server:
       name: my_instance
       image: cirros
       flavor: m1.tiny
   ```

### 49. **K8s**
   管理 Kubernetes 集群中的资源。
   ```yaml
   - name: Create a Kubernetes deployment
     k8s:
       definition:
         apiVersion: apps/v1
         kind: Deployment
         metadata:
           name: myapp
   ```

### 50. **Nxos**
   管理 Cisco NX-OS 网络设备。
   ```yaml
   - name: Ensure an interface is up
     nxos_interface:
       interface: Ethernet1/1
       state: up
   ```

这份列表基于 Ansible 常用功能的实际场景使用，涵盖了系统管理、网络设备配置、云平台管理等多方面的插件。

当然，我可以帮助你编写一个专业且安全的 Ansible Playbook，用于对刚安装好的 Ubuntu 系统进行加固和初始化，同时安装 Docker 及常用的网络调试工具。以下是详细的步骤和脚本说明。

## 目录结构

首先，建议按照以下目录结构组织你的 Ansible 项目：

```
ubuntu_hardening/
├── inventory
├── playbook.yml
├── roles/
│   ├── common/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── vars/
│   │       └── main.yml
│   ├── docker/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── vars/
│   │       └── main.yml
│   └── security/
│       ├── tasks/
│       │   └── main.yml
│       └── vars/
│           └── main.yml
```

## 步骤一：安装 Ansible

确保你的控制节点（运行 Ansible 的机器）已安装 Ansible。你可以使用以下命令在 Ubuntu 上安装 Ansible：

```bash
sudo apt update
sudo apt install ansible -y
```

## 步骤二：配置 Inventory 文件

创建一个 `inventory` 文件，定义目标主机的信息。例如，如果你的 Ubuntu 服务器 IP 地址是 `192.168.1.100`：

```ini
[ubuntu_servers]
192.168.1.100 ansible_user=your_username ansible_ssh_private_key_file=~/.ssh/id_rsa
```

**说明：**
- `ansible_user`：用于连接目标主机的用户名。
- `ansible_ssh_private_key_file`：用于 SSH 连接的私钥路径。

## 步骤三：编写 Playbook

创建 `playbook.yml` 文件，并添加以下内容：

```yaml
---
- name: Ubuntu 系统加固与初始化
  hosts: ubuntu_servers
  become: yes
  vars_files:
    - roles/common/vars/main.yml
    - roles/docker/vars/main.yml
    - roles/security/vars/main.yml
  roles:
    - common
    - docker
    - security
```

## 步骤四：编写角色

### 1. **Common 角色**

负责系统更新、安装常用网络调试工具等。

#### `roles/common/vars/main.yml`

```yaml
---
common_packages:
  - curl
  - wget
  - vim
  - git
  - net-tools
  - ufw
  - fail2ban
```

#### `roles/common/tasks/main.yml`

```yaml
---
- name: 更新 APT 包缓存
  apt:
    update_cache: yes
    cache_valid_time: 600

- name: 升级所有包到最新版本
  apt:
    upgrade: dist
    autoremove: yes
    autoclean: yes

- name: 安装常用软件包
  apt:
    name: "{{ common_packages }}"
    state: present
    update_cache: yes

- name: 配置 UFW 默认策略
  ufw:
    default: "{{ item }}"
  loop:
    - deny incoming
    - allow outgoing

- name: 允许 SSH 通过 UFW
  ufw:
    rule: allow
    name: OpenSSH

- name: 启用 UFW
  ufw:
    state: enabled
    logging: 'on'

- name: 启用并启动 Fail2Ban 服务
  systemd:
    name: fail2ban
    enabled: yes
    state: started
```

### 2. **Docker 角色**

负责安装 Docker 和 Docker Compose。

#### `roles/docker/vars/main.yml`

```yaml
---
docker_packages:
  - docker.io
  - docker-compose
```

#### `roles/docker/tasks/main.yml`

```yaml
---
- name: 安装 Docker
  apt:
    name: "{{ docker_packages }}"
    state: present
    update_cache: yes

- name: 启动并启用 Docker 服务
  systemd:
    name: docker
    enabled: yes
    state: started

- name: 添加用户到 Docker 组
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes
```

### 3. **Security 角色**

负责 SSH 配置、创建非 root 用户等。

#### `roles/security/vars/main.yml`

```yaml
---
ssh_port: 22
allowed_users:
  - your_username
```

#### `roles/security/tasks/main.yml`

```yaml
---
- name: 备份原始 SSH 配置
  copy:
    src: /etc/ssh/sshd_config
    dest: /etc/ssh/sshd_config.bak
    remote_src: yes
    backup: yes

- name: 配置 SSH 以增强安全性
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: "^{{ item.regexp }}"
    line: "{{ item.line }}"
    state: present
  loop:
    - { regexp: 'PermitRootLogin', line: 'PermitRootLogin no' }
    - { regexp: 'PasswordAuthentication', line: 'PasswordAuthentication no' }
    - { regexp: '^#?Port', line: "Port {{ ssh_port }}" }
    - { regexp: 'AllowUsers', line: "AllowUsers {{ allowed_users | join(' ') }}" }

- name: 重启 SSH 服务以应用更改
  systemd:
    name: ssh
    state: restarted
    enabled: yes

- name: 创建新用户（如果不存在）
  user:
    name: "{{ ansible_user }}"
    shell: /bin/bash
    create_home: yes
    state: present

- name: 设置用户密码（可选，推荐使用 SSH key）
  # 取消注释以下任务并设置密码
  # user:
  #   name: "{{ ansible_user }}"
  #   password: "{{ 'your_password' | password_hash('sha512') }}"
  #   update_password: always
```

**注意：**
- 请将 `your_username` 替换为你实际使用的用户名。
- 为了更高的安全性，建议使用 SSH 密钥认证，避免使用密码认证。

## 步骤五：运行 Playbook

### 1. **检查连接**

在运行 Playbook 之前，确保控制节点可以通过 SSH 无密码访问目标主机。你可以使用以下命令测试连接：

```bash
ansible -i inventory ubuntu_servers -m ping
```

你应该看到类似以下的输出：

```
192.168.1.100 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### 2. **运行 Playbook**

执行以下命令运行 Playbook：

```bash
ansible-playbook -i inventory playbook.yml
```

### 3. **验证结果**

- **系统更新和软件安装**：登录到目标 Ubuntu 服务器，检查已安装的软件包是否正确。
  
  ```bash
  dpkg -l | grep -E "curl|wget|vim|git|net-tools|ufw|fail2ban|docker|docker-compose"
  ```

- **防火墙配置**：检查 UFW 状态和规则。
  
  ```bash
  sudo ufw status verbose
  ```

- **SSH 配置**：确保 SSH 配置已更新，禁止 root 登录和密码认证。

  ```bash
  sudo grep -E "PermitRootLogin|PasswordAuthentication|Port|AllowUsers" /etc/ssh/sshd_config
  ```

- **Docker 安装**：验证 Docker 是否已安装并正在运行。

  ```bash
  sudo systemctl status docker
  docker --version
  docker-compose --version
  ```

## Playbook 完整代码

为了方便，你可以将以下完整代码复制到相应的文件中。

### `inventory`

```ini
[ubuntu_servers]
192.168.1.100 ansible_user=your_username ansible_ssh_private_key_file=~/.ssh/id_rsa
```

### `playbook.yml`

```yaml
---
- name: Ubuntu 系统加固与初始化
  hosts: ubuntu_servers
  become: yes
  vars_files:
    - roles/common/vars/main.yml
    - roles/docker/vars/main.yml
    - roles/security/vars/main.yml
  roles:
    - common
    - docker
    - security
```

### `roles/common/vars/main.yml`

```yaml
---
common_packages:
  - curl
  - wget
  - vim
  - git
  - net-tools
  - ufw
  - fail2ban
```

### `roles/common/tasks/main.yml`

```yaml
---
- name: 更新 APT 包缓存
  apt:
    update_cache: yes
    cache_valid_time: 600

- name: 升级所有包到最新版本
  apt:
    upgrade: dist
    autoremove: yes
    autoclean: yes

- name: 安装常用软件包
  apt:
    name: "{{ common_packages }}"
    state: present
    update_cache: yes

- name: 配置 UFW 默认策略
  ufw:
    default: "{{ item }}"
  loop:
    - deny incoming
    - allow outgoing

- name: 允许 SSH 通过 UFW
  ufw:
    rule: allow
    name: OpenSSH

- name: 启用 UFW
  ufw:
    state: enabled
    logging: 'on'

- name: 启用并启动 Fail2Ban 服务
  systemd:
    name: fail2ban
    enabled: yes
    state: started
```

### `roles/docker/vars/main.yml`

```yaml
---
docker_packages:
  - docker.io
  - docker-compose
```

### `roles/docker/tasks/main.yml`

```yaml
---
- name: 安装 Docker
  apt:
    name: "{{ docker_packages }}"
    state: present
    update_cache: yes

- name: 启动并启用 Docker 服务
  systemd:
    name: docker
    enabled: yes
    state: started

- name: 添加用户到 Docker 组
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes
```

### `roles/security/vars/main.yml`

```yaml
---
ssh_port: 22
allowed_users:
  - your_username
```

### `roles/security/tasks/main.yml`

```yaml
---
- name: 备份原始 SSH 配置
  copy:
    src: /etc/ssh/sshd_config
    dest: /etc/ssh/sshd_config.bak
    remote_src: yes
    backup: yes

- name: 配置 SSH 以增强安全性
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: "^{{ item.regexp }}"
    line: "{{ item.line }}"
    state: present
  loop:
    - { regexp: 'PermitRootLogin', line: 'PermitRootLogin no' }
    - { regexp: 'PasswordAuthentication', line: 'PasswordAuthentication no' }
    - { regexp: '^#?Port', line: "Port {{ ssh_port }}" }
    - { regexp: 'AllowUsers', line: "AllowUsers {{ allowed_users | join(' ') }}" }

- name: 重启 SSH 服务以应用更改
  systemd:
    name: ssh
    state: restarted
    enabled: yes

- name: 创建新用户（如果不存在）
  user:
    name: "{{ ansible_user }}"
    shell: /bin/bash
    create_home: yes
    state: present

- name: 设置用户密码（可选，推荐使用 SSH key）
  # 取消注释以下任务并设置密码
  # user:
  #   name: "{{ ansible_user }}"
  #   password: "{{ 'your_password' | password_hash('sha512') }}"
  #   update_password: always
```

## 进一步的安全加固建议

上述 Playbook 已涵盖基本的系统加固和初始化步骤，但为了进一步增强系统安全性，你还可以考虑以下措施：

1. **安装并配置 SELinux 或 AppArmor**：增强系统的访问控制。
2. **启用日志审计**：使用 `auditd` 监控系统事件。
3. **限制 sudo 权限**：确保只有必要的用户具有 sudo 权限。
4. **自动化安全更新**：确保系统自动安装安全补丁。
5. **监控与告警**：集成监控工具如 Prometheus 和 Grafana。

## 总结

通过上述步骤，你可以使用 Ansible 自动化地对 Ubuntu 系统进行加固和初始化，并安装 Docker 及常用的网络调试工具。这样不仅提高了配置的一致性和可重复性，还显著提升了系统的安全性和管理效率。如果有任何问题或需要进一步的帮助，请随时提问！

好的，以下是在之前提供的 Ansible Playbook 基础上，进一步加强系统安全性的修改和新增代码。我们将按照以下几点进行增强：

1. **安装并配置 AppArmor**
2. **安装并配置 auditd 进行日志审计**
3. **限制 sudo 权限**
4. **启用自动安全更新**

## 修改和新增内容概述

### 1. **安装并配置 AppArmor**
Ubuntu 默认使用 AppArmor 作为安全模块，但我们可以确保它已启用并配置为强制模式。

### 2. **安装并配置 auditd 进行日志审计**
`auditd` 是一个强大的日志审计工具，可以监控系统事件。

### 3. **限制 sudo 权限**
确保只有必要的用户具有 sudo 权限，并限制 sudoers 文件的访问。

### 4. **启用自动安全更新**
配置系统自动安装安全更新，以确保系统始终保持最新和安全。

### 目录结构更新

```
ubuntu_hardening/
├── inventory
├── playbook.yml
├── roles/
│   ├── common/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── vars/
│   │       └── main.yml
│   ├── docker/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── vars/
│   │       └── main.yml
│   └── security/
│       ├── tasks/
│       │   └── main.yml
│       └── vars/
│           └── main.yml
```

## 具体修改和新增的代码

### 1. **修改 `roles/common/tasks/main.yml` 以确保 AppArmor 启用**

```yaml
# roles/common/tasks/main.yml

---
- name: 更新 APT 包缓存
  apt:
    update_cache: yes
    cache_valid_time: 600

- name: 升级所有包到最新版本
  apt:
    upgrade: dist
    autoremove: yes
    autoclean: yes

- name: 安装常用软件包
  apt:
    name: "{{ common_packages }}"
    state: present
    update_cache: yes

- name: 安装 AppArmor
  apt:
    name: apparmor
    state: present
    update_cache: yes

- name: 启用并启动 AppArmor 服务
  systemd:
    name: apparmor
    enabled: yes
    state: started

- name: 设置 AppArmor 为强制模式
  command: aa-enforce /etc/apparmor.d/*
  args:
    warn: false

- name: 配置 UFW 默认策略
  ufw:
    default: "{{ item }}"
  loop:
    - deny incoming
    - allow outgoing

- name: 允许 SSH 通过 UFW
  ufw:
    rule: allow
    name: OpenSSH

- name: 启用 UFW
  ufw:
    state: enabled
    logging: 'on'

- name: 启用并启动 Fail2Ban 服务
  systemd:
    name: fail2ban
    enabled: yes
    state: started
```

### 2. **新增 `roles/security/tasks/auditd.yml` 来安装和配置 auditd**

首先，在 `roles/security/tasks/` 目录下创建一个新的任务文件 `auditd.yml`。

```yaml
# roles/security/tasks/auditd.yml

---
- name: 安装 auditd
  apt:
    name: auditd
    state: present
    update_cache: yes

- name: 启用并启动 auditd 服务
  systemd:
    name: auditd
    enabled: yes
    state: started

- name: 配置 auditd 规则
  copy:
    src: audit.rules
    dest: /etc/audit/rules.d/audit.rules
    owner: root
    group: root
    mode: '0640'
    backup: yes

- name: 重启 auditd 以应用新规则
  systemd:
    name: auditd
    state: restarted
```

### 3. **新增 `roles/security/files/audit.rules` 配置文件**

在 `roles/security/files/` 目录下创建 `audit.rules` 文件，并添加以下内容：

```bash
# roles/security/files/audit.rules

# 监控所有用户的登录和登出
-w /var/log/auth.log -p wa -k auth

# 监控/etc/passwd和/etc/shadow的更改
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes

# 监控关键系统目录
-w /etc/ -p wa -k etc_changes
-w /bin/ -p wa -k bin_changes
-w /sbin/ -p wa -k sbin_changes

# 监控网络配置
-w /etc/network/ -p wa -k network_changes

# 监控 SSH 配置
-w /etc/ssh/sshd_config -p wa -k ssh_changes
```

### 4. **修改 `roles/security/tasks/main.yml` 以包含 auditd 和 sudo 限制**

```yaml
# roles/security/tasks/main.yml

---
- name: 备份原始 SSH 配置
  copy:
    src: /etc/ssh/sshd_config
    dest: /etc/ssh/sshd_config.bak
    remote_src: yes
    backup: yes

- name: 配置 SSH 以增强安全性
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: "^{{ item.regexp }}"
    line: "{{ item.line }}"
    state: present
  loop:
    - { regexp: 'PermitRootLogin', line: 'PermitRootLogin no' }
    - { regexp: 'PasswordAuthentication', line: 'PasswordAuthentication no' }
    - { regexp: '^#?Port', line: "Port {{ ssh_port }}" }
    - { regexp: 'AllowUsers', line: "AllowUsers {{ allowed_users | join(' ') }}" }

- name: 重启 SSH 服务以应用更改
  systemd:
    name: ssh
    state: restarted
    enabled: yes

- name: 创建新用户（如果不存在）
  user:
    name: "{{ ansible_user }}"
    shell: /bin/bash
    create_home: yes
    state: present

- name: 设置用户密码（可选，推荐使用 SSH key）
  # 取消注释以下任务并设置密码
  # user:
  #   name: "{{ ansible_user }}"
  #   password: "{{ 'your_password' | password_hash('sha512') }}"
  #   update_password: always

# 添加以下内容以安装并配置 auditd
- import_tasks: auditd.yml

# 限制 sudo 权限
- name: 安装 sudo 包（如果未安装）
  apt:
    name: sudo
    state: present
    update_cache: yes

- name: 限制 sudo 访问权限
  copy:
    src: sudoers
    dest: /etc/sudoers.d/limited_sudo
    owner: root
    group: root
    mode: '0440'
    backup: yes

- name: 确保 sudoers 文件权限正确
  file:
    path: /etc/sudoers.d/limited_sudo
    owner: root
    group: root
    mode: '0440'
```

### 5. **新增 `roles/security/files/sudoers` 配置文件**

在 `roles/security/files/` 目录下创建 `sudoers` 文件，并添加以下内容以限制 sudo 权限，仅允许特定用户使用 sudo：

```bash
# roles/security/files/sudoers

# 限制 sudo 权限，仅允许指定用户组或用户使用 sudo
%sudo   ALL=(ALL:ALL) ALL
# 仅允许特定用户使用 sudo，例如 your_username
your_username ALL=(ALL) NOPASSWD:ALL
```

**注意：**
- 将 `your_username` 替换为实际需要具有 sudo 权限的用户名。
- 确保 `sudoers` 文件的语法正确，避免锁定 sudo 访问。

### 6. **新增自动安全更新配置**

在 `roles/common/tasks/main.yml` 中新增自动安全更新的配置任务：

```yaml
# roles/common/tasks/main.yml

---
# 之前的任务...

- name: 安装自动安全更新包
  apt:
    name: unattended-upgrades
    state: present
    update_cache: yes

- name: 配置自动安全更新
  copy:
    src: 50unattended-upgrades
    dest: /etc/apt/apt.conf.d/50unattended-upgrades
    owner: root
    group: root
    mode: '0644'
    backup: yes

- name: 启用自动安全更新
  copy:
    src: 20auto-upgrades
    dest: /etc/apt/apt.conf.d/20auto-upgrades
    owner: root
    group: root
    mode: '0644'
    backup: yes
```

### 7. **新增自动安全更新配置文件**

在 `roles/common/files/` 目录下创建以下两个文件：

#### a. `50unattended-upgrades`

```json
// roles/common/files/50unattended-upgrades

Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
    // "${distro_id}:${distro_codename}-updates";
    // "${distro_id}:${distro_codename}-proposed";
    // "${distro_id}:${distro_codename}-backports";
};

// 自动删除不需要的软件包
Unattended-Upgrade::Remove-Unused-Dependencies "true";

// 自动重启服务，如果需要的话
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "02:00";
```

#### b. `20auto-upgrades`

```json
// roles/common/files/20auto-upgrades

APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

### 8. **更新 `roles/security/vars/main.yml` （如果需要变量化部分配置）**

如果希望将某些配置参数化，可以在 `roles/security/vars/main.yml` 中添加相关变量。例如，设置自动重启时间：

```yaml
# roles/security/vars/main.yml

---
ssh_port: 22
allowed_users:
  - your_username

# 自动重启时间
auto_reboot_time: "02:00"
```

然后，在 `roles/common/tasks/main.yml` 中引用该变量：

```yaml
# roles/common/tasks/main.yml

---
# 之前的任务...

- name: 配置自动安全更新
  template:
    src: 50unattended-upgrades.j2
    dest: /etc/apt/apt.conf.d/50unattended-upgrades
    owner: root
    group: root
    mode: '0644'
    backup: yes

- name: 启用自动安全更新
  template:
    src: 20auto-upgrades.j2
    dest: /etc/apt/apt.conf.d/20auto-upgrades
    owner: root
    group: root
    mode: '0644'
    backup: yes
```

并将 `roles/common/files/50unattended-upgrades` 和 `20auto-upgrades` 文件改为模板文件 `50unattended-upgrades.j2` 和 `20auto-upgrades.j2`，以便使用变量：

#### a. `50unattended-upgrades.j2`

```jinja
# roles/common/templates/50unattended-upgrades.j2

Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
    // "${distro_id}:${distro_codename}-updates";
    // "${distro_id}:${distro_codename}-proposed";
    // "${distro_id}:${distro_codename}-backports";
};

// 自动删除不需要的软件包
Unattended-Upgrade::Remove-Unused-Dependencies "true";

// 自动重启服务，如果需要的话
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "{{ auto_reboot_time }}";
```

#### b. `20auto-upgrades.j2`

```jinja
# roles/common/templates/20auto-upgrades.j2

APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

### 9. **更新 `playbook.yml` 以确保新任务被包含**

确保在 `roles/security/tasks/main.yml` 中导入 `auditd.yml`，以及其他新增的任务已经被正确包含。如前所示，`auditd.yml` 已通过 `import_tasks` 引入。

## 使用步骤

### 1. **更新目录结构**

确保你的 Ansible 项目目录结构如下，并包含所有新增的文件：

```
ubuntu_hardening/
├── inventory
├── playbook.yml
├── roles/
│   ├── common/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── files/
│   │   │   ├── 50unattended-upgrades
│   │   │   └── 20auto-upgrades
│   │   ├── templates/
│   │   │   ├── 50unattended-upgrades.j2
│   │   │   └── 20auto-upgrades.j2
│   │   └── vars/
│   │       └── main.yml
│   ├── docker/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── vars/
│   │       └── main.yml
│   └── security/
│       ├── tasks/
│       │   ├── main.yml
│       │   └── auditd.yml
│       ├── files/
│       │   ├── audit.rules
│       │   └── sudoers
│       └── vars/
│           └── main.yml
```

### 2. **确保所有文件内容正确**

根据上述内容，创建或更新以下文件：

- **roles/common/tasks/main.yml**：添加 AppArmor 配置和自动安全更新任务。
- **roles/common/files/50unattended-upgrades** 和 **20auto-upgrades**：添加自动安全更新配置文件。
- **roles/common/templates/50unattended-upgrades.j2** 和 **20auto-upgrades.j2**：如果使用模板化配置。
- **roles/security/tasks/auditd.yml**：安装和配置 auditd。
- **roles/security/files/audit.rules**：配置 auditd 规则。
- **roles/security/files/sudoers**：限制 sudo 权限。

### 3. **更新变量**

在 `roles/security/vars/main.yml` 中，确保所有变量正确配置。例如：

```yaml
# roles/security/vars/main.yml

---
ssh_port: 22
allowed_users:
  - your_username

# 自动重启时间
auto_reboot_time: "02:00"
```

将 `your_username` 替换为实际的用户名。

### 4. **检查并验证 Playbook**

在运行 Playbook 之前，建议使用 `ansible-playbook --syntax-check` 进行语法检查：

```bash
ansible-playbook -i inventory playbook.yml --syntax-check
```

### 5. **运行 Playbook**

执行以下命令运行 Playbook：

```bash
ansible-playbook -i inventory playbook.yml
```

### 6. **验证增强的安全配置**

- **AppArmor**：
  ```bash
  sudo aa-status
  ```
  确保 AppArmor 正在运行，并且配置为强制模式。

- **auditd**：
  ```bash
  sudo systemctl status auditd
  sudo auditctl -l
  ```
  检查 auditd 是否正在运行，并应用了正确的审计规则。

- **sudo 权限**：
  ```bash
  sudo -l
  ```
  确保只有指定的用户具有 sudo 权限，并且权限受限。

- **自动安全更新**：
  ```bash
  cat /etc/apt/apt.conf.d/50unattended-upgrades
  cat /etc/apt/apt.conf.d/20auto-upgrades
  ```
  检查自动更新配置是否正确。

## 完整的新增和修改代码

### `roles/common/tasks/main.yml`

```yaml
# roles/common/tasks/main.yml

---
- name: 更新 APT 包缓存
  apt:
    update_cache: yes
    cache_valid_time: 600

- name: 升级所有包到最新版本
  apt:
    upgrade: dist
    autoremove: yes
    autoclean: yes

- name: 安装常用软件包
  apt:
    name: "{{ common_packages }}"
    state: present
    update_cache: yes

- name: 安装 AppArmor
  apt:
    name: apparmor
    state: present
    update_cache: yes

- name: 启用并启动 AppArmor 服务
  systemd:
    name: apparmor
    enabled: yes
    state: started

- name: 设置 AppArmor 为强制模式
  command: aa-enforce /etc/apparmor.d/*
  args:
    warn: false

- name: 安装自动安全更新包
  apt:
    name: unattended-upgrades
    state: present
    update_cache: yes

- name: 配置自动安全更新
  template:
    src: 50unattended-upgrades.j2
    dest: /etc/apt/apt.conf.d/50unattended-upgrades
    owner: root
    group: root
    mode: '0644'
    backup: yes

- name: 启用自动安全更新
  template:
    src: 20auto-upgrades.j2
    dest: /etc/apt/apt.conf.d/20auto-upgrades
    owner: root
    group: root
    mode: '0644'
    backup: yes

- name: 配置 UFW 默认策略
  ufw:
    default: "{{ item }}"
  loop:
    - deny incoming
    - allow outgoing

- name: 允许 SSH 通过 UFW
  ufw:
    rule: allow
    name: OpenSSH

- name: 启用 UFW
  ufw:
    state: enabled
    logging: 'on'

- name: 启用并启动 Fail2Ban 服务
  systemd:
    name: fail2ban
    enabled: yes
    state: started
```

### `roles/common/templates/50unattended-upgrades.j2`

```jinja
# roles/common/templates/50unattended-upgrades.j2

Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
    // "${distro_id}:${distro_codename}-updates";
    // "${distro_id}:${distro_codename}-proposed";
    // "${distro_id}:${distro_codename}-backports";
};

// 自动删除不需要的软件包
Unattended-Upgrade::Remove-Unused-Dependencies "true";

// 自动重启服务，如果需要的话
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "{{ auto_reboot_time }}";
```

### `roles/common/templates/20auto-upgrades.j2`

```jinja
# roles/common/templates/20auto-upgrades.j2

APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

### `roles/security/tasks/main.yml`

```yaml
# roles/security/tasks/main.yml

---
- name: 备份原始 SSH 配置
  copy:
    src: /etc/ssh/sshd_config
    dest: /etc/ssh/sshd_config.bak
    remote_src: yes
    backup: yes

- name: 配置 SSH 以增强安全性
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: "^{{ item.regexp }}"
    line: "{{ item.line }}"
    state: present
  loop:
    - { regexp: 'PermitRootLogin', line: 'PermitRootLogin no' }
    - { regexp: 'PasswordAuthentication', line: 'PasswordAuthentication no' }
    - { regexp: '^#?Port', line: "Port {{ ssh_port }}" }
    - { regexp: 'AllowUsers', line: "AllowUsers {{ allowed_users | join(' ') }}" }

- name: 重启 SSH 服务以应用更改
  systemd:
    name: ssh
    state: restarted
    enabled: yes

- name: 创建新用户（如果不存在）
  user:
    name: "{{ ansible_user }}"
    shell: /bin/bash
    create_home: yes
    state: present

- name: 设置用户密码（可选，推荐使用 SSH key）
  # 取消注释以下任务并设置密码
  # user:
  #   name: "{{ ansible_user }}"
  #   password: "{{ 'your_password' | password_hash('sha512') }}"
  #   update_password: always

# 添加以下内容以安装并配置 auditd
- import_tasks: auditd.yml

# 限制 sudo 权限
- name: 安装 sudo 包（如果未安装）
  apt:
    name: sudo
    state: present
    update_cache: yes

- name: 限制 sudo 访问权限
  copy:
    src: sudoers
    dest: /etc/sudoers.d/limited_sudo
    owner: root
    group: root
    mode: '0440'
    backup: yes

- name: 确保 sudoers 文件权限正确
  file:
    path: /etc/sudoers.d/limited_sudo
    owner: root
    group: root
    mode: '0440'
```

### `roles/security/tasks/auditd.yml`

```yaml
# roles/security/tasks/auditd.yml

---
- name: 安装 auditd
  apt:
    name: auditd
    state: present
    update_cache: yes

- name: 启用并启动 auditd 服务
  systemd:
    name: auditd
    enabled: yes
    state: started

- name: 配置 auditd 规则
  copy:
    src: audit.rules
    dest: /etc/audit/rules.d/audit.rules
    owner: root
    group: root
    mode: '0640'
    backup: yes

- name: 重启 auditd 以应用新规则
  systemd:
    name: auditd
    state: restarted
```

### `roles/security/files/audit.rules`

```bash
# roles/security/files/audit.rules

# 监控所有用户的登录和登出
-w /var/log/auth.log -p wa -k auth

# 监控/etc/passwd和/etc/shadow的更改
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes

# 监控关键系统目录
-w /etc/ -p wa -k etc_changes
-w /bin/ -p wa -k bin_changes
-w /sbin/ -p wa -k sbin_changes

# 监控网络配置
-w /etc/network/ -p wa -k network_changes

# 监控 SSH 配置
-w /etc/ssh/sshd_config -p wa -k ssh_changes
```

### `roles/security/files/sudoers`

```bash
# roles/security/files/sudoers

# 限制 sudo 权限，仅允许指定用户组或用户使用 sudo
%sudo   ALL=(ALL:ALL) ALL
# 仅允许特定用户使用 sudo，例如 your_username
your_username ALL=(ALL) NOPASSWD:ALL
```

**注意：**
- 将 `your_username` 替换为实际需要具有 sudo 权限的用户名。
- 确保 `sudoers` 文件的语法正确，避免锁定 sudo 访问。

### `roles/security/vars/main.yml`

```yaml
# roles/security/vars/main.yml

---
ssh_port: 22
allowed_users:
  - your_username

# 自动重启时间
auto_reboot_time: "02:00"
```

将 `your_username` 替换为实际的用户名。

## 运行和验证

### 1. **检查 Ansible Playbook 语法**

```bash
ansible-playbook -i inventory playbook.yml --syntax-check
```

### 2. **运行 Playbook**

```bash
ansible-playbook -i inventory playbook.yml
```

### 3. **验证增强的安全配置**

- **AppArmor**：
  ```bash
  sudo aa-status
  ```
  确保 AppArmor 正在运行，并且配置为强制模式。

- **auditd**：
  ```bash
  sudo systemctl status auditd
  sudo auditctl -l
  ```
  检查 auditd 是否正在运行，并应用了正确的审计规则。

- **sudo 权限**：
  ```bash
  sudo -l
  ```
  确保只有指定的用户具有 sudo 权限，并且权限受限。

- **自动安全更新**：
  ```bash
  cat /etc/apt/apt.conf.d/50unattended-upgrades
  cat /etc/apt/apt.conf.d/20auto-upgrades
  ```
  检查自动更新配置是否正确。

通过以上修改和新增的代码，你的 Ubuntu 系统将具备更强的安全性和自动化管理能力。如有任何问题或需要进一步的帮助，请随时提问！

#  使用命令行工具格式化YAML

如果你喜欢使用命令行，可以使用 `yamllint` 工具来检查和格式化 YAML 文件。

#### 安装 `yamllint`：

```sh
sudo apt update
sudo apt install yamllint -y
```

#### 使用 `yamllint` 检查文件：

```sh
yamllint /path/to/your/file.yml
```

#### 示例：

```sh
yamllint roles/common/tasks/main.yml
```

`yamllint` 会指出文件中的语法错误和格式问题，帮助你快速定位并修复。

### 注意事项：

1. **缩进**：每个层级使用 **2 个空格**，确保 `apt:` 和其参数与 `- name:` 同一级别。
2. **YAML 文件开头**：建议以 `---` 开始，表示 YAML 文件的开始。
3. **模块参数缩进**：模块（如 `apt:`、`systemd:`）的参数需要比模块名缩进 **2 个空格**。

## 验证和运行 Playbook

### 1. 检查 YAML 语法

在运行 Playbook 前，确保 YAML 文件语法正确：

```sh
yamllint roles/common/tasks/main.yml
```

确保输出没有错误。如果有错误，根据提示进行修正。

### 2. 运行 Playbook

```sh
ansible-playbook -i inventory playbook.yml
```

如果仍然遇到错误，请再次检查 `main.yml` 的缩进和格式，确保与示例一致。

### 3、推荐的最佳实践

1. **使用一致的缩进**：始终使用 **2 个空格**，避免使用 Tab。
2. **保持文件清晰**：合理分配任务，使用注释解释复杂步骤。
3. **使用模板**：对于配置文件，使用 Jinja2 模板（`.j2` 文件）来动态生成配置。
4. **版本控制**：将 Ansible 项目放入 Git 仓库，便于追踪更改和协作。
5. **分角色管理**：将不同功能分配到不同的角色（如 `common`、`docker`、`security`），提高可维护性。

### 4、进一步的帮助资源

- **Ansible 官方文档**：https://docs.ansible.com/ansible/latest/index.html
- **YAML 官方文档**：https://yaml.org/spec/1.2/spec.html
- **YAML Lint 在线工具**：https://www.yamllint.com/

如果按照上述步骤操作，应该能快速格式化并修复你的 YAML 文件，顺利运行 Ansible Playbook。如果仍有问题，请提供相关的 YAML 文件内容，以便进一步诊断和帮助。

# 在运行 Playbook 时提供 `sudo` 密码 

如果不想配置无密码 `sudo`，可以在运行 Playbook 时手动输入 `sudo` 密码。

```sh
ansible-playbook -i inventory playbook.yml --ask-become-pass
```

运行此命令后，系统会提示你输入 `sudo` 密码。

