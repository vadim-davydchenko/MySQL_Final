#### Given 3 machines, set up replication on the second host from the first mysql, transferring data using xtrabackup.Configure proxysql to balance read requests between the master server and the replica.Set up telegraf on the mysql master and slave with export of metrics in prometheus format and connect it to the prometheus server on the 3rd node.

**Database backup from the first host to the second**

*Installing the xtrbackup utility on the first host*

`sudo apt update`

`sudo apt install percona-xtrabackup-80 -y`

*Making a database backup*

`xtrabackup --password=Pssw0RdD --backup  --target-dir=/opt/backup`

`xtrabackup --prepare --target-dir=/opt/backup`

*Installing percona-server on a third host*

`wget https://repo.percona.com/apt/percona-release_latest.$(lsb_release -sc)_all.deb`

`sudo dpkg -i percona-release_latest.$(lsb_release -sc)_all.deb`

`sudo percona-release setup ps80`

`sudo apt install percona-server-server -y`

On the third host, stop mysql and clear the datadir folder

```
sudo systemctl stop mysql
rm -rf /var/lib/mysql/*
```

Copy backup from the first host to the second and create the necessary folder on the 2nd node to launch

`sudo scp -r /opt/backup user@<node3>:/home/user/`

```
mkdir '#innodb_redo'
chown mysql:mysql *
mv * /var/lib/mysql/
systemctl start mysql
```

**Setting up replication**

*First host*

`vim /etc/mysql/mysql.conf.d/mysqld.cnf`

```
server-id       = 1
```
`systemctl restart mysql`

*Create a user for the replica*

```
mysql -pPssw0RdD
CREATE USER replica;
ALTER USER replica IDENTIFIED WITH mysql_native_password BY '123456';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO replica;
SHOW MASTER STATUS\G
```

*On the second host*

```
server-id       = 2
log_bin         = /var/log/mysql/mysql-bin.log
```
We turn on Slave, take data about the log file and position from the first node from SHOW MASTER STATUS

```
CHANGE MASTER TO MASTER_HOST='ip_master_node', MASTER_USER='replica', MASTER_PASSWORD='123456', MASTER_LOG_FILE='binlog.000013', MASTER_LOG_POS=157;
START SLAVE;
SHOW SLAVE STATUS\G
```
**п.2 Configure proxysql to balance read requests between master server and replica**

Install ProxySQL

```
wget -O - 'https://repo.proxysql.com/ProxySQL/repo_pub_key' | apt-key add - 
echo deb https://repo.proxysql.com/ProxySQL/proxysql-2.2.x/$(lsb_release -sc)/ ./ | tee /etc/apt/sources.list.d/proxysql.list
apt-get update
apt-get install proxysql
```
Change config в [/etc/proxysql.cnf](https://github.com/vadim-davydchenko/MySQL_Final/blob/master/proxysql.cnf)

**п.3 Set up telegraf on the mysql master and slave with export of metrics in prometheus format and connect it to the prometheus server on the 2nd node**

*Install telegraf*

```
wget -q https://repos.influxdata.com/influxdata-archive_compat.key
echo '393e8779c89ac8d958f81f942f9ad7fb82a25e133faddaf92e15b16e6ac9ce4c influxdata-archive_compat.key' | sha256sum -c && cat influxdata-archive_compat.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg > /dev/null
echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg] https://repos.influxdata.com/debian stable main' | sudo tee /etc/apt/sources.list.d/influxdata.list
sudo apt-get update && sudo apt-get install telegraf
```
*Creating an exporter user on both mysql (enough for master, replicated on slave)*

```
CREATE USER 'exporter'@'127.0.0.1' IDENTIFIED BY 'password' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'127.0.0.1';
```
We set up [telegraf.conf](https://github.com/vadim-davydchenko/MySQL_Final/blob/master/telegraf.conf) , on slave mysql the config is the same, just change the line

`"exporter:password@tcp(<public_slave_mysql>:3306)/?tls=false"`

Setting up prometheus along the path /opt/prometheus/prometheus.yml

```
  - job_name: 'telegraf'
    static_configs:
    - targets: ['51.250.18.64:9144']
  - job_name: 'mysql'
    static_configs:
    - targets: ['<master_sql>:9273', '<slave_sql>:9273']
```

