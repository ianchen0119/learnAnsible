# Inventory

本文目標：
- 熟悉 Inventory
- 認識 module 與 ansible.cfg

## 進入正題

Inventory 可以幫助使用者藉由單一檔案（可以是 `.yaml` 或是 `.ini` 格式的檔案）管理多個分散式的節點：
- 使用 Inventory 可以有效降低運行命令時需要填入的命令列選項
- 提供群組的概念，讓我們可以一次操控多台主機

### 如果沒有 Inventory...

如果沒有 Inventory，我們在輸入命令時必須要帶入大量的選項，例如：
- HostName
- HostIP
- RemoteNodePassword

### 撰寫 Inventory
前面有提到，Ansible 接受副檔名為 `.ini` 或是 `.yaml` 的文件作為 inventory，本篇文章會使用 `yaml` 來示範如何撰寫 inventory：
```yaml
vmGroup:
  hosts:
    vm01:
      ansible_host: <HOST_IP>
      ansible_port: 22
      ansible_user: <USER>
      ansible_password: <PASSWORD>
      ansible_private_key_file: <FILE>
```
除了上面範例中列出的選項，Ansible 還為 Inventory 提供了多樣的可選參數，下方會以 `OptionName (default value)` 列出其它選項：
- `ansible_connection（smart）`
- `ansible_shell_type（sh）`
- `ansible_python_intepreter（/usr/bin/python）`
- `ansible_*_intepreter（無預設值）`

### 列出 Inventory file

使用以下指令可以列出 inventory 的資訊：
```
$ ansible-inventory -i inventory.yaml --list
```

### 使用 Ansible 執行模組

命令格式如下：
```
$ ansible <GROUP_NAME> -m <MODULE_NAME> -i <INVENTORY_FILE>
```
- `<GROUP_NAME>`：指定群組名稱，請設定已寫入 Inventory file 的群組。
- `<MODULE_NAME>`：Ansible 社群提供多樣化的模組供開發者使用，Ansible 不像是撰寫 Shell Script 一樣讓目標主機執行一行行命令，而是使用各式各樣的模組拼裝出自己需要的功能。
- `<INVENTORY_FILE>`：指定使用哪一個 Inventory。

以下方的命令來看：
```
$ ansible vmGroup -m command -a "uname -r" -i inventory.yaml
```
範例中使用到 `command` 這個預設模組，該模組可以使用 `-a` 選項接受期望命令，透過 `command` 模組，我們可以拼量的對一個群組中的所有主機下達同樣的命令：
```
ianchen0119@ubuntu:~/learnAnsible/private$ ansible vmGroup -m command -a "uname -r" -i inventory.yaml
vm01 | CHANGED | rc=0 >>
5.4.0-131-generic
```
下達命令後，我們可以看到 Ansible 告訴我們 vm-1 使用的 Linux kernel 版本號（此訊息為 `uname -r` 回傳的預期結果）。

### 使用 `ansible.cfg`
在不使用 `ansible.cfg` 的情況下，我們在不同的環境（作業系統、Ansible 版本）執行一樣的命令與腳本都有可能出現不同的結果，使用者可以透過 `ansible.cfg` 配置個人的 Ansible 組態設定：
```
[defaults]
host_key_checking = False
deprecation_warnings = False
```
以上面的例子來說，Ansible 會關閉 deprecated feature 的警告訊息，以及連線至遠端主機時跳出的 key checking。

此外，`ansible.cfg` 還能夠幫助我們複寫 Inventory 的選項預設值，`ansible_port:remote_port` 表示 `ansible.cfg` 的 `remote_port` 選項可以替換 Inventory 中 `ansible_port` 選項的預設值：
- `ansible_port:remote_port`
- `ansible_user:remote_user`
- `ansible_private_key_file:private_key_file`
- `ansible_shell_type:executable`

### 分組
Inventory 提供分組的功能，讓我們能夠套用設定至指定的群組中，而不是直接套用到 Inventory 中描述的所有主機。讓我們稍微改寫一下先前的 Inventory 範例腳本：
```yaml
vmGroup:
  children:
    webservers:
      hosts:
        server01:
          ansible_host: <HOST_IP_1>
          ansible_port: 22
          ansible_user: <USER>
          ansible_password: <PASSWORD>
          ansible_private_key_file: <FILE>
    dbs:
      hosts:
        db01:
          ansible_host: <HOST_IP_2>
          ansible_port: 22
          ansible_user: <USER>
          ansible_password: <PASSWORD>
          ansible_private_key_file: <FILE>
```
看到上方的範例，`vmGroup` 有兩個子群組 `webservers` 以及 `dbs`，這兩個子群組中分別又管理了一台主機（當然，真實的情況會在複雜一些，每個群組都可能管理著數個主機）。
我們可以依序執行下列命令，觀察不同命令影響著哪些主機：
```
# 影響 server01 與 db01
$ ansible vmGroup -m ping -i inventory.yaml
# 影響 server01
$ ansible webservers -m ping -i inventory.yaml
# 影響 db01
$ ansible dbs -m ping -i inventory.yaml
```

### 使用 `variables` 精簡 Inventory

觀察上方的 Inventory 範例不難發現 server01 與 db01 之間的重複欄位過多，為了精簡 Inventory file，我們可以將多數重複的資訊設定為變數：
```yaml
all:
  vars:
    ansible_user: <USER_NAME>
    ansible_password: <PASSWORD>
vmGroup:
  children:
    webservers:
      hosts:
        vm01:
          ansible_host: <HOST_IP_1>
          ansible_network_os: ubuntu
    dbs:
      hosts:
        vm02:
          ansible_host: <HOST_IP_2>
          ansible_network_os: ubuntu
```
參考上面的範例，我們將變數的作用域設置為 `all`，也就代表 `ansible_user` 與 `ansible_password` 這兩個變數可以作用在 `vmGroup`、`servers` 以及 `dbs` 群組。

> 補充：
> 為了驗證變數的作用域，也可以嘗試將 `all` 修改為 `servers` 或是 `dbs`，並且執行：
> ```
> $ ansible vmGroup -m ping -i inventory.yaml
> ```
> 觀察不同作用域下執行結果是否符合預期。

剛剛我們在範例中利用變數刪去了多餘的設定，如果我們不需要為群組中的主機設置別名，我們還可以再次精簡 Inventory file：
```yaml
all:
  vars:
    ansible_user: <USER_NAME>
    ansible_password: <PASSWORD>
vmGroup:
  children:
    webservers:
      hosts:
        <HOST_IP_1>:
    dbs:
      hosts:
        <HOST_IP_2>:
```
當然，如果配合 `ansible.cfg` 覆蓋預設值，我們的 Inventory 更可以精簡成：
```yaml
vmGroup:
  children:
    webservers:
      hosts:
        <HOST_IP_1>:
    dbs:
      hosts:
        <HOST_IP_2>:
```
而我們的 `ansible.cfg` 會增加以下兩行選項：
```
+ remote_user = <USER_NAME>
+ remote_password = <PASSWORD>
```

## 總結
本篇文章介紹了 Inventory 常見的用法，並且粗略的介紹了 `ansible.cfg` 與模組的概念，在之後的篇章中，我們會更深入的介紹這些概念，逐步的建構第一份 Ansible playbooks ：）