**centos6.3上 gitlab4.0安装指南。**
在RHEL 6.3 上也进行了安装测试，到目前发现并记录了一些不同。

阅读 `doc/install/requirements.md` 了解安装gitlab4.0时硬件和平台环境的要求。

## 概述 ##
本文主要介绍在一个全新的操作系统上安装基于MySQL数据库的 gitlab4.0。

**提示：**
本文的安装步骤测试能够正常执行.
也可以不按照本文的步骤进行安装，但是要遵照gitlab4.0约定的运行环境进行安装(如果你了解的话)。
在AWS上安装或者其他web服务器的配置等请参考"高级技巧" 章节.

**提示：**
如果发现本文的错误还请提交给给我们

**提示：**
绝大多数情况下，你需要用系统root用户执行命令。
需要的时候你需要切换到'git' 或者 'gitlab' 用户上执行命令，

从root用户切换到其它用户用下面的命令，如：

    su - gitlab

**提示：**
很多Linux上的软件安装教程简单的规定: "禁止 selinux 和 防火墙".
ubuntu上的原生 gitlab安装需要完全禁止 StrictHostKeyChecking.
而本安装教程不需要禁止任何安全项，我们只需要按照安全策略简单的进行配置.

- - -

# 安装概要

 GitLab的安装主要有下列组件的安装构成：

1. 安装操作系统 (如CentOS 6.3 Minimal) 和依赖库（ Packages / Dependencies）
2. 安装Ruby
3. 创建系统用户
4. 安装Gitolite
5. 安装GitLab


----------

# 1. 安装操作系统 (CentOS 6.3 Minimal)

首先需要下载一个全新的 CentOS 6.3 "minimal" 系统。 如果知识测试安装可以下载centos的ISO文件用虚拟机安装

1. centos6.3下载地址：http://mirrors.163.com/centos/
2. VirtualBox下载：https://www.virtualbox.org/wiki/Downloads

## 增加和更新基本的软件和服务
### 增加 EPEL repository

*登陆到root账号*

    rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm

### 安装 gitlab 和 gitolite 需要的工具

*登陆到root账号*

    yum -y groupinstall 'Development Tools'

    ### 'Additional Development'
    yum -y install vim-enhanced httpd readline readline-devel ncurses-devel gdbm-devel glibc-devel \
                   tcl-devel openssl-devel curl-devel expat-devel db4-devel byacc \
                   sqlite-devel gcc-c++ libyaml libyaml-devel libffi libffi-devel \
                   libxml2 libxml2-devel libxslt libxslt-devel libicu libicu-devel \
                   system-config-firewall-tui python-devel redis sudo mysql-server wget \
                   mysql-devel crontabs logwatch logrotate sendmail-cf qtwebkit qtwebkit-devel \
                   perl-Time-HiRes

 

### 更新 CentOS 到最新

*登陆到root账号*

    yum -y update

## 配置 redis
确保redis在下次重启系统时可以自动运行

*登陆到root账号*

    chkconfig redis on
    service redis start

## 配置 mysql
确保MySQL在下次重启系统时可以自动运行。

*登陆到root账号*

    chkconfig mysqld on
    service mysqld start

配置 MySQL ， 设置MySQL root账号的密码，根据提示一路"Yes" 

    /usr/bin/mysql_secure_installation

## 配置 httpd

我们用 Apache 作为gitlab的前端
确保Apache在下次重启系统时可以自动运行。

    chkconfig httpd on

创建文件 **/etc/httpd/conf.d/gitlab.conf** ，并且增加下面的内容(替换 git.example.org 为你的域名!!). 

    <VirtualHost *:80>
      ServerName git.example.org
      ProxyRequests Off
        <Proxy *>
           Order deny,allow
           Allow from all
        </Proxy>
        ProxyPreserveHost On
        ProxyPass / http://localhost:3000/
        ProxyPassReverse / http://localhost:3000/
    </VirtualHost>

注: 如果还有其他web站点在同一台服务器上，你需要在 **/etc/httpd/conf/httpd.conf** 如下配置

    NameVirtualHost *:80

还要配置 selinux 

    setsebool -P httpd_can_network_connect on

## 配置防火墙

   修改 **/etc/sysconfig/iptables** 增加如下内容

    # Firewall configuration written by system-config-firewall
    # Manual customization of this file is not recommended.
    *filter
    :INPUT ACCEPT [0:0]
    :FORWARD ACCEPT [0:0]
    :OUTPUT ACCEPT [0:0]
    -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
    -A INPUT -p icmp -j ACCEPT
    -A INPUT -i lo -j ACCEPT
    -A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
    -A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT
    -A INPUT -m state --state NEW -m tcp -p tcp --dport 443 -j ACCEPT
    -A INPUT -j REJECT --reject-with icmp-host-prohibited
    -A FORWARD -j REJECT --reject-with icmp-host-prohibited
    COMMIT

## 配置 email

    cd /etc/mail
    vim /etc/mail/sendmail.mc

增加一行 smtp gateway hostname

    define(`SMART_HOST', `smtp.example.com')dnl

注释掉下面这行

    EXPOSED_USER(`root')dnl

至于奥在这行前面增加 'dnl ' 如：

    dnl EXPOSED_USER(`root')dnl
 
启用这个配置

    make
    chkconfig sendmail on


## 重启系统

    reboot

----------

# 2. 安装Ruby
下载和编译:

*登陆到root账号*

    mkdir /tmp/ruby && cd /tmp/ruby
    wget http://ftp.ruby-lang.org/pub/ruby/1.9/ruby-1.9.3-p327.tar.gz
    tar xfvz ruby-1.9.3-p327.tar.gz
    cd ruby-1.9.3-p327
    ./configure
    make
    make install

安装 Bundler Gem:

*登陆到root账号*

    gem install bundler

----------

# 3. 增加系统用户

## 为 Git and Gitolite 增加用户
*登陆到root账号*

    adduser \
      --system \
      --shell /bin/bash \
      --comment 'Git Version Control' \
      --create-home \
      --home-dir /home/git \
      git

    adduser \
      --shell /bin/bash \
      --comment 'GitLab user' \
      --create-home \
      --home-dir /home/gitlab \
      gitlab

    usermod -a -G git gitlab 

稍后gitlab这个用户需要用到密码，我们现在给gitlab设置一个登陆密码
    passwd gitlab # please choose a good password :)  

*登陆到root账号*

    # 生成 SSH key
    sudo -u gitlab -H ssh-keygen -q -N '' -t rsa -f /home/gitlab/.ssh/id_rsa

 
    
----------

# 4. 安装Gitolite

## 获取 GitLab 的源码:

*登陆到root账号*

    cd /home/git
    sudo -u git -H git clone -b gl-v320 https://github.com/gitlabhq/gitolite.git /home/git/gitolite

## 设置GitLab为 Gitolite 的管理员:


*登陆到root账号*

    # 添加 Gitolite 脚本到环境变量
    sudo -u git -H mkdir /home/git/bin
    sudo -u git -H sh -c 'printf "%b\n%b\n" "PATH=\$PATH:/home/git/bin" "export PATH" >> /home/git/.profile'
    sudo -u git -H sh -c 'gitolite/install -ln /home/git/bin'

    # 复制 gitlab用户的公钥（ SSH key） ...
    cp /home/gitlab/.ssh/id_rsa.pub /home/git/gitlab.pub
    chmod 0444 /home/git/gitlab.pub

    # ...用和上面的公钥作为安装Gitolite的管理员公钥
    sudo -u git -H sh -c "PATH=/home/git/bin:$PATH; gitolite setup -pk /home/git/gitlab.pub"

### 确包配置文件的所属权限:

    # 确保git用户有Gitolite配置目录的所有权限
    chmod 750 /home/git/.gitolite/
    chown -R git:git /home/git/.gitolite/

### 设置repositories目录权限:

    # Make sure the repositories dir is owned by git and it stays that way
    chmod -R ug+rwXs,o-rwx /home/git/repositories/
    chown -R git:git /home/git/repositories/

    # Make sure the gitlab user can access the required directories
    chmod g+x /home/git

### 配置git客户端，增加gitlab里的账户

*登陆到root账号*

    su - gitlab

*登陆到用户**gitlab***    
    
    ssh git@localhost  # type 'yes' and press <Enter>.

顺利的话你会获得类似下面的结果，并且连接关闭:

    PTY allocation request failed on channel 0
    hello gitlab, this is git@gitlab running gitolite3 v3.2-gitlab-patched-0-g2d29cf7 on git 1.7.1

## 测试工作是否正常
*登陆到用户 **gitlab***

    # 克隆用户 admin 的 仓库  ...
    # ... 并且确认用户是否有权访问 Gitolite
    git clone git@localhost:gitolite-admin.git /tmp/gitolite-admin

    # 如果没有任何问题你可以删除刚才克隆的仓库
    rm -rf /tmp/gitolite-admin

**提示：**
如果上面检测没有成功 : **先暂时停止安装步骤**!
检查 [问题列表](https://github.com/gitlabhq/gitlab-public-wiki/wiki/Trouble-Shooting-Guide)
并确保上面的步骤都正确执行.

----------
# 5. 安装GitLab

*登陆到用户 gitlab*

    #我们将把 GitLab 安装到用户 "gitlab" 的家目录
    cd /home/gitlab

## 克隆Gitlab源码

    # 克隆Gitlab源码
    git clone https://github.com/gitlabhq/gitlabhq.git gitlab

    # 进入到 gitlab 目录 
    cd /home/gitlab/gitlab
   
    # 检出稳定版
    git checkout 4-0-stable

**提示：**
如果你想体验开发版本，你可以吧 `4-0-stable` 改为 `master` 不建议这样做，会遇到很多麻烦!

## 配置Gitlab

复制GitLab 的配置实例

    cp /home/gitlab/gitlab/config/gitlab.yml{.example,}

把配置文件中的 "localhost"改为你自己的域名. 顺带检测下其他配置.

    vim /home/gitlab/gitlab/config/gitlab.yml

复制 Unicorn 的配置实例
    cp /home/gitlab/gitlab/config/unicorn.rb{.example,}

编辑unicorn 配置

    vim /home/gitlab/gitlab/config/unicorn.rb

在最下面增加一行:

    listen "127.0.0.1:3000"  # listen to port 3000 on the loopback interface

 
## Gitlab的数据库配置

    # MySQL
    cp /home/gitlab/gitlab/config/database.yml{.mysql,}

编辑数据库配置，填写正确的数据库用户名密码等

    vim /home/gitlab/gitlab/config/database.yml

数据库配置大致如下:

    production:
      adapter: mysql2
      encoding: utf8
      reconnect: false
      database: gitlabhq_production
      pool: 5
      username: 数据库用户名
      password: 数据库密码
      # host: localhost
      # socket: /tmp/mysql.sock
    
## 安装 Gems

*登陆到用户 **root***

    cd /home/gitlab/gitlab

    gem install charlock_holmes --version '0.6.9'

    su - gitlab

*登陆到用户 **gitlab***

    cd /home/gitlab/gitlab

    # For mysql db
    bundle install --deployment --without development test postgres

## 配置 Git

 Git客户端需要用户名和邮箱 。. (账号配置见`config/gitlab.yml`)

*登陆到用户gitlab*

    git config --global user.name "GitLab"
    git config --global user.email "gitlab@localhost"

## 设置 GitLab 钩子
 

*登陆到用户 **root***

    cd /home/gitlab/gitlab
    cp ./lib/hooks/post-receive /home/git/.gitolite/hooks/common/post-receive
    chown git:git /home/git/.gitolite/hooks/common/post-receive

## 初始化数据库 和激活高级功能

*登陆到用户 **gitlab***

    su - gitlab

 
    cd /home/gitlab/gitlab
    bundle exec rake gitlab:app:setup RAILS_ENV=production


## 安装启动脚本

下载启动脚本
 
 
*登陆到用户 root*

    curl https://raw.github.com/gitlabhq/gitlab-recipes/4-0-stable/init.d/gitlab-centos > /etc/init.d/gitlab
    chmod +x /etc/init.d/gitlab
    chkconfig --add gitlab
    chkconfig gitlab on

启动gitlab服务:

    service gitlab start
    # 或者
    /etc/init.d/gitlab start

## 检测服务状态
 

*登陆到用户 **gitlab***

    su - gitlab

    cd /home/gitlab/gitlab
    bundle exec rake gitlab:env:info RAILS_ENV=production


    cd /home/gitlab/gitlab
    bundle exec rake gitlab:check RAILS_ENV=production

如果你看到结果都是绿色显示的， 那么恭喜你，你成功的安装了Gitlab!


现在访问你的域名，体验下属于你自己的Gitlab吧！
Gitlab默认的管理员账号是
    admin@local.host
    5iveL!fe

 

