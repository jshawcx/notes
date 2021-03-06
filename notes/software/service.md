### 阿里云安装服务

- 修改ssh端口&添加防火墙端口
    - vim /etc/ssh/sshd_config 修改如下
    ```
        #Port 22
        Port 61209
        AddressFamily any
        ListenAddress 0.0.0.0
        ListenAddress ::
    ```
    - semanage port -l|grep ssh 查看SELinux开放给ssh的端口（如果关闭selinux 忽略）
    - semanage port -a -t ssh_port_t -p tcp 61209 添加端口（如果关闭selinux 忽略）
    - firewall-cmd --permanent --query-port=61209/tcp --zone=public 或
        firewall-cmd --permanent --list-port --zone=public 查看端口启用情况
    - firewall-cmd --zone=public --permanent --add-port=61209/tcp --add-port=80/tcp --add-port=443/tcp
    -  firewall-cmd --reload 重启防火墙
    - 重启ssh&防火墙&默认启用&服务器（可以不必） systemctl restart sshd ，systemctl restart firewalld，systemctl is-enabled firewalld， systemctl enable firewalld，shutdown -r now
    - 阿里云安全组规则中添加以上规则

- ssh免登录
    - 本地机执行 ssh-keygen -t rsa 生成密钥，直接三次回车
    - 复制本地.ssh/id_rsa.pub文件内容到远程机上的.ssh/authorized_keys
    - 编辑本地 .bashrc 添加alias alias z-ssh="ssh root@111.100.111.111 -p61209"，source .bashrc

- 安装web服务器
    - 添加group&user
    ```
    groupadd www
    useradd -g www  -M -s /sbin/nologin www
    ```
    - 安装openresty https://openresty.org/en/linux-packages.html#centos
    ```
    wget https://openresty.org/package/centos/openresty.repo
    mv openresty.repo /etc/yum.repos.d/
    yum check-update
    yum --disablerepo="*" --enablerepo="openresty" list available（列出所有的包）
    yum install openresty
    yum install openresty-resty(resty命令行，非必选)
    systemctl enable --now openresty (开机启动，并立即激活服务)
    ```
    - 安装php7.4&配置php-fpm
    ```
    wget http://rpms.remirepo.net/enterprise/remi-release-8.rpm
    yum install remi-release-8.rpm
    yum check-update
    yum --disablerepo="*" --enablerepo=remi list available
    yum install gd environment-modules scl-utils httpd-filesystem ( 非必选)
    yum  --disablerepo=* --enablerepo=remi --downloadonly --downloaddir=/root/cx/php install php74 php74-php-cli php74-php-fpm php74-php-pdo php74-php-mysqlnd php74-php-mbstring php74-php-opcache php74-php-gd php74-php-bcmath php74-php-xml php74-php-redis
    systemctl enable --now php-fpm
    ```
    - 配置php-fpm&php.ini
        php -i |grep "php.ini"
        编辑文件 /etc/opt/remi/php74/php-fpm.conf
        ```
            error_log = /var/log/php-fpm/error.log
            rlimit_files = 65535
            emergency_restart_threshold = 10
            emergency_restart_interval = 1m
        ```
        编辑文件 /etc/opt/remi/php74/php-fpm.d/www.conf
        ```
            user = www
            group = www
            listen.owner = www
            listen.group = www
            listen.mode = 0660
            listen.allowed_clients = 127.0.0.1
            pm.max_children = 50        ;内存/30M
            pm.start_servers = 10	    ;动态方式下的起始php-fpm进程数量
            pm.min_spare_servers = 5	;动态方式下的最小php-fpm进程数量
            pm.max_spare_servers = 20	;动态方式下的最大php-fpm进程
            pm.max_requests = 1000
            request_slowlog_timeout = 5s
            slowlog = /var/log/php-fpm/slow.log
            ;php_admin_value[error_log] = /var/opt/remi/php74/log/php-fpm/www-error.log
            ;php_admin_flag[log_errors] = on
        ```
        编辑文件 /etc/opt/remi/php74/php.ini
        ```
            date.timezone = PRC
            expose_php = Off
            memory_limit = 512M
            max_execution_time = 30
            error_reporting = E_ALL & ~E_NOTICE & ~E_DEPRECATED & ~E_STRICT & ~E_USER_NOTICE & ~E_USER_DEPRECATED
            display_errors =Off
            log_errors = On
            log_errors_max_len = 1024
            error_log =/var/log/php/error.log
        ```

    - 配置openresty
        - 文件 /etc/security/limits.conf 末尾添加
        ```
            root soft nofile 65535
            root hard nofile 65535
            * soft nofile 65535
            * hard nofile 65535
        ```
        - [nginx.conf](/software/nginx/nginx.conf)

- 安装mysql
    ```
        wget https://repo.mysql.com//mysql80-community-release-el7-1.noarch.rpm
        yum install remi-release-8.rpm
        yum check-update
        yum --disablerepo="*" --enablerepo=mysql80-community list available
        yum install  compat-openssl10 perl-Getopt-Long
        yum --disablerepo="*" --enablerepo=mysql80-community install mysql-community-server
        systemctl enable --now mysqld
    ```
    - 配置mysql
    查看密码 grep 'temporary password' /var/log/mysqld.log
    mysql --help|grep 'my.cnf'
    ```
        SET GLOBAL validate_password.policy=0;
        SET GLOBAL validate_password.length=4;
        ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'security421';
        CREATE USER IF NOT EXISTS 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'security421';
        GRANT ALL ON *.* TO 'root'@'%' WITH GRANT OPTION;
    ```
    - my.cnf
    ```
    [client]
    default-character-set=utf8

    [mysqld]
    default-authentication-plugin=mysql_native_password

    datadir=/var/lib/mysql
    socket=/var/lib/mysql/mysql.sock

    log-error=/var/log/mysqld.log
    pid-file=/var/run/mysqld/mysqld.pid

    # Network related
    bind-address=::
    port=20130

    server-id = 1

    # Enable query cache
    innodb_buffer_pool_size=20M
    max_connections=1000
    wait_timeout=60

    # Replication related
    slave_skip_errors=all

    slow_query_log = 1
    long_query_time = 3
    slow_query_log_file = /var/log/mysql/slow.log

    ```