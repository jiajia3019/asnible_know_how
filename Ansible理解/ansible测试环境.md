当然，我很乐意指导您如何在 Rocky Linux 9.4 服务器上测试这些 Ansible 示例。以下是详细的步骤，帮助您从安装 Ansible 到运行 Playbook，验证每个示例的正确性。

---

### **步骤 1：安装 Ansible**

1. **更新系统软件包：**

   ```bash
   sudo dnf update -y
   ```

2. **安装 Ansible：**

   ```bash
   sudo dnf install -y ansible
   ```

3. **验证 Ansible 安装：**

   ```bash
   ansible --version
   ```

   确保安装的 Ansible 版本为 **2.9** 或更高，以兼容示例。

---

### **步骤 2：设置 Ansible 项目目录结构**

1. **创建项目目录：**

   ```bash
   mkdir -p ~/ansible_project
   cd ~/ansible_project
   ```

2. **创建必要的子目录：**

   ```bash
   mkdir -p roles/sample_role/{tasks,defaults,vars}
   mkdir -p group_vars
   mkdir -p host_vars
   mkdir -p inventory/group_vars
   mkdir -p inventory/host_vars
   mkdir -p vars
   mkdir -p tasks
   ```

---

### **步骤 3：创建清单文件（Inventory）**

1. **创建清单文件：**

   ```bash
   nano inventory/hosts
   ```

2. **添加以下内容到 `inventory/hosts`：**

   ```ini
   [webservers]
   web1 ansible_host=127.0.0.1 sample_inventory_host_var="This is an inventory host variable"

   [dbservers]
   db1 ansible_host=127.0.0.1

   [webservers:vars]
   sample_inventory_group_var="This is an inventory group variable"
   ```

   **注意：** 由于您在本地测试，`ansible_host` 使用 `127.0.0.1`。

---

### **步骤 4：配置 SSH 访问（可选）**

1. **确保 SSH 服务正在运行：**

   ```bash
   sudo systemctl start sshd
   sudo systemctl enable sshd
   ```

2. **为本地用户设置 SSH 密钥登录：**

   ```bash
   ssh-keygen -t rsa -b 2048
   ssh-copy-id your_user@127.0.0.1
   ```

   将 `your_user` 替换为您的实际用户名。

---

### **步骤 5：创建 Playbook**

1. **创建 Playbook 文件：**

   ```bash
   nano playbook.yml
   ```

2. **添加以下内容到 `playbook.yml`（以测试第一个示例为例）：**

   ```yaml
   - hosts: all
     vars:
       sample_play_var: "This is a play variable"
     tasks:
       - name: "Use play variable"
         debug:
           var: sample_play_var
   ```

---

### **步骤 6：运行 Playbook**

1. **执行 Playbook：**

   ```bash
   ansible-playbook -i inventory/hosts playbook.yml
   ```

2. **查看输出结果：**

   您应该看到 `debug` 任务输出 `sample_play_var` 的值。

---

### **步骤 7：测试其他示例**

下面将指导您如何测试更多示例。

---

#### **示例 2：Role Defaults**

1. **定义角色默认变量：**

   ```bash
   nano roles/sample_role/defaults/main.yml
   ```

   **内容：**

   ```yaml
   sample_default_var: "This is a role default variable"
   ```

2. **创建角色任务文件：**

   ```bash
   nano roles/sample_role/tasks/main.yml
   ```

   **内容：**

   ```yaml
   - name: "Use role default variable"
     debug:
       var: sample_default_var
   ```

3. **修改 Playbook 以使用角色：**

   更新 `playbook.yml`：

   ```yaml
   - hosts: all
     roles:
       - sample_role
   ```

4. **运行 Playbook：**

   ```bash
   ansible-playbook -i inventory/hosts playbook.yml
   ```

5. **验证输出：**

   `debug` 任务应显示 `sample_default_var` 的值。

---

#### **示例 3：Inventory Group Vars**

1. **创建 `group_vars` 文件：**

   ```bash
   nano inventory/group_vars/all.yml
   ```

   **内容：**

   ```yaml
   sample_inventory_group_all_var: "This is an inventory group_vars/all variable"
   ```

2. **更新 Playbook：**

   ```yaml
   - hosts: all
     tasks:
       - name: "Use inventory group_vars/all variable"
         debug:
           var: sample_inventory_group_all_var
   ```

3. **运行 Playbook 并验证输出。**

---

#### **示例 4：Play Vars Files**

1. **创建变量文件：**

   ```bash
   nano vars/sample_vars_file.yml
   ```

   **内容：**

   ```yaml
   sample_vars_file_var: "This is a variable from vars_files"
   ```

2. **更新 Playbook：**

   ```yaml
   - hosts: all
     vars_files:
       - vars/sample_vars_file.yml
     tasks:
       - name: "Use vars_files variable"
         debug:
           var: sample_vars_file_var
   ```

3. **运行 Playbook 并验证输出。**

---

#### **示例 5：Include Vars**

1. **创建包含的变量文件：**

   ```bash
   nano vars/sample_include_vars.yml
   ```

   **内容：**

   ```yaml
   sample_include_vars: "This is include_vars"
   ```

2. **更新 Playbook：**

   ```yaml
   - hosts: all
     tasks:
       - name: "Include vars sample"
         include_vars: vars/sample_include_vars.yml

       - name: "Use included variable"
         debug:
           var: sample_include_vars
   ```

3. **运行 Playbook 并验证输出。**

---

#### **示例 6：Set Fact / Registered Vars**

**使用 `set_fact`：**

1. **更新 Playbook：**

   ```yaml
   - hosts: all
     tasks:
       - name: "Set a fact variable"
         set_fact:
           sample_fact_var: "This is a fact variable"

       - name: "Use fact variable"
         debug:
           var: sample_fact_var
   ```

2. **运行 Playbook 并验证输出。**

**使用 Registered Vars：**

1. **更新 Playbook：**

   ```yaml
   - hosts: all
     tasks:
       - name: "Run a command"
         command: echo "This is output"
         register: sample_registered_var

       - name: "Use registered variable"
         debug:
           var: sample_registered_var.stdout
   ```

2. **运行 Playbook 并验证输出。**

---

#### **示例 7：Extra Vars**

1. **更新 Playbook：**

   ```yaml
   - hosts: all
     tasks:
       - name: "Use extra variable"
         debug:
           var: sample_extra_var
   ```

2. **使用 Extra Vars 运行 Playbook：**

   ```bash
   ansible-playbook -i inventory/hosts playbook.yml -e "sample_extra_var=This is an extra variable"
   ```

3. **验证输出：**

   `debug` 任务应显示 `sample_extra_var` 的值。

---

### **步骤 8：继续测试剩余的示例**

按照上述模式，继续为其他示例创建必要的文件，更新 Playbook，并运行以验证输出。确保：

- **正确设置变量的定义和使用位置。**
- **在 Playbook 中指定正确的主机或组。**
- **变量名称和引用保持一致。**

---

### **步骤 9：检查目录结构**

您的项目目录结构应类似于：

```
ansible_project/
├── ansible.cfg
├── inventory/
│   ├── hosts
│   ├── group_vars/
│   │   ├── all.yml
│   │   └── webservers.yml
│   └── host_vars/
│       └── web1.yml
├── group_vars/
│   └── dbservers.yml
├── host_vars/
│   └── db1.yml
├── roles/
│   └── sample_role/
│       ├── defaults/
│       │   └── main.yml
│       ├── tasks/
│       │   └── main.yml
│       └── vars/
│           └── main.yml
├── vars/
│   └── sample_include_vars.yml
├── tasks/
│   └── sample_task.yml
└── playbook.yml
```

---

### **步骤 10：可选配置 Ansible 配置文件**

1. **创建 `ansible.cfg` 文件：**

   ```bash
   nano ansible.cfg
   ```

2. **添加以下内容：**

   ```ini
   [defaults]
   inventory = ./inventory/hosts
   host_key_checking = False
   ```

   **解释：**

   - 设置默认的清单文件路径。
   - 禁用主机密钥检查，避免首次连接提示。

---

### **额外提示**

- **使用 `--syntax-check` 检查 Playbook 语法：**

  ```bash
  ansible-playbook playbook.yml --syntax-check
  ```

- **在运行时增加详细程度（可选）：**

  ```bash
  ansible-playbook playbook.yml -v  # -vvv 可获得更详细的输出
  ```

- **了解变量优先级：**

  您可以在不同层次定义相同名称的变量，观察哪个值生效，以理解 Ansible 的变量优先级规则。

---

### **参考变量优先级顺序**

在 Ansible 中，变量的优先级（从低到高）为：

1. Role defaults
2. Inventory vars
3. Playbook vars
4. Task vars
5. Extra vars (使用 `-e` 指定，优先级最高)

---

### **常见问题排查**

- **变量未定义错误：**

  确保变量名称拼写正确，变量文件路径正确。

- **权限问题：**

  确保运行 Ansible 的用户有足够的权限访问必要的文件和目录。

- **SSH 连接失败：**

  检查 SSH 服务是否运行，主机名和 IP 地址是否正确，SSH 密钥是否配置正确。

---

### **结论**

通过上述步骤，您可以在 Rocky Linux 9.4 服务器上成功测试提供的 Ansible 示例。如果在测试过程中遇到任何问题，欢迎随时提问，我将竭诚为您解答。