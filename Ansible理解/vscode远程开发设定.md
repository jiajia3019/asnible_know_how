要在本地的 Visual Studio Code（VSCode）中编辑位于远程 Rocky Linux 9.4 服务器上的代码，您可以使用 VSCode 的 **Remote - SSH** 扩展。这使您能够通过 SSH 连接到远程服务器，并在本地编辑远程文件，就像它们在本地一样。以下是详细的配置步骤：

---

### **步骤 1：在本地计算机上安装 VSCode**

如果您还没有安装 VSCode，请从 [Visual Studio Code 官方网站](https://code.visualstudio.com/)下载并安装适用于您操作系统的版本。

---

### **步骤 2：安装 Remote - SSH 扩展**

1. **打开 VSCode**。

2. **安装 Remote - SSH 扩展：**

   - 点击左侧活动栏中的**扩展**图标（或按 `Ctrl+Shift+X`）。
   - 在搜索栏中输入 `Remote - SSH`。
   - 找到名为 **Remote - SSH** 的扩展，由 Microsoft 发布。
   - 点击 **Install** 按钮进行安装。

   ![Install Remote - SSH Extension](https://code.visualstudio.com/assets/docs/remote/ssh/remote-install-extension.png)

---

### **步骤 3：配置 SSH 连接**

#### **方法一：使用现有的 SSH 配置**

如果您已经在本地计算机上配置了 SSH，并且可以通过 SSH 终端连接到服务器，可以直接使用现有的 SSH 配置。

1. **打开 SSH 配置文件：**

   - 在 VSCode 中按 `F1`（或 `Ctrl+Shift+P`）打开命令面板。
   - 输入 `Remote-SSH: Open SSH Configuration File...`，然后按回车。
   - 选择您的 SSH 配置文件，一般位于 `~/.ssh/config`。

2. **添加服务器配置：**

   在 SSH 配置文件中添加以下内容：

   ```ssh
   Host my_remote_server
       HostName your_server_ip_or_hostname
       User your_username
       Port 22  # 如果不是默认端口，请更改为实际端口
   ```

   **示例：**

   ```ssh
   Host rocky_server
       HostName 192.168.1.100
       User root
       Port 22
   ```

   **注意：** 为了安全，不建议使用 `root` 用户，最好创建一个具有适当权限的普通用户。

#### **方法二：直接在 VSCode 中添加新主机**

1. **打开命令面板并添加新主机：**

   - 按 `F1`（或 `Ctrl+Shift+P`）打开命令面板。
   - 输入 `Remote-SSH: Connect to Host...`，然后选择 **Add New SSH Host...**。

2. **输入 SSH 连接命令：**

   在提示中输入 SSH 连接命令，例如：

   ```bash
   ssh your_username@your_server_ip_or_hostname
   ```

   **示例：**

   ```bash
   ssh user@192.168.1.100
   ```

3. **选择保存 SSH 配置的位置：**

   选择保存到 `~/.ssh/config`。

---

### **步骤 4：设置 SSH 密钥（可选但推荐）**

为了更安全和方便，建议使用 SSH 密钥认证，而不是密码。

1. **在本地生成 SSH 密钥：**

   打开终端，输入：

   ```bash
   ssh-keygen -t rsa -b 4096
   ```

   按提示操作，通常可以直接按回车使用默认设置。

2. **将公钥复制到服务器：**

   ```bash
   ssh-copy-id your_username@your_server_ip_or_hostname
   ```

   **示例：**

   ```bash
   ssh-copy-id user@192.168.1.100
   ```

   **注意：** 您需要输入远程服务器用户的密码。

3. **测试 SSH 连接：**

   ```bash
   ssh your_username@your_server_ip_or_hostname
   ```

   如果能够无密码登录，则设置成功。

---

### **步骤 5：使用 VSCode 连接到远程服务器**

1. **打开命令面板并连接到主机：**

   - 按 `F1`（或 `Ctrl+Shift+P`）打开命令面板。
   - 输入 `Remote-SSH: Connect to Host...`，然后选择您在之前步骤中配置的主机（如 `my_remote_server` 或 `rocky_server`）。

2. **选择窗口打开方式：**

   - 选择 **Connect in Current Window** 或 **Connect in New Window**，根据您的偏好。

3. **首次连接配置（仅需一次）：**

   - VSCode 会在远程服务器上安装所需的服务器组件。
   - 如果出现提示，可能需要输入远程服务器的密码或接受主机密钥。

4. **连接成功后，左下角会显示远程连接的名称。**

---

### **步骤 6：打开远程目录并编辑文件**

1. **打开文件夹：**

   - 在 VSCode 中，点击菜单 **File > Open Folder**，或者在命令面板中输入 `Remote-SSH: Open Folder`。
   - 浏览远程服务器的文件系统，选择您要编辑的项目目录。

2. **编辑代码：**

   - 您现在可以像在本地一样编辑远程服务器上的文件。
   - 所有的 VSCode 功能，如语法高亮、代码补全、扩展等，都可以正常工作。

---

### **步骤 7：安装必要的扩展（可选）**

- 当您在远程环境中需要特定的 VSCode 扩展（例如 Python、Ansible 扩展）时，可以在远程服务器上安装这些扩展。

1. **在 VSCode 左侧点击扩展图标。**

2. **搜索您需要的扩展，例如 `Ansible`。**

3. **在扩展的安装按钮旁边，点击下拉箭头，选择 **Install on SSH: your_server_name**。

   ![Install Extension on Remote](https://code.visualstudio.com/assets/docs/remote/ssh/remote-install-extension-target.png)

---

### **步骤 8：设置文件同步（可选）**

如果您的项目需要在本地和远程之间同步文件，可以考虑以下方法：

- **使用版本控制系统（如 Git）：** 在本地编辑代码，通过 Git 推送到远程仓库，然后在服务器上拉取更新。

- **使用文件同步工具：** 如 rsync、Unison，但在使用 VSCode 的 Remote - SSH 扩展时，这通常不是必要的。

---

### **注意事项**

- **安全性：** 确保您的 SSH 密钥安全，不要与他人共享。定期更新密码和密钥。

- **网络连接：** 由于编辑的是远程文件，网络连接质量会影响编辑体验。确保有稳定的网络连接。

- **权限问题：** 编辑文件时，如果遇到权限不足的问题，可能需要在服务器上调整文件权限或以具有适当权限的用户连接。

---

### **常见问题排查**

1. **无法连接到远程服务器：**

   - 检查 SSH 配置是否正确，主机名、用户名、端口是否正确。
   - 确保防火墙允许 SSH 连接。
   - 测试能否在终端中使用 SSH 连接到服务器。

2. **提示主机密钥验证失败：**

   - 可能是由于主机密钥发生变化。编辑 `~/.ssh/known_hosts` 文件，删除对应的旧条目，然后重新连接。

3. **远程扩展无法安装：**

   - 确保远程服务器有互联网访问权限，以下载扩展。
   - 检查磁盘空间是否足够。

4. **编辑时出现权限错误：**

   - 确认您有权限编辑目标文件和目录。
   - 如果需要以特权用户身份操作，考虑使用 `sudo`，但在 VSCode 中直接提升权限并不推荐。

---

### **总结**

通过以上步骤，您可以在本地的 VSCode 中无缝地编辑远程 Rocky Linux 服务器上的代码。这种方式结合了本地开发的便利性和远程部署的灵活性，适用于多种开发场景。

如果您在配置过程中遇到任何问题，或者需要进一步的帮助，请随时告诉我！