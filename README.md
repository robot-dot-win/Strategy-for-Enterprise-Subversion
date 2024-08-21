## SCENARIO

- Apache Subversion - "Enterprise-class centralized version control for the masses."
- A large-scale organization may have some hundreds of teams, each of which may have a changing number of members and may have been developing some dozens of projects in some years. One member may participate in several projects.
- They have hierarchical administrative roles for safe and efficient management of projects and teams.
- They usually need to meet the requirements of ISO/IEC 27001 standard for information security management. At least, all information systems should comply with the AAA security model.

## FEATURES

- Centralized large-scale multiple repositories
- Hierarchical administrative roles as needed:
  - System Administrator(OS user)
    - Maintain server system and software environment.
    - Maintain repositories(create/config/backup/verify/etc.).
    - Initialize the first SVN administrator.
  - SVN Administrator
    - Manage SVN administrators and auditors.
    - Manage repository administrators and auditors.
  - Repository Administrator
    - Manage repository authorization.
  - System Auditor(OS user)
  - SVN Auditor
  - Repository Auditor
- No-code implementation and all based on Subversion features
- Easy to meet the security management requirements

## STRATEGIES

### Repository Organization
Every team has its own repository containing all its projects. For example, one team's repository might look like this:
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
For a new team, a system administrator will create a new repository, and an SVN administrator will set the repository administrators, one of whom will set the Authz files.

### Repository Hosting
Nowadays with the help of cloud computing, we nearly don't have to consider the physical location, networking, computility, storage, backup, etc. of the Subversion server.
Just set up a cloud server, hosting all the repositories.

### Access Control
Since Subversion version 1.8.0 there came a desirable feature called ["In-Repo Authz"](https://cwiki.apache.org/confluence/display/SVN/In-Repo+Authz), which allows us to store Authz files in a Subversion repository.
Thus we can gain versioning and audit trail of the Authz files.

Now we create a dedicated repository called Authz-repo, in which we create subdirectories beneath the root, each corresponding to one team's repository.
The layout might look like this:
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

- `admin` - Name of the Authz-repo
- `teama` - Name of the repository of Team A
- `teamb` - Name of the repository of Team B
- `authz.txt`  - Authorization definition file for the corresponding repository
- `groups.txt` - Group definition file for the corresponding repository

By setting different access rules of these subdirectories, we can clearly and easily define the manager roles. For example, users who have `rw` access to `/admin` are SVN administrators,
and users who have `rw` access to `/teamb` are repository administrators of `teamb`. See `templets` for more details.

### Extension
We can create another dedicated repository called Command-repo. By writing hooks we can implement system administrators'/auditors' tasks, such as creating new repositories, viewing logs, etc..
Users write commands and parameters in a "command file" and commit it to the Command-repo, then the hooks run, committing results into the Command-repo, which can be pulled down via `svn update`.

## EXAMPLE
The example included in this repository is based on the `svnserve` server, running on CentOS Stream 9. It uses SASL to authenticate users against an SMTP server via the Linux PAM module
[`pam_smtp`](https://github.com/robot-dot-win/pam_smtp). Note that because the `svnserve` protocol is not encrypted, a secure tunnel(e.g VPN) should have been established if users access the server through
the wide-open Internet.

The following steps should be done by a System Administrator:

1. Install:
```bash
[root@localhost ~]# dnf install svn cyrus-sasl cyrus-sasl-plain
```

2. Create the Authz-repo:
```bash
[root@localhost ~]# mkdir /var/svn
[root@localhost ~]# svnadmin create /var/svn/admin
```

3. Check out and put the following configuration files:
- `/etc/sysconfig/svnserve` - Server options of `svnserve`
- `/etc/sysconfig/saslauthd` - Server options of `saslauthd`
- `/etc/sasl2/svn.conf` - SASL configuration used by `svnserve`
- `/etc/pam.d/svn` - PAM configuration used by `saslauthd`
- `/var/svn/admin/conf/svnserve.conf` - Authz-repo configuration

4. Start the services:
```bash
[root@localhost ~]# systemctl enable --now saslauthd
[root@localhost ~]# systemctl enable --now svnserve
```

5. Import initialization authz files to the Authz-repo:
Check out the two templet files `authz.txt.tmpl` and `groups.txt.tmpl`, specifying at least one SVN Administrator, then put the two `.txt` files into a new directory, e.g. `/usr/tmp/admin`.
```bash
[root@localhost ~]# svn import -m "Initialize admin authz files" /usr/tmp/admin file:///var/svn/admin/admin
```

6. Create all Team-repo and check out their configuration files `svnserve.conf`:
```bash
[root@localhost ~]# svnadmin create /var/svn/teama
[root@localhost ~]# svnadmin create /var/svn/teamb
[root@localhost ~]# systemctl restart svnserve
```

Now with a Subversion client an SVN Administrator can proceed to all his jobs against `svn://svn.foobar.com/admin/admin/`, and also, a Repository Administrator of `teama` can finish his jobs
against `svn://svn.foobar.com/admin/teama/`.

## LICENSE
This repository is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).
