## 应用场景

- Apache Subversion官网将其定义为“面向大规模用户的企业级集中式版本控制系统”。
- 一个大型的组织机构可能有上百个项目组，各项目组在几年内可能完成几十个项目；项目组成员可能动态调整，每个成员可能参与多个项目。
- 基于安全性和管理效率，大型组织机构对项目组和项目通常实行分级管理模式。
- 信息安全管理通常应当符合ISO/IEC 27001规范要求，至少来说，所有信息系统都应符合AAA安全模式。

## 方案特点

- 集中式大规模多仓库
- 分级的管理员角色：
  - 系统管理员(由操作系统用户担任)
    - 维护服务器及软件系统环境
    - 维护仓库(创建/配置/备份/校验等)
    - 初始化第一个SVN系统管理员
  - SVN系统管理员
    - 管理SVN系统管理员和SVN审计员
    - 管理项目仓库的管理员和审计员
  - 仓库管理员
    - 管理项目仓库的用户授权
  - 系统审计员(由操作系统用户担任)
  - SVN审计员
  - 仓库审计员
- 完全基于SVN的特性，零代码实现
- 极易满足安全管理规范要求

## 部署策略

### 仓库组织
为每个项目组设立一个仓库，保存该项目组的所有项目。例如，项目组A的仓库目录结构可能是这样的：
```
/
    project1/
        trunk/
        tags/
        branches/
    project2/
        trunk/
        tags/
        branches/
    ...
```
一个新项目组成立，由系统管理员为其创建一个新仓库，由SVN系统管理员设置该仓库的管理员和审计员，
然后，仓库管理员即可管理该仓库的用户授权。

### 服务器规划
随着云计算技术的发展，系统管理员极少需要考虑服务器的物理位置、网络设置、存储容量、计算能力、
数据备份等问题，通常只需要建立一台云服务器来管理所有仓库。

### 访问控制
SVN从1.8.0版开始提供了一个极佳的特性称为["库内授权"](https://cwiki.apache.org/confluence/display/SVN/In-Repo+Authz)，允许将授权管理文件保存在仓库里面。
这样，就可以通过这些文件的版本变化来实现审计追踪。

这个方案策略的核心，就是单独建立一个授权管理仓库，这个仓库根目录下的一级子目录，分别对应着各个仓库的名字（包括授权管理仓库和所有项目组仓库）。
授权管理仓库的目录结构应该是这样的：
```
/
    admin/
        authz.txt
        groups.txt
    teama/
        authz.txt
        groups.txt
    teamb/
        authz.txt
        groups.txt
    ...
```
其中，
- `admin` - 授权管理仓库的名字
- `teama` - 项目组A仓库的名字
- `teamb` - 项目组B仓库的名字
- `authz.txt`  - 相应仓库的授权管理文件
- `groups.txt` - 相应仓库的组定义文件

通过对这些子目录进行授权管理，就可以简单而清晰地定义各级管理角色。例如，
对`/admin`具有`rw`权限的用户就是SVN系统管理员，
对`/teamb`具有`rw`权限的用户就是项目组B仓库的仓库管理员，依此类推。
具体请参见`templets`下的模板文件。

### 扩展考虑
通过灵活的钩子脚本，可以实现形形色色的功能。例如，可以考虑再创建一个单独的命令仓库，
通过钩子来完成系统管理员/审计员的工作，如创建新仓库、检索日志等等。思路是：
将命令及参数写在一个约定好的“命令文件”中，将其提交到命令仓库，从而触发钩子脚本执行相应的命令，
并将命令结果提交到仓库，这样，从SVN客户端用`svn update`命令即可取得运行结果。

## 示例模板
本仓库所提供的模板和示例是基于`svnserve`服务实现的，在CentOS Stream 9中运行。
利用一个Linux PAM模块[`pam_smtp`](https://github.com/robot-dot-win/pam_smtp)，
使用SASL通过一个SMTP服务来实现用户认证。应特别注意：由于`svnserve`协议是非加密的，
如果要通过互联网进行连接，则首先应通过某种方式（例如VPN）建立起加密通道。

下列步骤需要系统管理员来操作：

1. 安装：
```bash
[root@localhost ~]# dnf install svn cyrus-sasl cyrus-sasl-plain
```

2. 创建授权管理仓库：
```bash
[root@localhost ~]# mkdir /var/svn
[root@localhost ~]# svnadmin create /var/svn/admin
```

3. 检查和设置如下配置文件：
- `/etc/sysconfig/svnserve` - `svnserve`服务参数选项
- `/etc/sysconfig/saslauthd` - `saslauthd`服务参数选项
- `/etc/sasl2/svn.conf` - `svnserve`使用的SASL配置
- `/etc/pam.d/svn` - `saslauthd`使用的PAM配置
- `/var/svn/admin/conf/svnserve.conf` - 授权管理仓库配置文件

4. 启动服务：
```bash
[root@localhost ~]# systemctl enable --now saslauthd
[root@localhost ~]# systemctl enable --now svnserve
```

5. 把初始化授权管理文件导入授权管理仓库：
检查两个模板文件`authz.txt.tmpl`和`groups.txt.tmpl`，设置至少1个SVN系统管理员，生成两个`.txt`文件，
放到一个新目录中，例如`/usr/tmp/admin`。然后执行
```bash
[root@localhost ~]# svn import -m "Initialize admin authz files" /usr/tmp/admin file:///var/svn/admin/admin
```

6. 创建项目组仓库，检查并设置好它们的配置文件`svnserve.conf`：
```bash
[root@localhost ~]# svnadmin create /var/svn/teama
[root@localhost ~]# svnadmin create /var/svn/teamb
[root@localhost ~]# systemctl restart svnserve
```

此后，借助于SVN客户端，SVN系统管理员通过操作`svn://svn.foobar.com/admin/admin/`就可以完成所有管理工作，
同样，`teama`的仓库管理员可以通过操作`svn://svn.foobar.com/admin/teama/`来完成其管理工作。

## 授权许可
本仓库遵循[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/deed.zh-hans)授权许可条款。
