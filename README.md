# Companion applications for cp4ba-installations tool

## BAW

### cp4ba-custom-db-test

Test application for configuration with custom application db.

see 'Custom db' setion in 'env1-authoring-baw-multi-db.properties'
```bash
export CP4BA_INST_DB_CUSTOMDB_NAME="mydb"
export CP4BA_INST_DB_CUSTOMDB_USER="myuser"
export CP4BA_INST_DB_CUSTOMDB_PWD="dem0s"
```
use jdbc name: jdbc/mydb

setup db

```sql
create table myuser.utenti (
  id bigint primary key,
  name text not null,
  email text not null,
  id_status bigint not null
);

ALTER TABLE myuser.utenti OWNER TO myuser;

INSERT INTO myuser.utenti (id, name, email, id_status) VALUES (1, 'vuxuser1', 'vuxuser1@example.com', 1);
INSERT INTO myuser.utenti (id, name, email, id_status) VALUES (2, 'vuxuser2', 'vuxuser2@example.com', 1);
INSERT INTO myuser.utenti (id, name, email, id_status) VALUES (3, 'vuxuser3', 'vuxuser3@example.com', 1);
INSERT INTO myuser.utenti (id, name, email, id_status) VALUES (4, 'vuxuser4', 'vuxuser4@example.com', 1);
INSERT INTO myuser.utenti (id, name, email, id_status) VALUES (5, 'vuxuser5', 'vuxuser5@example.com', 1);

select * from myuser.utenti;
```

## ADS

https://github.com/icp4a/automation-decision-services-samples

