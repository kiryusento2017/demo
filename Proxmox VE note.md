网络核心思想：存算分离

如有错误，请评论告知我，我非圣贤，孰能无过？

参考教程（红茶海）： [家庭数据中心教程汇总](https://www.yuque.com/hongcha-6c6o6/klr1op/xqao5sggi82efuet?singleDoc#)

# 零、前言
1.操作均使用红茶海的ubuntu24.04LTS模板[夸克网盘分享](https://pan.quark.cn/s/6799e4254f27?pwd=P6kc#/list/share)，您亦可以使用其他模板，但是请注意切换控制台。

2.若与我使用同一款操作台，请记得安装语言包。

3.建议平时备份几个pve的内核，以备不时之需，万一一个内核在设置中出了问题，无法启动，还可以有其他内核选择

```dockerfile
# 更新软件包列表
apt update
# 搜索可用的PVE内核
apt search proxmox-kernel | grep pve-signed
# 安装特定版本（如6.8.12-17）
apt install proxmox-kernel-6.8.12-17-pve-signed
# 安装特定版本的头文件
wget http://mirrors.nju.edu.cn/proxmox/debian/dists/bookworm/pve-no-subscription/binary-amd64/proxmox-headers-6.8.12-17-pve_6.8.12-13_amd64.deb
dpkg -i proxmox-headers-6.8.12-17-pve_6.8.12-17_amd64.deb
#安装头文件
ls -la /boot/*6.8.12-17-pve*
#检查内核完整度
proxmox-boot-tool kernel add 6.8.12-17-pve
#将内核手动添加到引导项
update-grub
#更新grub选项
proxmox-boot-tool refresh
#刷新引导配置
然后机器连上hdmi线开机重启试试看能不能选择这个内核
```

# 一、基础操作
1.进入promox ve后先禁用企业级  enterprise  然后进行换源，命令（选择南京大学源）

```bash
bash <(curl -sSL https://linuxmirrors.cn/main.sh)
#发现使用南京大学源的时候，存储库里面的东西会很全
```

2.此后删除订阅弹窗提示

```bash
sed -i_orig "s/data.status === 'Active'/true/g" /usr/share/pve-manager/js/pvemanagerlib.js
sed -i_orig "s/if (res === null || res === undefined || \!res || res/if(/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
sed -i_orig "s/.data.status.toLowerCase() !== 'active'/false/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy
```

3.绑定网卡，防止因为增减pcie设备而发生变动，原理：创建一个文件以此绑定，首先使用  ip a  查看你的网卡MAC地址，然后使用以下命令创建并填写文件

```bash
nano /usr/lib/systemd/network/50-custom-net0.link #创建一个文件用于绑定mac地址和物理网口
#复制以下文字进去，注意修改为自己的ip地址
[Match]
MACAddress=50:e9:71:01:e0:16

[Link]
Name=enp4s0

#额外代码
ip a
#查看网卡
lspci
#查看pcie设备
```

4.查看有没有开启ssh（默认应该是开启的）

```bash
systemctl status ssh
```

5.安装目录树（方便看，可以不安装）

```bash
apt update
apt install tree
```

6.我还会创建一下zfs池专门用于存储虚拟机

7.创建资源池

8.开启彩色标签

9.DNS地址千万千万不要放自己家的路由器ip

# 二、虚拟机immortalwrt的安装（旁路由）
1.虚拟机文件下载，我这里[单机下载immortalwrt-24.10.0-x86-64-generic-squashfs-combined-efi.img.gz](https://downloads.immortalwrt.org/releases/24.10.0/targets/x86/64/immortalwrt-24.10.0-x86-64-generic-squashfs-combined-efi.img.gz)  文件，下载完之后进行解压，接下来正常创建虚拟机我的id是901，但是选择`不使用介质`，可以选择没有磁盘。

2.上传完 immortalwrt-24.10.0-x86-64-generic-squashfs-combined-efi.img 文件之后，该文件被放置在了  /var/lib/vz/template/iso  中，我们把这个文件给发送到虚拟机里面，执行以下命令。

```bash
qm importdisk 901 /var/lib/vz/template/iso/immortalwrt-24.10.0-x86-64-generic-squashfs-combined-efi.img vm-zfs-M2 #vm-zfs-M2表示目标存储位置
```

3.调整启动顺序，将导入的新磁盘设置为首选启动项

4.启动完成后进入

```bash
vi /etc/config/network
```

调整ip地址，之后` reboot ` 重启

5.接下来首先配置好`接口`，然后配置`openclash`。

# 三、lxc容器的模板部署（可直接使用红茶海的模板）
 [家庭数据中心教程汇总](https://www.yuque.com/hongcha-6c6o6/klr1op/xqao5sggi82efuet?singleDoc#)

注意：第一：在使用lxc容器时有时候控制面板会无法跳出，经过研究，原因如下：

                    LXC容器控制台默认时tty，要求路径是/dev/ttyx

                   <font style="color:#DF2A3F;"> ubuntu24.04</font>版本下，<font style="color:#DF2A3F;">ttyx</font>确实位于<font style="color:#DF2A3F;">/dev/ttyx</font>

                    但是<font style="color:#DF2A3F;">ubuntu24.10</font>版本下，<font style="color:#DF2A3F;">tty</font>不再位于<font style="color:#DF2A3F;">/dev/ttyx</font>，所以此时请切换控制台为<font style="color:#DF2A3F;">shell</font>

           第二：在使用茶佬的 <font style="color:#DF2A3F;">ubuntu24.04</font>版本的lxc容器模板时，我遇到了docker容器无法运行的问题

                     原因是我没有开启lxc的嵌套，请开启嵌套和按键，具体原因问AI

[http://download.proxmox.com/](http://download.proxmox.com/)（PVE官方模板镜像下载地址）

[http://download.proxmox.com/images/system/](http://download.proxmox.com/images/system/)（LXC容器模板）

## 1.修改时区
```bash
sudo timedatectl set-timezone Asia/Shanghai
```

## 2.执行以下代码
```bash
sudo systemctl status ssh #查看有没有开启ssh，如果开启可以不执行以下命令
echo -e "PasswordAuthentication yes\nPermitRootLogin yes" >        /etc/ssh/sshd_config.d/edit.conf #它创建了一个配置文件，强制 SSH 服务器接受密码登录并且允许 root 用户用密码远程登录。
```

（无须执行）

```bash
sudo systemctl start ssh #开启ssh
sudo systemctl stop ssh #停用ssh服务
sudo systemctl enable ssh #将ssh服务设置为开机自启动
```

## 3.执行以下代码（建议看懂之后分步执行）
```bash
sudo apt update -y #刷新软件源列表
sudo apt upgrade -y #升级所有可升级的已安装软件包
apt install curl -y #安装curl
bash <(curl -sSL https://linuxmirrors.cn/main.sh) #进行系统换源
bash <(curl -sSL https://linuxmirrors.cn/docker.sh) #docker换源以及安装
sudo apt install net-tools -y #安装net tools
```

## 4.加速镜像地址更换命令
(特别鸣谢：红茶海)

```bash
bash <(curl -sSL https://raw.githubusercontent.com/GoGoBlacktea/Home.Data.Center/main/docker.speeder.sh)
```

## 5.部署portainer
```dockerfile
sudo docker volume create portainer_data
sudo docker run -d -p 8000:8000 \
-p 9443:9443 \
--name portainer \
--restart=always \
-v /var/run/docker.sock:/var/run/docker.sock \
-v portainer_data:/data \
6053537/portainer-ce:2.24.1
```

## 6.解决lxc容器的乱码问题
首先，使用命令查询自己lxc容器中的语言包。

```bash
locale
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/webp/55551868/1755970231447-7a4c773a-52ae-48b5-b59b-3ebf116dd4cd.webp)

我们需要安装中文的安装包，使用以下命令进行安装。（pdup和pgdn可以快速翻页）

```dockerfile
sudo dpkg-reconfigure locales
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/webp/55551868/1755970231491-00e566f6-e8c1-4523-af88-76d7436cf020.webp)

出现以下界面，即安装包的安装程序，一直按↓键找到以下两个安装包为止：

+  **zh_CN.UTF-**  **8 UTF-8** 
+  **en_US.UTF-**  **8 UTF-8** 

按下空格键以进行选中

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/webp/55551868/1755970231459-39d0ba00-d2be-4805-8db0-13a5a5336141.webp)

按下两次回车确认安装。

执行以下命令以切换安装包为中文语言。

```dockerfile
LANG=zh_CN.UTF-8
LC_ALL=zh_CN.UTF-8
```

再次执行命令，查看此时的语言包。

```dockerfile
locale
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/webp/55551868/1755970231493-167076af-cc2e-40da-bccb-94cb52815cf5.webp)

发现已经完成更换。

## 7.添加macvlan
（macvlan的特性决定了使用该网络的容器将与宿主机隔离）

```dockerfile
sudo docker network create -d macvlan \
 --subnet=192.168.4.0/23 \
 --ip-range=192.168.4.0/24 \
 --gateway=192.168.5.1 \
 -o parent=eth0 \
macvlan1

sudo docker network create -d macvlan \
 --subnet=192.168.4.0/23 \
 --ip-range=192.168.5.0/24 \
 --gateway=192.168.4.2 \
 -o parent=eth0 \
macvlan2
 #如果你使用DHCP，那请你限制好ip-range，如果你一般都是用静态地址，ip-range随意
```

8.至此，以上应该可以当成一个模板了

# 四、MariaDB和PostgreSQL的模板部署(抄茶佬的)
## 1.MariaDB
```dockerfile
apt update
apt install curl apt-transport-https
mkdir -p /home/MariaDB
cd /home/MariaDB
curl -LsSO https://r.mariadb.com/downloads/mariadb_repo_setup
chmod +x mariadb_repo_setup
./mariadb_repo_setup
```

```dockerfile
apt update
apt-get install mariadb-server mariadb-client mariadb-backup
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/50073755/1754968661490-f2dea0f7-cf65-48a6-9417-b2fd0f1e9b9b.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_42%2Ctext_5Zad57qi6Iy26IGK5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_42%2Ctext_5Zad57qi6Iy26IGK5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fformat%2Cwebp)

```dockerfile
root@MariaDB:/home/MariaDB# mysql

MariaDB [(none)]> SELECT VERSION();
+------------------------+
| VERSION()              |
+------------------------+
| 12.0.2-MariaDB-ubu2404 |
+------------------------+
1 row in set (0.000 sec)
```

```dockerfile
MariaDB [(none)]> SHOW VARIABLES LIKE 'collation%';
+----------------------+-----------------------+
| Variable_name        | Value                 |
+----------------------+-----------------------+
| collation_connection | latin1_swedish_ci     |
| collation_database   | utf8mb4_uca1400_ai_ci |
| collation_server     | utf8mb4_uca1400_ai_ci |
+----------------------+-----------------------+
3 rows in set (0.000 sec)

MariaDB [(none)]> SHOW VARIABLES LIKE 'character%';
+--------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
| Variable_name            | Value                                                                                                                                   |
+--------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
| character_set_client     | latin1                                                                                                                                  |
| character_set_collations | utf8mb3=utf8mb3_uca1400_ai_ci,ucs2=ucs2_uca1400_ai_ci,utf8mb4=utf8mb4_uca1400_ai_ci,utf16=utf16_uca1400_ai_ci,utf32=utf32_uca1400_ai_ci |
| character_set_connection | latin1                                                                                                                                  |
| character_set_database   | utf8mb4                                                                                                                                 |
| character_set_filesystem | binary                                                                                                                                  |
| character_set_results    | latin1                                                                                                                                  |
| character_set_server     | utf8mb4                                                                                                                                 |
| character_set_system     | utf8mb3                                                                                                                                 |
| character_sets_dir       | /usr/share/mariadb/charsets/                                                                                                            |
+--------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
9 rows in set (0.001 sec)

```

修改远程访问的配置

vi /etc/mysql/mariadb.conf.d/50-server.cnf

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/50073755/1758027116261-4954bd9b-af3e-42e6-a8c4-8da076700985.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_26%2Ctext_5Zad57qi6Iy26IGK5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_26%2Ctext_5Zad57qi6Iy26IGK5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fformat%2Cwebp)

```dockerfile
mariadb -u root -p
show databases;           					#查看数据库
SELECT user, host FROM mysql.user;	#查看用户
#以下是创建phpipam数据库、用户并赋权
CREATE DATABASE phpipam;
CREATE USER 'phpipam'@'%' IDENTIFIED BY 'phpipampassword';
GRANT ALL PRIVILEGES ON phpipam.* TO 'phpipam'@'%';
GRANT SELECT ON mysql.user TO 'phpipam'@'%';
FLUSH PRIVILEGES;
```

注意点

<font style="color:rgba(0, 0, 0, 0.9);">MariaDB 12.0.2 里 </font>**<font style="color:rgba(0, 0, 0, 0.9);">正式移除</font>**<font style="color:rgba(0, 0, 0, 0.9);"> 的三个历史变量：</font>

    1. <font style="color:rgba(0, 0, 0, 0.9);">big_tables</font>
    - <font style="color:rgba(0, 0, 0, 0.9);">作用：老版本里想让 </font>**<font style="color:rgba(0, 0, 0, 0.9);">所有临时表都落在磁盘</font>**<font style="color:rgba(0, 0, 0, 0.9);"> 时，把 </font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgba(0, 0, 0, 0.03);">big_tables=1</font>`<font style="color:rgba(0, 0, 0, 0.9);"> 写在 my.cnf 里。</font>
    - <font style="color:rgba(0, 0, 0, 0.9);">废弃：10.5 起 </font>**<font style="color:rgba(0, 0, 0, 0.9);">不再起任何作用</font>**<font style="color:rgba(0, 0, 0, 0.9);">，代码里直接忽略；12.0 的发行说明干脆把它从源码里删掉。</font>
    - <font style="color:rgba(0, 0, 0, 0.9);">升级影响：配置文件里如果还留着这行，启动只会得到一行 </font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgba(0, 0, 0, 0.03);">unknown variable</font>`<font style="color:rgba(0, 0, 0, 0.9);"> 警告，直接删掉即可。</font>
    2. <font style="color:rgba(0, 0, 0, 0.9);">large_page_size（注意不是 innodb_large_prefix）</font>
    - <font style="color:rgba(0, 0, 0, 0.9);">作用：早期允许在启动时把 InnoDB 页大小改成 32 k/64 k（</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgba(0, 0, 0, 0.03);">--large-page-size=32768</font>`<font style="color:rgba(0, 0, 0, 0.9);">）。</font>
    - <font style="color:rgba(0, 0, 0, 0.9);">废弃：10.5 开始代码就固定 16 k 页大小，变量形同虚设；12.0 正式摘出源码。</font>
    - <font style="color:rgba(0, 0, 0, 0.9);">升级影响：同样只会报 </font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgba(0, 0, 0, 0.03);">unknown variable</font>`<font style="color:rgba(0, 0, 0, 0.9);">，删除配置行即可。</font>
    3. <font style="color:rgba(0, 0, 0, 0.9);">storage_engine（老名字 default_storage_engine）</font>
    - <font style="color:rgba(0, 0, 0, 0.9);">作用：5.x 时代用来把 </font>**<font style="color:rgba(0, 0, 0, 0.9);">全局默认引擎</font>**<font style="color:rgba(0, 0, 0, 0.9);"> 设成 MyISAM 或 InnoDB。</font>
    - <font style="color:rgba(0, 0, 0, 0.9);">废弃：10.5 起系统始终默认 InnoDB，变量被忽略；12.0 源码级删除。</font>
    - <font style="color:rgba(0, 0, 0, 0.9);">升级影响：</font>
        * <font style="color:rgba(0, 0, 0, 0.9);">my.cnf 里若写 </font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgba(0, 0, 0, 0.03);">storage_engine=MyISAM</font>`<font style="color:rgba(0, 0, 0, 0.9);"> 会启动失败，需改为 </font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgba(0, 0, 0, 0.03);">default_storage_engine=MyISAM</font>`<font style="color:rgba(0, 0, 0, 0.9);">（或干脆删掉，直接用 InnoDB）。</font>
        * <font style="color:rgba(0, 0, 0, 0.9);">SQL 脚本里若用 </font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgba(0, 0, 0, 0.03);">SET storage_engine=...</font>`<font style="color:rgba(0, 0, 0, 0.9);"> 也会报错，需改写成 </font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgba(0, 0, 0, 0.03);">SET default_storage_engine=...</font>`<font style="color:rgba(0, 0, 0, 0.9);">。</font>



## 2.PostgreSQL
```dockerfile
sudo apt update
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```

```dockerfile
sudo apt update
#16 17 二选一
sudo apt -y install postgresql-16     #安装16
sudo apt -y install postgresql-17     #安装17
```

修改配置文件：

    1. 修改监听IP和端口

/etc/postgresql/16/main/postgresql.conf

/etc/postgresql/17/main/postgresql.conf

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2024/png/50073755/1732149480645-db82c2f7-24a3-4127-93b7-1351ad6734d3.png?x-oss-process=image%2Fformat%2Cwebp%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_38%2Ctext_5Zad57qi6Iy26IGK5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_38%2Ctext_5Zad57qi6Iy26IGK5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

    2. 修改可连接IP网段

/etc/postgresql/16/main/pg_hba.conf

/etc/postgresql/17/main/pg_hba.conf

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/50073755/1757489787214-32971fb7-e979-4bdd-b285-16fbacae9c36.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_19%2Ctext_5Zad57qi6Iy26IGK5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_19%2Ctext_5Zad57qi6Iy26IGK5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fformat%2Cwebp)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/50073755/1758028738931-21d6e208-a422-4a6e-8b2d-c6822a5c5e08.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_20%2Ctext_5Zad57qi6Iy26IGK5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_20%2Ctext_5Zad57qi6Iy26IGK5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fformat%2Cwebp)

```dockerfile
systemctl restart postgresql
su postgres
psql
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/50073755/1757490812332-712def12-7450-48a2-bfb9-d732b8fca44c.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_16%2Ctext_5Zad57qi6Iy26IGK5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_16%2Ctext_5Zad57qi6Iy26IGK5oqA5pyv%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fformat%2Cwebp)

```dockerfile
UPDATE pg_database SET datistemplate = false WHERE datname = 'template1';
DROP DATABASE template1;
CREATE DATABASE template1
  WITH TEMPLATE = template0
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.utf8'
  LC_CTYPE = 'C.utf8';
UPDATE pg_database SET datistemplate = true WHERE datname = 'template1';
UPDATE 1
DROP DATABASE
CREATE DATABASE
UPDATE 1
\l
```

```dockerfile
postgres=# \l
                                                 List of databases
   Name    |  Owner   | Encoding  | Locale Provider | Collate | Ctype | Locale | ICU Rules |   Access privileges
-----------+----------+-----------+-----------------+---------+-------+--------+-----------+-----------------------
 postgres  | postgres | SQL_ASCII | libc            | C       | C     |        |           |
 template0 | postgres | SQL_ASCII | libc            | C       | C     |        |           | =c/postgres          +
           |          |           |                 |         |       |        |           | postgres=CTc/postgres
 template1 | postgres | SQL_ASCII | libc            | C       | C     |        |           | =c/postgres          +
           |          |           |                 |         |       |        |           | postgres=CTc/postgres
(3 rows)

postgres=# UPDATE pg_database SET datistemplate = false WHERE datname = 'template1';
DROP DATABASE template1;
CREATE DATABASE template1
  WITH TEMPLATE = template0
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.utf8'
  LC_CTYPE = 'C.utf8';
UPDATE pg_database SET datistemplate = true WHERE datname = 'template1';
UPDATE 1
DROP DATABASE
CREATE DATABASE
UPDATE 1
postgres=# \l
                                                 List of databases
   Name    |  Owner   | Encoding  | Locale Provider | Collate | Ctype  | Locale | ICU Rules |   Access privileges
-----------+----------+-----------+-----------------+---------+--------+--------+-----------+-----------------------
 postgres  | postgres | SQL_ASCII | libc            | C       | C      |        |           |
 template0 | postgres | SQL_ASCII | libc            | C       | C      |        |           | =c/postgres          +
           |          |           |                 |         |        |        |           | postgres=CTc/postgres
 template1 | postgres | UTF8      | libc            | C.utf8  | C.utf8 |        |
```

```dockerfile
CREATE USER "MPAdmin" WITH PASSWORD 'MPAdminPassword';
CREATE DATABASE "MPDB";
ALTER DATABASE "MPDB" OWNER TO "MPAdmin";
GRANT ALL PRIVILEGES ON DATABASE "MPDB" TO "MPAdmin";
#DBA是一个统管的角色，在坏库或者忘记用户名时可以使用DBA操作
CREATE USER "DBA" WITH PASSWORD 'DBAPassword';
GRANT ALL PRIVILEGES ON DATABASE "MPDB" TO "DBA";
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO "DBA";

#若你想删除该数据库和使用者，使用以下命令
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = '数据库名称' 
  AND pid <> pg_backend_pid();
#以上为清除该数据库所有连接
DROP DATABASE 数据库名称;
DROP USER 数据库使用者;
```

oci镜像部署请参见immich的部署

## 3.Navicat的连接免费使用
[直接下载 | Navicat](https://www.navicat.com.cn/download/direct-download?product=navicat170_premium_lite_cs_x64.exe&location=1)

# ~~五、虚拟机OVM（存储系统的安装）~~（我改成unraid了）
~~1.下载虚拟机 ~~~~ ~~[~~https://sourceforge.net/projects/openmediavault/files/iso/7.0-32/openmediavault_7.0-32-amd64.iso~~](https://sourceforge.net/projects/openmediavault/files/iso/7.0-32/openmediavault_7.0-32-amd64.iso)

~~<font style="color:rgb(0, 0, 0);">2.（配置也可以询问AI）</font>~~~~<font style="color:rgb(215, 37, 43);">先安装虚拟机，再直通硬盘</font>~~~~。磁盘总线选择VirtIO Block，容量给个10G，无缓存~~

~~注意，请先在命令行中输入 ~~`~~omv-firstaid~~`~~之后修改密码，然后再进入webui~~

~~3.配置：请率先安装一个插件，里面含有md，安装完成之后， 可以使用raid，但是这里由于我使用的是高贵的西数红盘plus，所以我单盘读写，使用EXT4文件系统。~~

~~4.与大部分的文件系统nas一样，我们需要擦除硬盘，先创建文件系统，再创建共享文件夹，再创建用户并且赋予用户权限，之后启动smb协议和nfs协议即可。~~

~~细节注解截图：~~

+ ~~ ~~~~ nfs挂载的时候，~~~~  ~~~~读/写，192.168.4.0/23，选项：~~

```dockerfile
all_squash,anongid=0,anonuid=0,insecure,rw,subtree_check
```

~~<font style="color:#DF2A3F;">这样的配置</font>~~~~**<font style="color:#DF2A3F;">非常危险</font>**~~~~<font style="color:#DF2A3F;">，请大家酌情考虑，因为：</font>~~

1. ~~**<font style="color:#DF2A3F;">任何能访问NFS的客户端都拥有root权限</font>**~~
2. ~~**<font style="color:#DF2A3F;">可以修改、删除服务器上的任何文件</font>**~~
3. ~~**<font style="color:#DF2A3F;">存在严重的安全风险</font>**~~

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/webp/55551868/1755970231494-8d79b9d6-3d38-4df2-b4f1-8bc111473642.webp)

+ ~~电源设置，务必设置为关机，否则虚拟机将无法关机~~

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/webp/55551868/1755970232088-38d28b59-3e09-48d5-b9ec-cf8701976da2.webp)

+ ~~如果你希望你的权限较高，可以给到root权限即可，不要跟图中一样，我一般给到root即可~~

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/webp/55551868/1755970232111-ae4dde52-ce7e-4221-a718-71c20afc499a.webp)

+ ~~要有足够的权限，请选~~~~择这一项~~

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/webp/55551868/1755970232104-1fc4eb65-adbc-49a6-b7f3-2a86551e9345.webp)

~~<font style="color:rgb(215, 37, 43);">5.如果你有挂载在nfs上的docker，当你启动pve的时候，你发现docker没有启动，有可能是因为，你的OMV比docker启动的要慢，所以这个时候，你可以给OMV设置启动顺序，顺序为0，时间30，意味着OMV将第一个启动，之后30秒其他LXC启动。</font>~~

~~<font style="color:rgb(215, 37, 43);">6.务必把你直通的硬盘备份关掉，否则等你备份的时候，你宿主机就炸了，那就好玩了！！！！！！！</font>~~

unraid的安装方法，请参见此pdf[制作 Unraid 开心版启动盘_v6.12.11.pdf](https://www.yuque.com/attachments/yuque/0/2026/pdf/55551868/1771914824129-4ae93504-8975-4b1c-b243-fc2fa685b032.pdf)



# 六、硬盘的直通
1.硬盘直通(请直接查看链接！作者：喝红茶聊技术  [PVE下硬盘直通](https://www.bilibili.com/read/cv39571871/?opus_fallback=1) 出处：bilibili））

大致步骤如下：

+ 确保开启了VT-D，或者也可能被标记为“Virtualization Support”、“VT for Direct I/O”、“Trusted Execution” 
+ 修改内核参数，编辑`<font style="color:rgb(215, 37, 43);">/e</font>~~<font style="color:rgb(215, 37, 43);"></font>~~<font style="color:rgb(215, 37, 43);">tc/default/grub</font>`<font style="color:rgb(0, 0, 0);">文件找到</font>GRUB_CMDLINE_LINUX_DEFAULT变量

 例如：`<font style="color:rgb(215, 37, 43);">GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on  iommu=pt"</font>`

+ <font style="color:rgb(0, 0, 0);">保存更改后运行</font> `update-grub` 更新GRUB配置。
+ 编辑`/etc/modules`文件，添加以下模块以启用IOMMU支持：

    添加以下内容

```dockerfile
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

+ 保存后重启系统
+ 输入命令`dmesg | grep -e DMAR -e IOMMU`验证IOMMU是否启动。

与作者不同的是，我在分别执行完 `update-grub`  和  `dmesg | grep -e DMAR -e IOMMU`  之后，系统给到的结果如下

```dockerfile
#以下就是执行update-grub的反馈
root@pve:~# update-grub
Generating grub configuration file ...
W: This system is booted via proxmox-boot-tool:
W: Executing 'update-grub' directly does not update the correct configs!
W: Running: 'proxmox-boot-tool refresh'

Copying and configuring kernels on /dev/disk/by-uuid/EA72-BD09
        Copying kernel and creating boot-entry for 6.8.12-9-pve
Copying and configuring kernels on /dev/disk/by-uuid/EA73-473A
        Copying kernel and creating boot-entry for 6.8.12-9-pve
Found linux image: /boot/vmlinuz-6.8.12-9-pve
Found initrd image: /boot/initrd.img-6.8.12-9-pve
/usr/sbin/grub-probe: error: unknown filesystem.
/usr/sbin/grub-probe: error: unknown filesystem.
Adding boot menu entry for UEFI Firmware Settings ...
done



root@pve:~# dmesg | grep -e DMAR -e IOMMU
[    0.013320] ACPI: DMAR 0x00000000390B2000 000088 (v01 INTEL  EDK2     00000002      01000013)
[    0.013348] ACPI: Reserving DMAR table memory at [mem 0x390b2000-0x390b2087]
[    0.161227] DMAR: Host address width 39
[    0.161228] DMAR: DRHD base: 0x000000fed90000 flags: 0x0
[    0.161234] DMAR: dmar0: reg_base_addr fed90000 ver 4:0 cap 1c0000c40660462 ecap 29a00f0505e
[    0.161236] DMAR: DRHD base: 0x000000fed91000 flags: 0x1
[    0.161241] DMAR: dmar1: reg_base_addr fed91000 ver 5:0 cap d2008c40660462 ecap f050da
[    0.161243] DMAR: RMRR base: 0x00000042000000 end: 0x000000527fffff
[    0.161246] DMAR-IR: IOAPIC id 2 under DRHD base  0xfed91000 IOMMU 1
[    0.161247] DMAR-IR: HPET id 0 under DRHD base 0xfed91000
[    0.161248] DMAR-IR: Queued invalidation will be enabled to support x2apic and Intr-remapping.
[    0.162767] DMAR-IR: Enabled IRQ remapping in x2apic mode
[    0.358109] pci 0000:00:02.0: DMAR: Skip IOMMU disabling for graphics
[    0.432467] DMAR: No ATSR found
[    0.432468] DMAR: No SATC found
[    0.432470] DMAR: IOMMU feature fl1gp_support inconsistent
[    0.432471] DMAR: IOMMU feature pgsel_inv inconsistent
[    0.432473] DMAR: IOMMU feature nwfs inconsistent
[    0.432474] DMAR: IOMMU feature dit inconsistent
[    0.432476] DMAR: IOMMU feature sc_support inconsistent
[    0.432477] DMAR: IOMMU feature dev_iotlb_support inconsistent
[    0.432479] DMAR: dmar0: Using Queued invalidation
[    0.432483] DMAR: dmar1: Using Queued invalidation
[    0.435977] DMAR: Intel(R) Virtualization Technology for Directed I/O
```

在查询完deepseek之后，虽然和茶佬的输出有所不同总的来说，我都成功了。

2.硬盘直通

`ls -l /dev/disk/by-id/`

将硬盘直通给对于的虚拟机：

`qm set <VM ID> --<磁盘 ID> /dev/disk/by-id/<磁盘标识>`

```plain
qm set 902 --sata0 /dev/disk/by-id/ata-WDC_WD80EFPX-6XXXXXX_WD-XXXXXXXX
qm set 902 --sata1 /dev/disk/by-id/ata-WDC_WD80EFPX-6XXXXXX_WD-XXXXXXXX
qm set 902 --sata2 /dev/disk/by-id/ata-WDC_WD80EFPX-6XXXXXX_WD-XXXXXXXX
```

# <font style="color:rgb(0, 0, 0);">七、文件的挂载</font>
1.在挂载文件的时候，总体思路是，先把unraid中的文件挂在到pve中，然后推送给每个LXC身上。

首先，我们可以执行以下命令，以此来安装nfs的客户端。（注意，挂载的时候，一定要挂载空文件夹，否则会直接清除）

```plain
dpkg -l | grep nfs-common #查看是否有nfs-common
apt-get update
apt-get install nfs-common #更新nfs-common
```

2.接下来挂载文件即可，挂载的时候，可以只选取片段来进行挂载，后期推送给lxc容器务必使用命令行推送。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/webp/55551868/1755970232153-8c81e702-0582-4c1f-a4fc-ae74beca0f9a.webp)

```bash
pct set VMID -mountpoint /NFSPath,mp=/MountPath #左侧主机右侧lxc
#例： pct set 101 -mp0 /mnt/pve/Share,mp=/mnt/nfs/Share
```

# 八、电子看板的部署（InfluxDB）（Grafana）
1.下载InfluxDB和Grafana（最好挂梯）

```plain
wget https://download.influxdata.com/influxdb/releases/influxdb_1.8.10_amd64.deb
dpkg -i influxdb_1.8.10_amd64.deb
#以上为安装influxDB
apt-get install -y adduser libfontconfig1 musl
wget https://dl.grafana.com/oss/release/grafana_12.1.0_amd64.deb
dpkg -i grafana_12.1.0_amd64.deb
#以上为安装Grafana
```

也可以docker-compose安装Grafana

```plain
docker run -d --name=grafana -p 3000:3000 grafana/grafana-oss
```

Dockerfile：我在使用以下的时候,删除环境变量（有环境变量无法成功部署）如果用不了记得执行最后的两行命令

```dockerfile
version: '3'
services:
  grafana:
    image: grafana/grafana-oss
    container_name: Grafana
    hostname: Grafana
    ports:
      - '3000:3000'
    volumes:
      - /mnt/appdata/grafana/storage:/var/lib/grafana               #宿主机目录要给权限。chmod -R 777 /home/ctac/grafana/storage
      - /mnt/appdata/grafana/logs:/var/log/grafana
      - /mnt/appdata/grafana/plugins:/var/lib/grafana/plugins       #宿主机目录要给权限。chmod -R 777 /home/ctac/grafana/plugins
      - /mnt/appdata/grafana/provisioning:/etc/grafana/provisioning
    environment:
      - PUID=0
      - PGID=0
      - TZ=Etc/GMT-8
    restart: always
#sudo chown -R 1000:1000 /mnt/appdata/grafana 改变使用者	
#sudo chmod -R 775 /mnt/appdata/grafana 改变权限
```

2.修改配置文件：（记得去掉星号）

```plain
修改配置文件：
nano /etc/influxdb/influxdb.conf
在530行找到以下修改如下内容：
        enabled = true #开启udp			
        bind-address = "0.0.0.0:8089" #监听端口，此处为UDP端口，不要和下面HTTP端口混淆			
        database = "proxmox"			
        batch-size = 1000			
        batch-timeout = "1s"
#在 nano 编辑器中，你可以通过以下步骤直接跳转到特定行：
跳转到指定行
按 Ctrl + _（即 Ctrl 键和下划线键）。
会出现一个提示 Enter line number:。
输入你要跳转到的行号，然后按 Enter。
```

3.启动DB的服务和Grafana的服务

```plain
systemctl start influxdb
systemctl enable influxdb
systemctl start grafana-server
systemctl enable grafana-server
```

4.初始化账号和数据库(一行一行执行)

```plain
influx
>CREATE USER "admin" WITH PASSWORD '123456' WITH ALL PRIVILEGES
>create database proxmox
>show users
>show databases
>quit
```

5.后面配置PVE和DB对接

点击数据中心-指标服务器-添加DB

6.访问Grafana

`浏览器访问Grafana默认端口3000`

`默认用户名密码 admin/admin`

7.这不先选个中文简体？然后点击连接-数据源-influxDB

`由于我使用macvlan使用习惯了，我在这里使用macvlan部署的grafana，会出现一个问题，macvlan与宿主机不连通，所以我后来改成了bridge模式，你可以看看你会不会出现这样的误区。`

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/webp/55551868/1755970232128-e1624939-644b-418d-8be8-47ad25eea5a5.webp)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/webp/55551868/1755970232398-43af5ad4-7b30-4f9f-87bd-4e16b94fb3f9.webp)

8.选择仪表盘-创建仪表盘-导入仪表盘-`10048`即可使用

---

<font style="color:#DF2A3F;">（2025年9月5日更新）</font>

<font style="color:#000000;">9.近期茶佬发布了新的教程，链接</font>[PVE 9.0 性能监控 v2.0(PVE 8.X也通用)](https://www.yuque.com/hongcha-6c6o6/klr1op/mfypd4ndutcaw7bd?singleDoc#%E6%94%B6%E8%B5%B7)<font style="color:#000000;">，</font>[PT监控](https://www.yuque.com/hongcha-6c6o6/klr1op/sofkr4mqd6egg4if?singleDoc#)<font style="color:#000000;">，照着新的教程，升级一下。</font>

+ <font style="color:#000000;">鉴于之前已经二进制形式安装了influxdb而本次改为docker安装，所以我先禁用并删除之前的influxdb，命令如下</font>

```bash
# 停止InfluxDB服务
sudo systemctl stop influxdb

# 禁用开机启动
sudo systemctl disable influxdb
#我这里仅仅停止服务并禁止开机自启，一方面是为了保留一个备份，另一方面是因为我也不会完全删除二进制文件
```

+ 接着使用docker形式来部署influxDB

```dockerfile
mkdir -p /mnt/appdata/InfluxDB2/data
mkdir -p /mnt/appdata/InfluxDB2/config
```

```dockerfile
services:
  influxdb:
    image: influxdb:latest
    container_name: influxdb2
    hostname: InfluxDB2
    ports:
      - 8086:8086
    environment:
      - TZ=Etc/GMT-8
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=Password
      - DOCKER_INFLUXDB_INIT_ORG=Home						#PVE配置时需要
      - DOCKER_INFLUXDB_INIT_BUCKET=Proxmox			#PVE配置时需要
    volumes:
      - type: bind
        source: /mnt/appdata/InfluxDB2/data
        target: /var/lib/influxdb2
      - type: bind
        source: /mnt/appdata/InfluxDB2/config
        target: /etc/influxdb2
    restart: always
```

+ 按照配置文件登录influxDB

创建水桶buckets，例如Proxmox Temp

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1757172920584-1352af6c-a668-47d1-bf3a-cb66ccf4bccd.png)

创建API Token：

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1757172983973-5d4ddccb-c5c1-4bb1-a5c3-6694fd3ee7ba.png)

创建新的指标服务器：

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1757173101270-fc29fa2b-f926-486a-9edf-4616c97d0022.png)

去往grafana<font style="color:#DF2A3F;">添加新连接</font>,相比之前的二进制会少填写很多的信息：

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1757173251441-9411bf4a-d7f9-455d-9b49-a33c26a9b8f6.png)

与上面一样，仪表盘-导入-10048即可。

10.PT监控的部署

+ 镜像：`caseyscarborough/qbittorrent-exporter:latest`

                 `prom/prometheus`

```dockerfile
version: '3.8'

services:
  qbittorrent-exporter:
    image: caseyscarborough/qbittorrent-exporter:latest
    container_name: qBittorrent-exporter
    hostname: qB-exporter
    environment:
      - QBITTORRENT_USERNAME=username #填入自己的qb的用户名
      - QBITTORRENT_PASSWORD=password #填入自己的qb的密码
      - QBITTORRENT_BASE_URL=http://192.168.4.11:8080  #qBittorrent的登录页面
      - TZ=Etc/GMT-8
    ports:
      - "17871:17871"
```

在部署完成之后，可以查看日志看看探针能否正常运行，或者使用以下命令：

```bash
curl http://localhost:17871/metrics #如果返回一些qb里面的数据则说明成功
```

接着部署prometheus

```dockerfile
version: '3.8'
services:
  prometheus:
    image: prom/prometheus
    container_name: Prometheus
    hostname: Prometheus
    ports:
      - 9090:9090
    environment:
      - TZ=Etc/GMT-8
    volumes:
      - /mnt/appdata/Prometheus/conf:/etc/prometheus           #最终使用的配置文件
      - /mnt/appdata/Prometheus/confbak:/etc/prometheusbak     #配置文件备份缓冲区
    restart: always
```

部署完成之后将普罗米修斯停止运行，使用以下命令在普罗米修斯的`/mnt/appdata/Prometheus/conf`文件夹下新建一个yml文件：

```bash
sudo cat > /mnt/appdata/Prometheus/conf/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'qbittorrent'
    static_configs:
      - targets: ['192.168.4.17:17871'] #上面的探针ip以及port
        labels:
          instance: 'qbittorrent-server'
EOF
```

之后进入普罗米修斯的控制面板，如果控制面板显示qb的探针up，则正常。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1757178180160-6bdfa04d-f027-44dd-80a2-3ef67462d998.png)

进入grafana，新增数据源，只需要写一个ip地址，然后进入仪表盘，新增15116，即可监控qb。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1757178293509-bda0532f-1018-4433-9e78-56c425714b17.png)

# 九、DDNS-GO以及Nginx-Proxy-Manager的部署
1.新开LXC容器，使用到的docker是`jlesage/nginx-proxy-manager`和`jeessy/ddns-go`。

2.拉取镜像

```dockerfile
docker pull jlesage/nginx-proxy-manager:v25.06.1
docker pull jeessy/ddns-go
```

3.编辑compose

```dockerfile
version: '3.8'
services:
  ddns-go:
    image: jeessy/ddns-go
    container_name: ddns-go
    restart: always
    network_mode: host
    volumes:
      - /mnt/appdata/ddns-go:/config
```

```dockerfile
version: '3'
services:
  nginx-proxy-manager:
    image: jlesage/nginx-proxy-manager:v25.06.1
    container_name: nginx-proxy-manager
    network_mode: host
    volumes:
      - /mnt/appdata/nginx-proxy-manager/config:/config #映射这一个足够了，里面包含了很多的配置信息
    restart: always
```

4.获取域名，推荐地址（us.kg），自己谷歌搜索一下，买一个一直用。

5.注册一个cloudflare账号，将域名托管到cf，之后用ddns-go监听域名，用npm实现反向代理即可。

（此处不再赘述）

6.域名申请链接：

[https://domain.digitalplat.org/](https://domain.digitalplat.org/)

cloudflare官方链接：

[科赋锐信息科技Cloudflare](https://www.cloudflare.com/zh-cn/)

推荐教程链接：（结合起来耐心看完）

[SP.05.反向代理Nginx_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1HKkZYVEBR/?spm_id_from=333.1387.upload.video_card.click&vd_source=c754e72425da37d66bb27d9d4da8193d)

[05.反向代理Nginx](https://www.yuque.com/hongcha-6c6o6/phdu9c/upu31c6df3mug6ur?singleDoc#)

[https://www.youtube.com/watch?v=go5xfSc3VaE&t=1096s](https://www.youtube.com/watch?v=go5xfSc3VaE&t=1096s)

7.友情提示：（DNS地址千万别用自己家的路由器地址，最好是114.114.114.114和自家附近的运营商dns）

8.顺便加一条，当你的ikuai做主路由的时候，为了获取ipv6地址你可以像我一样配置

+ ipv6配置：

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756555017326-45c783b6-840e-48c1-9b86-d64a2a765374.png)

+ ACL规则

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756555091306-188ad652-c156-4598-9b7e-448cd73b6461.png)

+ 在配置允许进入的ipv6地址时，可以采用`::aaaa:aaaa:aaaa:aaaa/::ffff`的写法来固定ipv6地址的后面的eui64。

# 十、PVE中的NAS虚拟机UNRAID的部署
1.购买U盘，记得购买带有guid的u盘

2.下载unraid happy-version，不要问我在哪里找，要的话私信我我发你。

3.下载unraid的u盘工具，[Unleash Your Hardware](https://unraid.net/download)。

4.使用U盘工具擦除掉U盘，记录下u盘的guid，并且更名U盘的名字为UNRAID。

5.复制文件进入U盘，并打开go文件，修改guid为U盘的guid，然后管理员权限执行windows批量处理文件。

6.创建虚拟机,不使用任何介质，机型选新的机型，取消预注册密钥，BIOS使用UEFI，磁盘给1G都嫌多，CPU可以使用host。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756296656765-b587a3d4-43a9-4ebf-a07b-7c0077fec50c.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756296720383-14e659e5-bce6-4720-a8d1-c5c5e013d8d5.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756296768793-d7b13fe3-522c-407d-bab4-89aa1d506c2c.png)

7.建立好之后，挂载U盘上去，然后按照常规流程登录即可，接下来就是直通硬盘（参照上面）。

# 十一、导航面板sun-panel和HD-icons的部署
1.只提供compose，镜像自己拉，后面设置自己搞，记得备份。

```dockerfile
version: "3.2"
services:
  sun-panel:
    image: 'hslr/sun-panel:latest'
    container_name: sun-panel
    hostname: Sun-Panel
    volumes:
      - /mnt/appdata/sun-panel/conf:/app/conf
      - /mnt/appdata/sun-panel/uploads:/app/uploads
      - /mnt/appdata/sun-panel/database:/app/database
      - /var/run/docker.sock:/var/run/docker.sock
      - /mnt/appdata/sun-panel/runtime:/app/runtime
    ports:
      - 3002:3002
    environment:
      TZ: Etc/GMT-8
    networks:
      macvlan1:
        ipv4_address: 192.168.4.8  # 正确的放置位置：在服务网络配置中
    restart: always

networks:
  macvlan1:
    external: true  # 声明使用外部网络
```

```dockerfile
version: "3.8"
services:
  hd-icons:
    image: xushier/hd-icons:latest
    container_name: HD-Icons
    ports:
      - 50560:50560
    volumes:
      - /mnt/appdata/HD-Icons/icons:/app/icons
      # 若需要自定义字体可添加映射/app/static/font路径，在对应主机路径下放入字体文件，font_zh.ttf为中文，font_en.ttf为英文。然后ctrl+F5强制刷新首页生效，或者重启容器生效。
      #- /mnt/user/appdata/HD-Icons/font:/app/static/font
    networks:
      macvlan2:
        ipv4_address: 192.168.4.9  # 设置固定IP地址
    # environment:
    #   首次使用日志若一直显示卡在git clone，或者后续更新一直出错，那么是网络无法连接github，可添加 ALL_PROXY 变量设置 HTTP 代理解决，将下面的 http://192.168.1.2:7890 换一下地址和端口即可。
    #   - ALL_PROXY=http://192.168.1.2:7890
    #   自定义复制地址的前缀，若不填且切换到了云端模式则默认为 HD-Icons 项目图标真实地址前缀。
    #   - CUSTOM_URL=http://xxx.xxx.xxx/icons/HD-Icons
    #   自定义标题和网页标签页，不填默认为"小迪的图标库"。
    #   - TITLE=小迪的图标库
    restart: unless-stopped

networks:
  macvlan2:
    external: true  # 使用外部已创建的macvlan网络
```

# 十二、qbittorrent和百度网盘下载的部署
1.compose

```dockerfile
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:5.0.1
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/GMT-8
      - WEBUI_PORT=8080
      - TORRENTING_PORT=6881											#端口可修改（bridge模式下与下面相同）
    volumes:
      - /docker固化路径/qbittorrent/appdata:/config:rw				#种子文件保存位置
      - /下载临时保存固化路径:/downloads/qbincomplete:rw		#下载临时文件存储
      - /下载保存固化路径/complete1:/downloads/complete1:rw				#下载完成路径1
      - /下载保存固化路径/complete2:/downloads/complete2:rw				#下载完成路径2
    ports:
      - 8080:8080
      - 6881:6881																	#端口可修改（bridge模式下与上面相同）
      - 6881:6881/udp															#端口可修改（bridge模式下与上面相同）
      
#如果使用Bridge模式部署，删除以下部分
    networks:
      macvlan:   #根据你现在的macvlan名称改（与下面的一致）
        ipv4_address: A.B.C.D # 设置固定的 IPv4 地址
    restart: unless-stopped
networks:
  macvlan:   #根据你现在的macvlan名称改（与上面的一致）
    external: true
```

```dockerfile
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:4.6.0
    container_name: qbittorrent-media
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/GMT-8
    volumes:
      - /mnt/appdata/qbittorrent-media/config:/config
      - /mnt/nfs/media:/media #要和mp一起用，必须和mp一样的目录
    networks:
      macvlan1:
        ipv4_address: 192.168.4.11
    restart: always

networks:
  macvlan1:
    external: true
```

```dockerfile
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:4.6.0
    container_name: qbittorrent-download
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/GMT-8
    volumes:
      - /mnt/appdata/qbittorrent-download/config:/config
      - /mnt/nfs/download:/downloads
    networks:
      macvlan1:
        ipv4_address: 192.168.4.12
    restart: always

networks:
  macvlan1:
    external: true
```

2.设置分享：[S2.05.PT下载软件的容器化部署（下）](https://www.yuque.com/hongcha-6c6o6/klr1op/aegp0fev53b2k1vy?singleDoc#)（红茶海）

+ 推荐顺序写入
+ 最大连接数200.每个最大连接数20.全局上传数20.每个上传数量20
+ <font style="color:#DF2A3F;">如果你需要反向代理自己的qb，请按照下面的图片进行设置</font><!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756470514928-2a7a8b08-f5ef-4836-bf4c-472709b7d50a.png)

3.务必在路由器中设置端口转发，使得qb能够连接正常，而并非显示为防火墙

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756611298816-628ab4f0-c5f3-43d9-a43f-352055a31642.png)

连接正常显示为地球<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756611319225-74619fce-b275-447a-ba3c-1b4d19796a68.png)，而有防火墙显示为火焰<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756611346032-2eaa5429-56e3-46a6-bcff-a2f47001e902.png)。

4.其他配置：

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756654701451-46ff1579-65a0-4f37-8ae9-fc67ec7c7d2f.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756635864123-6ac11fba-58b1-4ca8-a128-9dfa9dac325d.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756637977139-e25fa028-f124-4925-a4e7-e10fc38b9a18.png)

5.附带百度网盘的docker compose

```dockerfile
services:
  baidunetdisk:
    image: johngong/baidunetdisk:latest
    container_name: baidunetdisk
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/GMT-8
    volumes:
      - /mnt/appdata/baidunetdisk/config:/config
      - /mnt/nfs/media:/config/baidunetdiskdownload
    networks:
      macvlan1:
        ipv4_address: 192.168.4.17
    restart: always

networks:
  macvlan1:
    external: true
    #注意，由于是通过VNC什么的乱七八糟的连接，所以想要控制这个容器，务必保证你的设备和容器在同一网段
```

# 十三、emby和[moviepilot-v2的部署配置(特别鸣谢：群友🐾Jin)](https://www.yuque.com/kiryusento/unmhad/cifkzsybl4zefym2)
1.创建macvlan并推送目录到容器里面去

2.compose

```dockerfile
services:
  emby:
    image: amilys/embyserver:latest
    container_name: embyserver
    hostname: EmbyServer
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/GMT-8
    volumes:
      - /mnt/appdata/emby/config:/config
      - /mnt/nfs/media:/data/tvshows
   #devices:
      #- "/dev/dri:/dev/dri"    # 这个对于Intel核显硬件加速至关重要,但我是LXC ，我没有，所以注释掉了
    networks:
      macvlan2:
        ipv4_address: 192.168.4.14
    restart: always

networks:
  macvlan2:
    external: true
    name: macvlan2
    #部署完成后去WebUI重启一下
    #神医助手版本我的是2.0.0.22+0497812
```

3.接下来安装神医助手，如果你想更换神医助手的版本，请前往strm assitant的github官方网站下载然后将它移动到emby的插件里面去。

4.神医助手的配置，我这里主要需要的是片头片尾的标记，只讲述这个：

第一步请前往emby中打开生成片头标记

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756728060917-8a6a4f02-b496-4260-a425-c32321f2f566.png)

请在追更模式中打开原生片头探测，可以将主线控控制调整到20，可以增加原生片头探测的速度。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756728075260-0e467420-927e-4f0e-9d87-9b698836f776.png)

接下来在片头片尾探测中打开原生探测增强，接着扫描媒体库即可。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756728086125-4a8b5938-11c8-4ec2-b9e7-75ec7b396f1a.png)

您亦可以使用`tail -f embyserver.txt` 实时查看logs

会出现类似于这样的日志

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756728597652-e2346c22-43fe-41b1-83cf-37ab006e86f2.png)

偷摸放个链接：[IPTV-AIO肥羊版](https://www.yuque.com/hongcha-6c6o6/klr1op/slkgfsikr03mvq1c?singleDoc#)



# 十四、windows虚拟机的部署
1.ios镜像：[登录](https://next.itellyou.cn/Original/Index#cbp=Product?ID=f905b2d9-11e7-4ee3-8b52-407a8befe8d1)

2.驱动ios镜像：[https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso)（这个下载的比较慢，可以直接问我要）

3.下载到PVE中去，然后安装，安装配置看我心情随便点点，设置不一定对，但是用下来没问题就是OK

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756733895025-94204f5a-be95-4b3e-a056-e86145cd68c9.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756733906589-710b8546-162e-4ebe-a2e0-eb7d70f31090.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756778221553-063a1911-9d2c-41d6-9164-b59a8c472c85.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756778325564-c8eb6b23-cb85-468c-bbb3-78f9ada0d3bc.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756778348955-6ec9e677-0d8f-4655-992a-9e9c0045433b.png)

后面看你心情看着给吧，启动！

4.先选择磁盘

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756734077448-59d38ae5-e97d-4bd1-83ef-df9f23423860.png)

然后安装完成之后，打开设备管理器。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756734727034-fc962c7b-d77c-45fa-b17e-62af9b1ba6d4.png)

浏览电脑以安装驱动

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756734755178-4a6af112-bf37-4f43-a60b-0cfcc0b89ab4.png)

选择这个安装

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1756734815556-be4171f5-90cd-4dbf-a5ff-948a652768d9.png)

然后正常联网即可

我装windows没啥用，无非就是运行todesk，这样方便我在网络不通畅的时候控制我的NAS。



# 十五、mariaDB的部署
1.伟大的茶佬的文档：[DB.01 MariaDB 部署](https://www.yuque.com/hongcha-6c6o6/klr1op/psgzxd8s9nsddhwz?singleDoc#)

2.大致步骤，也可以直接使用茶佬的模板[家庭数据中心教程汇总](https://www.yuque.com/hongcha-6c6o6/klr1op/xqao5sggi82efuet?singleDoc#)

```dockerfile
apt update
apt install curl apt-transport-https
mkdir -p /home/MariaDB
cd /home/MariaDB
curl -LsSO https://r.mariadb.com/downloads/mariadb_repo_setup
chmod +x mariadb_repo_setup
./mariadb_repo_setup
```

```dockerfile
apt update
apt-get install mariadb-server mariadb-client mariadb-backup
```

示例：

```dockerfile
#pgsql创建用户 给权限
CREATE USER "blinkoAdmin" WITH PASSWORD '123456';
CREATE DATABASE "blinko";
ALTER DATABASE "blinko" OWNER TO "blinkoAdmin";
GRANT ALL PRIVILEGES ON DATABASE "blinko" TO "blinkoAdmin";

#mariadb创建用户 给权限
CREATE USER 'iyuuAdmin'@'%' IDENTIFIED BY '123456'; 
CREATE DATABASE iyuu; 
GRANT ALL PRIVILEGES ON iyuu.* TO 'iyuuAdmin'@'%';
```

详见phpipam的教程

# 十六、postgresql的部署
1.一样的查看伟大茶佬的文档[DB.02 Postgresql 部署](https://www.yuque.com/hongcha-6c6o6/klr1op/ps4ugmwl7g9l9z1w?singleDoc#)

2.大致步骤，也可以直接使用茶佬的模板[家庭数据中心教程汇总](https://www.yuque.com/hongcha-6c6o6/klr1op/xqao5sggi82efuet?singleDoc#)

```dockerfile
sudo apt update
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```

```dockerfile
sudo apt update
#16 17 二选一
sudo apt -y install postgresql-16     #安装16
sudo apt -y install postgresql-17     #安装17
```

3.修改配置文件

```dockerfile
systemctl restart postgresql
su postgres
psql
#pgsql创建用户 给权限
CREATE USER "blinkoAdmin" WITH PASSWORD '123456';
CREATE DATABASE "blinko";
ALTER DATABASE "blinko" OWNER TO "blinkoAdmin";
GRANT ALL PRIVILEGES ON DATABASE "blinko" TO "blinkoAdmin";
\c embytoolkit;
GRANT ALL ON SCHEMA public TO "embytoolkit";
GRANT CREATE ON SCHEMA public TO "embytoolkit";
```

# 十七、phpipam地址管理与规划
伟大的茶佬的文档：[04.IP地址规划和管理](https://www.yuque.com/hongcha-6c6o6/phdu9c/knittmn6gs1f5usw?singleDoc#%20)

数据库操作：

```dockerfile
mariadb -u root -p
show databases;           					#查看数据库
SELECT user, host FROM mysql.user;	#查看用户
#以下是创建phpipam数据库、用户并赋权
CREATE DATABASE phpipam;
CREATE USER 'phpipam'@'%' IDENTIFIED BY 'phpipampassword';
GRANT ALL PRIVILEGES ON phpipam.* TO 'phpipam'@'%';
GRANT SELECT ON mysql.user TO 'phpipam'@'%';
FLUSH PRIVILEGES;
```

```dockerfile
version: '3'

services:
  phpipam-web:
    image: phpipam/phpipam-www:latest
    container_name: phpipam-www
    ports:
      - "80:80"                                             # phpipam网页登录端口
    environment:
      - TZ=Asia/Shanghai                                    # 改为亚洲上海时区
      - IPAM_DATABASE_USER=phpipam                          # 数据库用户
      - IPAM_DATABASE_HOST=192.168.4.20                     # 外部数据库IP地址
      - IPAM_DATABASE_PASS=phpipampassword                   # 使用你设置的安全密码
      - IPAM_DATABASE_NAME=phpipam                          # 数据库名称
      - IPAM_DATABASE_PORT=3306                             # 数据库端口
      - IPAM_DATABASE_WEBHOST=%                             # 允许任何主机连接
    restart: always
    volumes:
      - /mnt/appdata/phpipam/phpipam-logo:/phpipam/css/images/logo
      - /mnt/appdata/phpipam/phpipam-ca:/usr/local/share/ca-certificates:ro
    networks:
      - phpipam-network

  phpipam-cron:
    image: phpipam/phpipam-cron:latest
    container_name: phpipam-cron
    environment:
      - TZ=Asia/Shanghai                                    # 改为亚洲上海时区
      - IPAM_DATABASE_USER=phpipam                          # 数据库用户
      - IPAM_DATABASE_HOST=192.168.4.20                     # 外部数据库IP地址
      - IPAM_DATABASE_PASS=CYX19991229!@#                   # 使用你设置的安全密码
      - IPAM_DATABASE_NAME=phpipam                          # 数据库名称
      - IPAM_DATABASE_PORT=3306                             # 数据库端口
      - SCAN_INTERVAL=1h                                    # 扫描间隔
    restart: always
    volumes:
      - /mnt/appdata/phpipam-ca:/usr/local/share/ca-certificates:ro
    networks:
      - phpipam-network

# 注意：移除了 phpipam-mariadb 服务，使用外部数据mnt/appdata
networks:
  phpipam-network:
    driver: bridge

volumes:
  phpipam-logo:
  phpipam-ca:
```



# 十八、immich的安装与部署全流程包含postgresql17oci部署外挂
1.创建OCI的postgresql数据库

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771082151707-8a59c4a8-a4b3-4b3b-9520-a0fb0945d3c0.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771082483697-e9c84222-1287-4f0b-93c3-f78b6c6d0a71.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771082671270-80a8ca58-480b-4d82-81e8-67c74f020c85.png)

2.安装向量扩展插件

```dockerfile
apt update
apt install postgresql-17-pgvector
```

3.进入数据库进行一系列操作，全新的数据库需要先删除原模版创建新模板

```dockerfile
su postgres
psql
UPDATE pg_database SET datistemplate = false WHERE datname = 'template1';
DROP DATABASE template1;
CREATE DATABASE template1
  WITH TEMPLATE = template0
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.utf8'
  LC_CTYPE = 'C.utf8';
UPDATE pg_database SET datistemplate = true WHERE datname = 'template1';
UPDATE 1
DROP DATABASE
CREATE DATABASE
UPDATE 1
\l
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771083729029-7bc04ff4-2eec-477a-a14f-7a4780136b7b.png)

4.创建向量拓展

```dockerfile
CREATE EXTENSION vector;

SELECT * FROM pg_extension WHERE extname = 'vector';
```

5.创建用户与数据库

```dockerfile
CREATE USER "immich" WITH PASSWORD 'Password';
CREATE DATABASE "immich";
ALTER DATABASE "immich" OWNER TO "immich";
GRANT ALL PRIVILEGES ON DATABASE "immich" TO "immich";
#DBA是一个统管的角色，在坏库或者忘记用户名时可以使用DBA操作
CREATE USER "DBA" WITH PASSWORD 'Password';
GRANT ALL PRIVILEGES ON DATABASE "immich" TO "DBA";
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO "DBA";

CREATE EXTENSION IF NOT EXISTS vector CASCADE;
ALTER USER "immich" WITH SUPERUSER;
GRANT CREATE ON DATABASE "immich" TO "immich";
```

数据库层面操作到此结束

6.拉取镜像，顺便推送nfs到lxc

```dockerfile
docker pull ghcr.io/imagegenius/immich:latest
docker pull redis
```

7.茶佬的compose

```dockerfile
version: '3.8'
services:
#------------------------------------------Immich主服务部署安装--------------------------------------------#
  immich:                                                                  #Immich主服务部署
    image: ghcr.io/imagegenius/immich:latest
    container_name: immich
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/GMT-8
      - DB_HOSTNAME=postgres14                                             #ProstgreSql数据库服务器
      - DB_USERNAME=postgres                                               #ProstgreSql数据库用户名
      - DB_PASSWORD=PoStGrEs                                               #ProstgreSql数据库密码
      - DB_DATABASE_NAME=immich                                            #ProstgreSql数据库名
      - DB_PORT=5432                                                       #ProstgreSql数据库连接端口
      - REDIS_HOSTNAME=redis                                               #redis数据库服务器
      - REDIS_PORT=6379                                                    #redis数据库连接端口
      - REDIS_PASSWORD=                                                    #redis数据库密码，此处为空
      - MACHINE_LEARNING_HOST=0.0.0.0
      - MACHINE_LEARNING_PORT=3003
      - MACHINE_LEARNING_WORKERS=1
      - MACHINE_LEARNING_WORKER_TIMEOUT=120
    volumes:
      - /需要固化的路径/immich/config:/config                              #配置文件目录
      - /需要固化的路径/immich/photos:/photos                              #照片目录
      - /需要固化的路径/immich/libraries:/libraries
      - /mnt/nfs/photos:/mnt/photos:ro                                     #挂接的外部图库（只读访问）
    ports:
      - 8080:8080                                                          #Immich外部访问端口
      - 3003:3003                                                          #Immich机器学习访问端口
    devices:
      - /dev/dri:/dev/dri 			# 指定设备映射可是实现核显解码（需要本地支持）
    restart: unless-stopped
#---------------------------------------------Redis数据库安装----------------------------------------------#
  redis:                                                                   #redis数据库部署
    image: redis
    ports:
      - 6379:6379
    container_name: redis                                                  #需要被引用的redis数据库服务器
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/GMT-8
    volumes:
      - /需要固化的路径/immich/redis/data:/data                             #配置文件目录
#----------------------------------------PostgreSql数据库安装----------------------------------------------#
  postgres14:                                                               #PostgreSql部署
    image: tensorchord/pgvecto-rs:pg14-v0.2.0
    ports:
      - 5432:5432
    container_name: postgres14                                              #需要被引用的ProstgreSql数据库服务器
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/GMT-8
      - POSTGRES_USER=postgres                                              #ProstgreSql数据库用户名
      - POSTGRES_PASSWORD=PoStGrEs                                          #ProstgreSql数据库密码
      - POSTGRES_DB=immich                                                  #ProstgreSql数据库名
    volumes:
      - /需要固化的路径/immich/postgresql/data:/var/lib/postgresql/data     #配置文件及数据库文件目录
#--------------------------------------------可选安装------------------------------------------------------#
  pgadmin:                                                                  #PGadmin部署
    image: dpage/pgadmin4
    container_name: pgadmin4_container
    restart: always
    ports:
      - "8888:80"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/GMT-8
      - PGADMIN_DEFAULT_EMAIL=Blacktea@admin.com                            #默认登录用户名（可修改）
      - PGADMIN_DEFAULT_PASSWORD=Blacktea                                   #默认登录密码（可修改）
    volumes:
      - /需要固化的路径/immich/pgadmin/pgadmin-data:/var/lib/pgadmin        #配置文件目录
```

8.我的修改后的compose

```dockerfile
version: '3.8'
services:
#------------------------------------------Immich主服务部署安装--------------------------------------------#
  immich:                                                                  #Immich主服务部署
    image: ghcr.io/imagegenius/immich:latest
    container_name: immich
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/GMT-8
      - DB_HOSTNAME=192.168.4.19                                             #ProstgreSql数据库服务器
      - DB_USERNAME=immich                                               #ProstgreSql数据库用户名
      - DB_PASSWORD=Password                                               #ProstgreSql数据库密码
      - DB_DATABASE_NAME=immich                                            #ProstgreSql数据库名
      - DB_PORT=5432                                                       #ProstgreSql数据库连接端口
      - REDIS_HOSTNAME=redis                                               #redis数据库服务器
      - REDIS_PORT=6379                                                    #redis数据库连接端口
      - REDIS_PASSWORD=                                                    #redis数据库密码，此处为空
      - MACHINE_LEARNING_HOST=0.0.0.0
      - MACHINE_LEARNING_PORT=3003
      - MACHINE_LEARNING_WORKERS=1
      - MACHINE_LEARNING_WORKER_TIMEOUT=120
    volumes:
      - /mnt/appdata/immich/config:/config                              #配置文件目录
      - /mnt/appdata/immich/photos:/photos                              #照片目录
      - /mnt/appdata/immich/libraries:/libraries
      - /mnt/nfs/photos/immich-photo:/mnt/photos:ro                                     #挂接的外部图库（只读访问）
    ports:
      - 8080:8080                                                          #Immich外部访问端口
      - 3003:3003                                                          #Immich机器学习访问端口
    #devices:
     # - /dev/dri:/dev/dri 			# 指定设备映射可是实现核显解码（需要本地支持）
    restart: always
#---------------------------------------------Redis数据库安装----------------------------------------------#
  redis:                                                                   #redis数据库部署
    image: redis
    ports:
      - 6379:6379
    container_name: redis                                                  #需要被引用的redis数据库服务器
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/GMT-8
    volumes:
      - /mnt/appdata/immich/redis/data:/data                             #配置文件目录
```

9.登陆后可以修改ai模型，我将照片使用photosycn上传到外挂的目录，然后在系统管理中设置好外部资产管理

# 十九、HomeAssistant的部署以及调试
0.请自备魔法上网的条件

1.由于使用pve的虚拟机安装，所以我们使用这个页面中的<font style="color:rgb(34, 34, 34);background-color:rgb(251, 251, 251);">qcow2格式安装，请先下载该文件，然后解压。</font>

[Alternative](https://www.home-assistant.io/installation/alternative/)

2.将解压后的文件导入到片段中，然后记录下所在位置`<font style="color:rgb(0, 0, 0);">/var/lib/vz/import/haos_ova-16.3.qcow2</font>`<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1767091369198-6dd8459e-eecd-430f-9e42-93c0613b6021.png)

3.建立虚拟机

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1767094337078-41e176af-8bad-49e5-9dd4-de1727570ea3.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1767094353274-f3d0a2ad-eb15-401a-b569-43c69fbe0332.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1767094382723-a459ae3c-a213-40eb-88c0-c64c455aa5b1.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1767094427868-f410a2a1-2232-4a75-b451-76e306a9bd77.png)



<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1767094454358-4bc020f6-86c0-49d3-8483-09761bb68f9c.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1767094508814-d1bc807d-04db-4218-9eff-1cba3b866b69.png)

4.使用命令将刚才的文件推送给到虚拟机

```dockerfile
qm importdisk 907 /var/lib/vz/import/haos_ova-16.3.qcow2 vm-pool #pool表示目标存储位置
```

5.添加磁盘后确认一下，然后修改启动顺序，启动虚拟机。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1767094572057-a0783dcd-a9e0-45a1-ad98-c59b5292b3f9.png)

6.进入路由器查看ip地址，设为静态地址后，指定通过魔法联网，然后重启虚拟机，接着耐心等待即可。

7.首先解决反向代理的问题，搜索插件file editor，下载该插件，并访问文件夹configurational.yaml.

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1767098211774-d7408a6c-6b18-4be9-bcf6-ce05d97dc421.png)

粘贴一下内容进去

```dockerfile
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 192.168.4.0/23  #你的子网范围
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/55551868/1767098777304-e6c643ba-e7eb-468b-8990-9d220da8d015.png)

8.安装HACS

打开高级模式-完全重启整个系统，并且在插件项中安装`<font style="color:rgb(31, 31, 31);">Terminal & SSH</font>`<font style="color:rgb(31, 31, 31);">。接着运行指令</font>

```dockerfile
wget -O - https://get.hacs.xyz | bash -
```

回到页面按住shift+CTRL+V，接着再次重启整个系统。

9.安装小米home一气呵成，注意登录小米的账号后，要手动把homeassistant.local那一段改成你的IP地址。

# 二十、核显的直通（失败）
若要直通请查看茶佬教程[PVE 9.0 降内核、装i915、直通显卡](https://www.yuque.com/hongcha-6c6o6/klr1op/ccmil1ag3wkrqeun?singleDoc#)

[~~PVE通过i915驱动直通Intel核显~~](https://www.bilibili.com/read/cv39480503/?spm_id_from=333.1387.0.0&opus_fallback=1)

~~1.打开BIOS中的 VT-d 和SR-IOV技术~~

~~2.克隆项目仓库：~~

```dockerfile
apt-get update #更新软件列表
apt install git    #安装git克隆工具
mkdir -p /home/i915.dkms.driver
cd /home/i915.dkms.driver
uname -r
#确认pve版本，按照你的内核查询你需要的驱动版本
ls /boot/vmlinuz-*
#该命令可以查看你的所有内核版本
wget -O /home/i915.dkms.driver/i915-sriov-dkms_2025.07.22_amd64.deb "http://github.com/strongtz/i915-sriov-dkms/releases/download/2025.07.22/i915-sriov-dkms_2025.07.22_amd64.deb"
#根据你的内核版本克隆i915驱动
```

~~3.安装编译工具~~

```dockerfile
apt install build-* dkms
```

~~4.安装所需版本的内核和头文件~~

```dockerfile
#安装与内核对应的头文件，例如
wget http://mirrors.nju.edu.cn/proxmox/debian/dists/bookworm/pve-no-subscription/binary-amd64/proxmox-headers-6.8.12-17-pve_6.8.12-17_amd64.deb
wget http://mirrors.nju.edu.cn/proxmox/debian/dists/bookworm/pve-no-subscription/binary-amd64/proxmox-kernel-6.8.12-17-pve-signed_6.8.12-17_amd64.deb
dpkg -i proxmox-headers-6.8.12-17-pve_6.8.12-17_amd64.deb
#安装头文件
dpkg -i /home/i915.dkms.driver/i915-sriov-dkms_2025.07.22_amd64.deb
#安装驱动
```

~~5.检查~~

```dockerfile
dkms status
```

~~6.重新修改内核~~

```dockerfile
nano /etc/default/grub
#修改内容如下
GRUB_CMDLINE_LINUX_DEFAULT
#变更为
quiet intel_iommu=on iommu=pt i915.enable_guc=4 i915.max_vfs=7
intel_iommu=on iommu=pt i915.enable_guc=3 i915.max_vfs=7 module_blacklist=xe quiet
```

`~~quiet~~`~~：这个参数告诉内核在启动时不要显示太多的信息，即静默启动。这样在启动过程中就不会显示过多的内核消息，使得启动过程看起来更简洁。~~

`~~iommu=pt~~`~~：这个参数启用了Passthrough模式的IOMMU（输入输出内存管理单元）。IOMMU是一种硬件技术，用于改善虚拟化环境中设备直通的效率，允许虚拟机直接访问物理设备，从而提升性能。pt代表“passthrough”，即直通模式。~~

`~~intel_iommu=on~~`~~：这个参数专门针对Intel处理器，启用VT-d技术，即Intel的IOMMU技术。这允许虚拟机直接访问PCI设备，提高了虚拟化的性能和安全性。~~

`~~i915.enable_guc=4~~`~~：这个参数与Intel的集成显卡有关。i915是Intel集成显卡的内核驱动程序，enable_guc（Graphics Unit Control）设置为3通常意味着启用了GUC子系统，并且配置为使用所有可用的资源。GUC可以提高显卡的性能，特别是在虚拟化环境中。~~

`~~i915.max_vfs=7~~`~~：这个参数同样与Intel的集成显卡有关，它设置了最大虚拟函数（Virtual Functions）的数量。在虚拟化环境中，这可以允许多个虚拟机共享同一个物理显卡，每个虚拟机使用一个虚拟函数。~~

~~7.为了启用VFs，需要配置sysfs~~

```plain
update-grub
update-initramfs -u
```

```plain
apt install sysfsutils
```

~~然后修改sysfs.conf文件的配置信息~~

```plain
lspci | grep VGA
```

```plain
echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7" > /etc/sysfs.conf
```

~~8.抢救内核小技巧：~~

```plain
dpkg --get-selections |grep kernel                #查询内核
apt remove pve-坏内核                                #删除对应坏内核
apt install pve-kernel-6.8.12-2-pve               #安装对应版本内核
update-grub                                                  #更新引导
然后重启。。。
```

~~9.由于我的机器只有2.5G网口，导致我进行以上操作之后，机器根本起不来，那怎么办，是因为网卡没有进行更新而导致的~~

```dockerfile
mkdir -p /home/r8125 cd /home/r8125
```

```dockerfile
apt update
apt install -y pv  #安装必要的工具
wget -q --show-progress http://ftp.cn.debian.org/debian/pool/non-free/r/r8125/r8125-dkms_9.016.01-1_all.deb  #下载驱动包并显示进度条
dpkg -i r8125-dkms_9.016.01-1_all.deb    #安装驱动包
apt install -f # 自动修复依赖问题
此处建议update-grub一下
apt install --reinstall r8125-dkms   #重新安装驱动
dkms install r8125/9.016.01 -k 6.8.12-17-pve  #动态获取已安装的 r8125 模块版本[可不执行]
modprobe r8125  #加载驱动模块
reboot
```

# 二十一、自家iptv的抓包，制作直播源
前置条件：（我的网络环境）安装pve系统，必须有光猫超管密码，自己的小主机必须有起码两个网口，必须能够正常观看营业厅提供的iptv，电脑安装wireshark，能够安装immortalwrt，本教程仅针对于自己的江苏无锡电信进行教学，其他的可能更简单。

1.网口配置，由于我的小主机有四个网口，因此我规定enp1s0负责普通网络，连接虚交换机vmbr0，而enp4s0直连光猫的iptv接口，负责转播数据，如图

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771263719765-eeffa4d6-1755-4e7b-8f1b-e2e8c04b79a0.png)

2.安装immortalwrt，见前面的教程,记得添加两个vmbr

3.修改密码，网络-接口-删除wan和wan6接口-点击保存并应用

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771264411530-78c2683d-be69-4706-a5a7-b75bcf29b6e7.png)

4.lan口设置修改-填写你的网关，DNS服务器指向自己的主路由，不在此接口进行dhcp

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771264540847-d3257251-9760-4a62-9593-3e32cb1e578e.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771264569297-cd388ce7-f189-4d41-9ebb-1db490957ea1.png)

5.点击网络-防火墙-添加iptv

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771265036905-a56d5029-7a0d-47f0-9a81-909b47169c09.png)

6.新增iptv接口，静态ip，ipv4地址必须修改为光猫麾下的地址，网关指向光猫，取消使用默认网关并且设置度量值为100，防火墙选择刚刚创建的iptv，忽略dhcp

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771264649183-5bfa3262-3717-46f0-b377-abb508b76433.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771264679524-1034717b-806a-4757-9344-4af00199d377.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771264762262-1cb9249e-1a7b-4ca5-81cd-82e54ad7c489.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771265086421-854cae78-a6d5-4f2f-8639-6a636aed2e12.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771264904925-341df90b-ce89-4770-aab8-aa7f5bb399da.png)

7.点击接口-网络-设备，将eth1（iptv所在接口）mac地址强制修改为你的真实机顶盒地址，MTU为1500.，禁用ipv6（涉及真实IP地址，不带图）

8.更新软件包，搜索msd，下载第一个和第二个第三个

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771265416606-a471d194-acf2-4f65-b24a-947fe245077b.png)

9.直接启用组播转换即可，可以去互联网搜索江苏无锡电信的iptv源.（优先考虑能直接获取组播地址的同学，不能获取的往后面看）

例如cctv1源为`239.49.0.1:8000`，你的immortalwrt的地址为`192.168.4.27`，使用potplayer播放器添加链接，`http://192.168.4.27:7088/rtp/239.49.0.1:8000`，如果播放正常说明成功了。

此处附上文件[江苏无锡电信m3u格式源.txt](https://www.yuque.com/attachments/yuque/0/2026/txt/55551868/1771265740639-61a06bfa-eb68-4499-86df-8d1fefd6b0b5.txt)（下载完自己改成m3u后缀）[江苏无锡电信iptv源.txt](https://www.yuque.com/attachments/yuque/0/2026/txt/55551868/1771265688399-b21e7a53-aaf0-46f8-ab60-03b6a0932a55.txt)尝试将m3u源中的ip地址修改为自己的immortalwrt的地址后，将文件导入到iptv的播放器中，即可正常观看。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771265535778-60dbaca9-eb1b-443f-aec9-941ec155202e.png)

10.如何创建订阅链接呢，进入immortalwrt，安装控制终端ttyd

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771268149072-fdcde4ab-ecee-4f57-9041-c1e967928d98.png)

 在www的文件夹下创建1.m3u，然后以记事本格式打开m3u文件并粘贴保存即可。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771268216708-a6b89653-9b39-488f-bb65-0fce3fc40cb0.png)

接下来使用iptv软件访问订阅连接http://192.168.4.27/1.m3u即可。

11.反向代理的操作：（大致说明，请自行操作）

需要反向代理两个地址

一个是immortaliptv.domain.com对应immortalwrt的ip的80端口；

另一个是iptv.domain.com对应immortalwrt的ip的7088端口

将m3u文件中的内网地址修改为上述的immortaliptv.domain.com

<font style="background-color:#FBDE28;">12.若你无法获取到自己的地方的ip源，则需要考虑抓包，抓包教程如下所示</font>

<font style="background-color:#FBDE28;">工具：笔记本电脑，足量的网线，光猫超级管理员密码，wireshark</font>

<font style="background-color:#FBDE28;">1.首先将你的电脑连接光猫进入超级管理员后台，我的光猫有四个网口分别为2.5G口，iptv口，千兆3口与千兆4口，平时千兆3和4口为空闲网口，此时我的电脑插入千兆3口，点击诊断-业务诊断-镜像功能，如图，将iptv的流量镜像到千兆4口，点击保存</font>

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771266565734-7d893dd5-fa8f-4c7b-92f1-7b16c40b0986.png)

<font style="background-color:#FBDE28;">如图</font>

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771266657474-2c5df250-2fad-4aeb-8e64-b5b27338181d.png)

<font style="background-color:#FBDE28;">接下来将电脑的网线插入到千兆口4，打开wireshark准备抓包</font>

<font style="background-color:#FBDE28;">2.打开控制面板，网路和共享中心-更改适配器，查看你插入的千兆4口究竟是哪个网口，确认之后，将光猫的iptv口直接连接到运营商给到的机顶盒使得iptv能够顺利播放，将电视调整到cctv1电视台，wireshark将抓到如下的包</font>

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771267305265-a6154a80-f784-45a6-9764-c1624d0983e3.png)

<font style="background-color:#FBDE28;">将电视调整到cctv2电视台，wireshark将抓到如下的包</font>

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1771267352774-b83b37cd-a899-4ecb-b701-56be81c3f1aa.png)

<font style="background-color:#FBDE28;">你发现cctv1的地址为239.49.0.1:8000,而cctv2的地址为239.49.0.2:8008,记录下这些地址，一个个的调电视台就可以一个个的获取到每个电视台的地址了，将他们整理成如下的格式，</font>[20250703iptvrtp.txt](https://www.yuque.com/attachments/yuque/0/2026/txt/55551868/1771267626824-493206b8-5364-44e0-926a-f34cf1302ab1.txt)<font style="background-color:#FBDE28;">，最终你可以通过各种办法，调整成为上文可以使用的m3u格式</font>[江苏无锡电信m3u格式源 - 副本.txt](https://www.yuque.com/attachments/yuque/0/2026/txt/55551868/1771267774707-a0293d43-66dc-406e-b554-15f3b8810bbb.txt)<font style="background-color:#FBDE28;">（自行修改为m3u格式）,就可以正常使用了,最后记得回到光猫删除你的创建的镜像。</font>

# 第二十二、OCI.MariaDB的部署以及数据库迁移
1.伟大的茶佬教程，和之前lxc建立mariadb的数据库操作几乎一样，不讲，唯一的区别就是需要手动加两个变量[02. 容器OCI部署推荐——MariaDB](https://www.yuque.com/hongcha-6c6o6/klr1op/rkdk1n2661506mwt?singleDoc#)

2.重点讲一下数据库的迁移，当你新建了一个数据库之后，你会将之前的mariadb的数据库导入到新的数据库，首先确保你在这两个lxc上面挂载了相同的目录，方便迁移，我将nas的test目录挂载在了这两个容器的/mnt/nfs/test下面，参照我之前phpipam的部署所以命令如下

```dockerfile
mysqldump -u root -p --default-character-set=utf8mb4 --single-transaction --routines --triggers --events --databases phpipam > /mnt/nfs/test/phpipam_backup.sql
```

在新的数据库建立好用户以及数据库之后，可以使用以下命令

```dockerfile
mysql -u root -p --default-character-set=utf8mb4 < /mnt/nfs/test/phpipam_backup.sql
```

就ok了



# 第二十三、音乐系统的部署
1.没什么技术含量，直接看茶佬的文档就可以了，部署Music Assistant，Navidrome，music-tag-web

链接贴在这里：

[D-E-01.MusicAssistant](https://www.yuque.com/hongcha-6c6o6/klr1op/zd4wrqlgblqlwpdz?singleDoc#)

[S2.02.打造自己的专属音乐中心](https://www.yuque.com/hongcha-6c6o6/klr1op/57b7bcc6f54f484ba4eb02dbfeaad06a?singleDoc#)

[D-E-02.AzuraCast](https://www.yuque.com/hongcha-6c6o6/klr1op/uf3w5un0nhe62638?singleDoc#)



# 第二十四、节点搭建
购买地址：

[RakSmart 全球服务器租用-云服务器｜裸机云｜独立服务器｜站群服务器](https://cn.raksmart.com/)

例如你购买了一个`1.1.1.1`的服务器，密码为`password`,端口号为`22`

那么接下来一步步搭建节点，参考教程[【零基础】2026最新保姆级纯小白节点搭建教程，人人都能学会，目前最简单、最安全、最稳定的专属节点搭建方法，手把手自建节点搭建教学，晚高峰高速稳定，4K秒开的科学上网线路体验 - 科学上网 技术分享](https://bulianglin.com/archives/nicename.html)

1.windows启用cmd，输入以下命令

```bash
ssh root@1.1.1.1 -p 22   #登录并输入密码进入vps
VERSION=1.2.2 && bash <(curl -Ls https://raw.githubusercontent.com/alireza0/s-ui/$VERSION/install.sh) $VERSION   #安装s-ui面板
#静候几分钟，[y/n]回车一下
```

2.你会看到类似的以下信息：s-ui的面板的账号密码以及登录的地址，记录下来

3.接下来，复制下面的命令

```bash
for d in lpcdn.lpsnmedia.net www.sony.com mscom.demdex.net cdn.bizible.com intelcorp.scene7.com cdn.userway.org github.gallerycdn.vsassets.io s.mp.marsflag.com ts3.tc.mm.bing.net www.bing.com ; do t1=$(date +%s%3N); timeout 1 openssl s_client -connect $d:443 -servername $d </dev/null &>/dev/null && t2=$(date +%s%3N) && echo "$d: $((t2 - t1)) ms" || echo "$d: timeout"; done
for d in iosapps.itunes.apple.com static.cloud.coveo.com github.gallerycdn.vsassets.io gsp-ssl.ls.apple.com aadcdn.msftauth.net c.s-microsoft.com api.company-target.com ms-vscode.gallerycdn.vsassets.io downloadmirror.intel.com logx.optimizely.com ; do t1=$(date +%s%3N); timeout 1 openssl s_client -connect $d:443 -servername $d </dev/null &>/dev/null && t2=$(date +%s%3N) && echo "$d: $((t2 - t1)) ms" || echo "$d: timeout"; done
for d in img-prod-cms-rt-microsoft-com.akamaized.net j.6sc.co www.xilinx.com www.bing.com beacon.gtv-pub.com amd.com ms-python.gallerycdn.vsassets.io www.sony.com d2c.aws.amazon.com consent.trustarc.com ; do t1=$(date +%s%3N); timeout 1 openssl s_client -connect $d:443 -servername $d </dev/null &>/dev/null && t2=$(date +%s%3N) && echo "$d: $((t2 - t1)) ms" || echo "$d: timeout"; done
for d in downloadmirror.intel.com c.6sc.co devblogs.microsoft.com downloaddispatch.itunes.apple.com ts1.tc.mm.bing.net electronics.sony.com snap.licdn.com catalog.gamepass.com drivers.amd.com intelcorp.scene7.com ; do t1=$(date +%s%3N); timeout 1 openssl s_client -connect $d:443 -servername $d </dev/null &>/dev/null && t2=$(date +%s%3N) && echo "$d: $((t2 - t1)) ms" || echo "$d: timeout"; done
for d in ms-vscode.gallerycdn.vsassets.io beacon.gtv-pub.com iosapps.itunes.apple.com aws.amazon.com downloadmirror.intel.com assets-www.xbox.com acctcdn.msftauth.net prod.log.shortbread.aws.dev assets-xbxweb.xbox.com www.intel.com ; do t1=$(date +%s%3N); timeout 1 openssl s_client -connect $d:443 -servername $d </dev/null &>/dev/null && t2=$(date +%s%3N) && echo "$d: $((t2 - t1)) ms" || echo "$d: timeout"; done
for d in statici.icloud.com r.bing.com static.cloud.coveo.com devblogs.microsoft.com intel.com apps.apple.com gray-wowt-prod.gtv-cdn.com gsp-ssl.ls.apple.com tags.tiqcdn.com d.oracleinfinity.io ; do t1=$(date +%s%3N); timeout 1 openssl s_client -connect $d:443 -servername $d </dev/null &>/dev/null && t2=$(date +%s%3N) && echo "$d: $((t2 - t1)) ms" || echo "$d: timeout"; done
for d in consent.trustarc.com is1-ssl.mzstatic.com amd.com tag.demandbase.com iosapps.itunes.apple.com apps.mzstatic.com drivers.amd.com s.company-target.com lpcdn.lpsnmedia.net www.sony.com ; do t1=$(date +%s%3N); timeout 1 openssl s_client -connect $d:443 -servername $d </dev/null &>/dev/null && t2=$(date +%s%3N) && echo "$d: $((t2 - t1)) ms" || echo "$d: timeout"; done
for d in images.nvidia.com t0.m.awsstatic.com mscom.demdex.net d.impactradius-event.com assets.adobedtm.com j.6sc.co b.6sc.co r.bing.com prod.us-east-1.ui.gcr-chat.marketing.aws.dev fpinit.itunes.apple.com ; do t1=$(date +%s%3N); timeout 1 openssl s_client -connect $d:443 -servername $d </dev/null &>/dev/null && t2=$(date +%s%3N) && echo "$d: $((t2 - t1)) ms" || echo "$d: timeout"; done
```

选择延时最低的域名记录下来

4.关闭cmd命令提示符然后重新开启，执行以下命令，意思是访问你计算机的本地2095端口相当于访问你vps的2095端口，接下来一直保持你的cmd命令框最小化不要关闭

```bash
ssh -L 2095:127.0.0.1:2095 root@1.1.1.1 -p 22
```

5.访问你的计算机`http://127.0.0.1:2095/app`，输入刚才记录的面板账户密码

点击tls设置，添加两个，配置如下，域名可以换成你的最小延时的域名

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1777304792409-a9a94c88-8406-45ae-9801-1e2d8f8abca6.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1777304809421-bfaabca0-26bb-4ed8-a5e4-ed4e242f0e70.png)

6.进入入站管理，添加两个

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1777304859455-febcae17-ecbd-4c6d-b99b-edd829e5ffc1.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/55551868/1777304874317-ac9aa352-ac8f-4b5f-adc4-910a1054fe26.png)

7.进入用户管理，新建用户，你将获得两个二维码，导入使用即可，记得ip地址改成vps



