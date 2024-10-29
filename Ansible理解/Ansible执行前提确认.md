在自动化配置和部署过程中，确保目标主机的基础环境满足执行 Ansible Playbook 的要求至关重要。这样可以减少错误发生的可能性，提升部署的可靠性和效率。专业的做法通常包括以下几个方面：

1. **定义前置任务（Pre-Tasks）**：
   - 在 Playbook 中使用 `pre_tasks` 来执行一系列前置检查，确保环境条件满足。
   
2. **使用 `assert` 模块进行条件验证**：
   - 利用 `assert` 模块验证关键变量、软件包是否已安装、服务是否运行等。
   
3. **检查目标主机的连通性和权限**：
   - 确保 Ansible 控制节点能够通过 SSH 无密码访问目标主机，并且具有足够的权限执行所需任务。
   
4. **验证所需软件和依赖是否存在**：
   - 检查 Ansible、Python 版本、必要的软件包是否已安装。
   
5. **利用 Ansible Facts 和自定义事实**：
   - 使用 Ansible Facts 收集目标主机的信息，基于这些信息进行环境验证。
   
6. **实现幂等性和错误处理**：
   - 确保前置检查任务是幂等的，并且在检查失败时能够优雅地中止 Playbook 执行。

以下是详细的专业建议和示例，帮助你在执行 Ansible Playbook 之前进行全面的前提条件检查。

## 一、使用 `pre_tasks` 进行前置检查

在 Ansible Playbook 中，`pre_tasks` 部分用于在执行主要任务之前运行一组任务。这些任务通常用于验证环境条件、安装必要的软件包、创建所需的目录等。

### 示例结构

```yaml
---
- name: 部署应用
  hosts: all
  become: yes

  pre_tasks:
    - name: 确保目标主机可以连接到互联网
      uri:
        url: https://www.google.com
        method: GET
        return_content: no
      register: internet_check
      failed_when: internet_check.status != 200
      changed_when: false

    - name: 检查必要的软件包是否已安装
      package:
        name: "{{ item }}"
        state: present
      loop:
        - git
        - curl
        - python3
      register: packages_check
      failed_when: packages_check.failed

    - name: 确保用户具有 sudo 权限
      assert:
        that:
          - ansible_user is defined
          - "'sudo' in ansible_user.shell"

  roles:
    - common
    - webserver
```

### 详细说明

1. **检查互联网连接**：
   - 使用 `uri` 模块尝试访问外部网站，验证目标主机是否能够连接到互联网。
   - 如果无法访问，将导致 Playbook 执行失败。

2. **检查必要的软件包**：
   - 使用 `package` 模块（跨平台支持），确保所需的软件包（如 `git`、`curl`、`python3`）已安装。
   - 如果某个软件包未安装，Ansible 会自动安装它。

3. **验证用户权限**：
   - 使用 `assert` 模块检查当前用户是否具备 sudo 权限，确保后续任务能够以提升权限执行。

## 二、使用 `assert` 模块进行条件验证

`assert` 模块用于验证某些条件是否为真，如果条件不满足，Playbook 将失败并输出指定的错误消息。

### 示例

```yaml
pre_tasks:
  - name: 验证操作系统类型
    assert:
      that:
        - ansible_facts['os_family'] == "Debian"
      fail_msg: "只支持 Debian 系统！"

  - name: 验证 Ansible 版本
    assert:
      that:
        - ansible_version.full is version('2.9', '>=')
      fail_msg: "需要 Ansible 版本 >= 2.9"

  - name: 确保磁盘空间足够
    assert:
      that:
        - ansible_facts['disk']['/']['available'] > 10485760  # 10MB
      fail_msg: "根分区磁盘空间不足！"
```

### 详细说明

1. **验证操作系统类型**：
   - 确保目标主机运行的是 Debian 系统。如果不是，则 Playbook 执行失败。

2. **验证 Ansible 版本**：
   - 确保控制节点运行的 Ansible 版本满足最低要求。

3. **检查磁盘空间**：
   - 使用 Ansible Facts 检查根分区的可用磁盘空间是否超过 10MB。如果不足，Playbook 执行失败。

## 三、检查目标主机的连通性和权限

确保 Ansible 控制节点能够通过 SSH 无密码访问目标主机，并且目标主机的用户具备足够的权限执行所需任务。

### 示例

```yaml
pre_tasks:
  - name: 检查 SSH 连接
    ping:
    register: ping_result
    failed_when: ping_result is failed

  - name: 确保用户可以使用 sudo
    command: sudo -n true
    register: sudo_check
    ignore_errors: yes
    failed_when: sudo_check.rc != 0
    changed_when: false
    notify:
      - 用户没有 sudo 权限

handlers:
  - name: 用户没有 sudo 权限
    fail:
      msg: "当前用户没有 sudo 权限，请配置 sudo 访问。"
```

### 详细说明

1. **检查 SSH 连接**：
   - 使用 `ping` 模块（实际执行 `ansible -m ping`），验证 Ansible 是否能够成功连接到目标主机。

2. **验证 sudo 权限**：
   - 使用 `command` 模块尝试执行 `sudo -n true` 命令，验证当前用户是否具备 sudo 权限。
   - 如果用户没有 sudo 权限，将触发一个处理程序，导致 Playbook 执行失败并输出错误消息。

## 四、检查目标主机上的特定配置或状态

在一些特定情况下，你可能需要验证目标主机上是否存在某些配置文件、服务是否正在运行等。

### 示例

```yaml
pre_tasks:
  - name: 检查是否已安装 Docker
    command: docker --version
    register: docker_check
    failed_when: docker_check.rc != 0
    changed_when: false
    ignore_errors: yes
    notify:
      - 安装 Docker

  - name: 检查防火墙状态
    command: ufw status
    register: firewall_status
    failed_when: firewall_status.rc != 0
    changed_when: false
```

### 详细说明

1. **检查 Docker 是否已安装**：
   - 尝试执行 `docker --version` 命令，如果 Docker 未安装，则触发处理程序安装 Docker。

2. **检查防火墙状态**：
   - 执行 `ufw status` 命令，验证防火墙是否已正确配置或开启。

## 五、使用自定义模块或脚本进行高级检查

对于复杂的前置条件检查，可以编写自定义 Ansible 模块或使用 `script` 模块执行自定义脚本。

### 示例

```yaml
pre_tasks:
  - name: 运行自定义前置检查脚本
    script: scripts/pre_check.sh
    register: pre_check_result
    failed_when: pre_check_result.rc != 0
    changed_when: false
```

### 自定义脚本示例 (`scripts/pre_check.sh`)

```bash
#!/bin/bash

# 检查是否安装了必需的软件包
REQUIRED_PACKAGES=("git" "curl" "python3")
for pkg in "${REQUIRED_PACKAGES[@]}"; do
  dpkg -l | grep -qw "$pkg" || { echo "$pkg 未安装！"; exit 1; }
done

# 检查用户权限
if [ "$(id -u)" -ne 0 ]; then
  echo "需要以 root 用户运行脚本。"
  exit 1
fi

exit 0
```

### 详细说明

1. **自定义脚本执行**：
   - 通过 `script` 模块运行一个 Bash 脚本，执行复杂的前置条件检查。
   - 如果脚本返回非零退出码，则 Playbook 执行失败。

## 六、使用 Ansible Roles 进行前置条件检查

将前置条件检查封装到一个独立的角色中，提高 Playbook 的模块化和可复用性。

### 示例目录结构

```
roles/
└── prerequisites/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── defaults/
    │   └── main.yml
    ├── vars/
    │   └── main.yml
    └── README.md
```

### `roles/prerequisites/tasks/main.yml`

```yaml
---
- name: 检查 SSH 连接
  ping:
  register: ping_result
  failed_when: ping_result is failed

- name: 确保用户可以使用 sudo
  command: sudo -n true
  register: sudo_check
  ignore_errors: yes
  failed_when: sudo_check.rc != 0
  changed_when: false
  notify:
    - 用户没有 sudo 权限

- name: 验证操作系统类型
  assert:
    that:
      - ansible_facts['os_family'] == "Debian"
    fail_msg: "只支持 Debian 系统！"

- name: 检查必要的软件包是否已安装
  package:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - curl
    - python3
  register: packages_check
  failed_when: packages_check.failed
```

### `roles/prerequisites/handlers/main.yml`

```yaml
---
- name: 用户没有 sudo 权限
  fail:
    msg: "当前用户没有 sudo 权限，请配置 sudo 访问。"
```

### `roles/prerequisites/README.md`

```markdown
# Prerequisites Role

## 描述

该角色用于在执行主要任务之前，验证目标主机的基础环境是否满足要求。包括检查 SSH 连接、用户权限、操作系统类型和必要的软件包是否已安装。

## 变量

- 无

## 使用示例

在 Playbook 中引用该角色作为 `pre_tasks`：

​```yaml
- name: 部署应用
  hosts: all
  become: yes

  pre_tasks:
    - name: 运行前置条件检查
      import_role:
        name: prerequisites

  roles:
    - common
    - webserver
```
```

### 详细说明

1. **角色封装**：
   - 将所有前置条件检查任务封装到 `prerequisites` 角色中，便于在不同 Playbook 中复用。

2. **使用 `import_role` 或 `include_role`**：
   - 在 Playbook 的 `pre_tasks` 部分引用 `prerequisites` 角色，确保在执行主要任务之前进行环境验证。

## 七、示例 Playbook 结合前置条件检查

以下是一个完整的 Playbook 示例，结合前置条件检查角色，部署一个应用。

### 示例 Playbook (`deploy_app.yml`)

​```yaml
---
- name: 部署应用
  hosts: all
  become: yes

  pre_tasks:
    - name: 运行前置条件检查
      import_role:
        name: prerequisites

  roles:
    - common
    - webserver
```

### 详细说明

1. **`pre_tasks` 部分**：
   - 引用 `prerequisites` 角色，执行所有前置条件检查任务。
   - 如果任何前置条件检查失败，Playbook 将中止执行。

2. **`roles` 部分**：
   - 在确保前置条件满足后，执行主要的 `common` 和 `webserver` 角色。

## 八、最佳实践总结

1. **模块化设计**：
   - 将前置条件检查封装到独立的角色中，提高 Playbook 的可复用性和可维护性。

2. **利用 Ansible Modules**：
   - 尽量使用 Ansible 提供的模块（如 `ping`、`assert`、`package`），避免使用 `shell` 或 `command`，以提高 Playbook 的幂等性和可读性。

3. **使用 `pre_tasks` 进行环境验证**：
   - 在 Playbook 的 `pre_tasks` 部分执行所有必要的前置条件检查，确保环境准备就绪。

4. **合理使用 `assert` 模块**：
   - 通过 `assert` 模块验证关键条件，确保 Playbook 执行的前提条件被满足。

5. **错误处理与通知**：
   - 使用 `fail` 模块或 `notify` 触发处理程序，在前置条件检查失败时提供明确的错误信息，帮助快速定位问题。

6. **文档化**：
   - 为前置条件检查角色编写详细的文档，说明其功能、变量和使用方法，提升团队协作效率。

7. **安全管理**：
   - 确保敏感信息（如 SSH 密钥、密码）通过 Ansible Vault 或 Jenkins 凭证进行安全管理，避免在 Playbook 中明文存储。

8. **测试与验证**：
   - 使用 Molecule 等工具编写和执行前置条件检查任务的测试用例，确保其在不同环境下的正确性和稳定性。

通过遵循上述专业建议和最佳实践，你可以在 Ansible Playbook 执行之前，全面验证目标主机的环境条件，确保部署过程顺利进行，减少潜在的错误和中断。

如果你有更多具体的问题或需要进一步的示例，欢迎随时提问！