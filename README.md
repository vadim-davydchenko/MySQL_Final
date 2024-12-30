# MySQL Replication and Monitoring Setup

## Task

This project configures:
1. MySQL replication using `xtrabackup` between two servers.
2. ProxySQL to balance read requests between the master and replica.
3. Telegraf for collecting MySQL metrics and exporting them in Prometheus format, connecting to the Prometheus server.

---

## How to Run

### 1. Environment Preparation

Ensure that Ansible is set up and an inventory file with the IP addresses of your hosts is available. See `inventory.yml` for an example structure.

### 2. Playbook Execution

Run the following command to execute the playbook:

```bash
ansible-playbook -i inventory.yml mysql.yml
```

### 3. Playbook Actions

- **MySQL Replication:**
  - Installs `xtrabackup` and creates a database backup.
  - Restores the data on the replica and sets up replication.

- **ProxySQL:**
  - Installs ProxySQL.
  - Configures read query balancing between the master and the replica.

- **Monitoring:**
  - Installs Telegraf on the master and replica servers.
  - Configures metric collection for MySQL and sets up Prometheus exporters.
  - Updates Prometheus configuration to monitor MySQL and Telegraf.

---

## Configuration Files

- **[telegraf.conf](roles/mysql/templates/telegraf.conf.j2):** Telegraf configuration.
- **[proxysql.cnf](roles/mysql/templates/proxysql.cnf.j2):** ProxySQL configuration.

---

## Notes

- Ensure all necessary variables, such as MySQL and ProxySQL passwords, are correctly set in `group_vars/all.yml`.
- MySQL replication automatically syncs the `exporter` user from the master to the replica.
