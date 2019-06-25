# mosquitto-auth-plug

I was try repository from JP Mens (https://github.com/jpmens/mosquitto-auth-plug)
but he "...archiving this repository and closing all currently open issues and pull requests without prejudice...." (((

and his mosquitto-auth-plug (https://github.com/jpmens/mosquitto-auth-plug) dont compile (build) with lattest Mosquitto broker (1.6.3).

I create fork and I was make some changes for compile mosquitto-auth-plug with lattest Mosquitto (try version 1.6.3)

I was tested with MySQL only. 

This is fork of https://github.com/jpmens/mosquitto-auth-plug

The original source https://github.com/jpmens/mosquitto-auth-plug

# mosquitto-auth-plug

This is a plugin to authenticate and authorize [Mosquitto] users from one or more
of a variety of back-ends:

* [CDB][cdb]
* [Files][files]
* **[HTTP][http]** (custom HTTP API)
* **[JWT][jwt]**
* [LDAP][ldap]
* **[MongoDB][mongo]**
* **[MySQL][mysql]**
* **[PostgreSQL][postgres]**
* [Redis][redis] key/value store
* [SQLite3 database][sqlite]
* [TLS PSK][psk] (the `psk` back-end is a bit of a shim which piggy-backs on the other database back-ends)

## Introduction

This plugin can perform authentication (check username / password)
and authorization (grant permission to subscribe and/or publish to specific topics via ACL). Currently, not all back-ends have the same capabilities
(see the section on the back-end you're interested in).

| Capability                 | [cdb] |[files]|[http]|[jwt]|[ldap]| [mongo] |[mysql]|[postgres]|[psk]|[redis]|[sqlite]|
| -------------------------- | :---: | :----:| :--: | :-: | :-:  | :-----: | :---: | :------: | :-: | :---: | :---:
| authentication             |   Y   | Y     |  Y   |  Y  |  Y   |  Y      |   Y   |    Y     |  Y  |   Y   |   Y
| superusers                 |       |       |  Y   |  Y  |      |  Y      |   Y   |    Y     |  3  |       |        |
| acl checking               |   2   | Y     |  Y   |  Y  |      |  Y      |   Y   |    Y     |  3  |   1   |   2
| static superusers          |   Y   | Y     |  Y   |  Y  |      |  Y      |   Y   |    Y     |  3  |   Y   |   Y

 1. Topic wildcards (+/#) are not supported
 2. Currently not implemented; back-end returns TRUE
 3. Dependent on the database used by PSK

Multiple back-ends can be configured simultaneously for authentication, and they're attempted in
the order you specify. Once a user has been authenticated, the _same_ back-end is used to
check authorization (ACLs). Superusers are checked for in all back-ends.
The configuration option is called `auth_opt_backends` and it takes a
comma-separated list of back-end names which are checked in exactly that order.

```
auth_opt_backends cdb,sqlite,mysql,redis,postgres,http,jwt,mongo
```

Note: anonymous MQTT connections are assigned a username configured in the
plugin as `auth_opt_anonusername` and they
are handled by a so-called _fallback back-end_ which is the *first* configured
back-end.

Passwords are obtained from the back-end as PBKDF2 strings (see [Passwords](#passwords) below). If you store a clear-text password or any hash not generated the same way,
the comparison and the authentication will fail.

The mysql and mongo back-ends support expansion of `%c` and `%u` as clientid and username
respectively. This allows ACLs in the database to look like this:

```
+-----------+---------------------------------+----+
| username  | topic                           | rw |
+-----------+---------------------------------+----+
| bridge-01 | $SYS/broker/connection/%c/state |  2 |
+-----------+---------------------------------+----+
```

The plugin supports so-called _superusers_. These are usernames exempt
from ACL checking. In other words, if a user is a _superuser_, that user
can access any topic without needing ACLs.

A _static superuser_ is one configured with the _fnmatch(3)_ `auth_opt_superusers`
option. Regular _superusers_ are configured (i.e., enabled) from within the
particular database back-end. Effectively, both are identical in that ACL
checking is disabled if a user is a superuser.

Note that not all back-ends currently have 'superuser' queries implemented.
This is a todo and the `auth_opt_superusers` option will probably disappear when it is finished.

## Building the plugin

In order to compile the plugin you'll require:
* a copy of the [Mosquitto] source code together with the libraries required for the back-end you want to use in
the plugin, and
* a recent version of OpenSSL (if the version with your OS, e.g., OS X, is too old, you may need to use one
supplied by home brew or build your own).

Copy `config.mk.in` to `config.mk` and modify `config.mk` to suit your building environment. In particular, you have
to configure which back-ends you want to provide as well as the path to the
[Mosquitto] source and its library, and possibly the path to OpenSSL (`OPENSSLDIR`).

After a `make` you should have a shared object called `auth-plug.so`
which you will reference in your `mosquitto.conf`.

## Configuration

The plugin is configured in [Mosquitto]'s configuration file (typically `mosquitto.conf`),
and it is loaded into Mosquitto auth with the ```auth_plugin``` option.


```
auth_plugin /path/to/auth-plug.so
```

Options therein with a leading ```auth_opt_``` are handed to the plugin. The following
"global" ```auth_opt_*``` plugin options exist:

| Option         | default    |  Mandatory  | Meaning               |
| -------------- | ---------- | :---------: | --------------------- |
| backends       |            |     Y       | comma-separated list of back-ends to load |
| superusers     |            |             | fnmatch(3) case-sensitive string
| log_quiet      | false      |             | don't log DEBUG messages |
| cacheseconds   |                   |             | Deprecated. Alias for acl_cacheseconds
| acl_cacheseconds  | 300               |             | number of seconds to cache ACL lookups. 0 disables
| auth_cacheseconds | 0                 |             | number of seconds to cache AUTH lookups. 0 disables
| acl_cachejitter   | 0                 |             | maximum number of seconds to add/remove to ACL lookups cache TTL. 0 disables
| auth_cachejitter  | 0                 |             | maximum number of seconds to add/remove to AUTH lookups cache TTL. 0 disables

Individual back-ends each have various additional options described in the sections below.

There are two caches, one for ACL and another for authentication. By default only the ACL cache is enabled.

After a backend responds (postitively or negatively) to an ACL or AUTH lookup, the result will be kept in cache for
the configured TTL. The same ACL lookup will be served from the cache as long as the TTL is valid.
The configured TTL is the `auth_cacheseconds`/`acl_cacheseconds` combined with a random value between -`auth_`/`acl_cachejitter` and +`auth_`/`acl_cachejitter`.
For example, with an acl_cacheseconds of 300 and acl_cachejitter of 10, ACL lookup TTLs are distributed between 290 and 310 seconds.

Set auth/acl_cachejitter to 0 disable any randomization of cache TTL. Setting auth/acl_cacheseconds to 0 disables caching entirely.
Caching is useful when your backend lookup is expensive. Remember that ACL lookup will be performed for each message which is sent/received on a topic.
Jitter is useful to reduce lookup storms that could occur every auth/acl_cacheseconds if lots of clients connect at the same time (for example,
after a server restart, all your clients may reconnect immediately and each cause ACL lookups every acl_cacheseconds).

### MySQL auth

The `mysql` back-end is currently the most feature-complete: it supports
obtaining passwords, checking for _superusers_, and verifying ACLs by
configuring up to three distinct SQL queries used to obtain those results.

You configure the SQL queries in order to adapt to whichever schema
you currently have.

The following `auth_opt_` options are supported by the mysql back-end:

| Option         | default           |  Mandatory  | Meaning               |
| -------------- | ----------------- | :---------: | --------------------- |
| host           | localhost         |             | hostname/address
| port           | 3306              |             | TCP port
| user           |                   |             | username
| pass           |                   |             | password
| dbname         |                   |     Y       | database name
| userquery      |                   |     Y       | SQL for users
| superquery     |                   |             | SQL for superusers
| aclquery       |                   |             | SQL for ACLs
| mysql_opt_reconnect | true         |             | enable MYSQL_OPT_RECONNECT option
| mysql_auto_connect  | true         |             | enable auto_connect function
| anonusername   | anonymous         |             | username to use for anonymous connections
| ssl_enabled    | false 	     |		   | enable SSL 
| ssl_key        |   	 	     |		   | path name of client private key file
| ssl_cert       | 	 	     |		   | path name of client public key certificate file  
| ssl_ca         | 	 	     |		   | path name of Certificate Authority(CA) certificate file 
| ssl_capath     | 	 	     |		   | path name of directory that contains trusted CA certifcate files 
| ssl_cipher     | 	 	     |		   | permitted ciphers for SSL encryption 

The SQL query for looking up a user's password hash is mandatory. The query
MUST return a single row only (any other number of rows is considered to be
"user not found"), and it MUST return a single column with only the PBKDF2
password hash. Two `'%s'` in the `auth_opt_userquery` string are replaced by the
username attempting to access the broker and the clientid, in that order. If the clientid is not
to be used in the SQL, insert just a single `'%s'`:

```sql
SELECT pw FROM users WHERE username = '%s' LIMIT 1
```

The SQL query for checking whether a user is a _superuser_ - and thus
circumventing ACL checks - is optional. If it is specified, the query MUST
return a single row with a single value: 0 is false and 1 is true. We recommend
using a `SELECT IFNULL(COUNT(*),0) FROM ...` for this query as it satisfies
both conditions. A single `'%s`' in the `auth_opt_superquery` string is replaced by the
username attempting to access the broker. The following example uses the
same `users` table, but it could just as well reference a distinct table
or view.

```sql
SELECT IFNULL(COUNT(*), 0) FROM users WHERE username = '%s' AND super = 1
```

The SQL query for checking ACLs is optional, but if it is specified, the
`mysql` back-end can try to limit access to particular topics or topic branches
depending on the value of a database table. The query MAY return zero or more
rows for a particular user, each containing EXACTLY one column containing a
topic (wildcards are supported). A single `'%s`' in the query string is
replaced by the username attempting to access the broker, and a single `'%d`' is
replaced with an integer, `1` signifying a read-only access attempt
(SUB) or `2` signifying a read-write access attempt (PUB).

WARNING: The new version of mosquitto add new flag MOSQ_ACL_SUBSCRIBE = 4. 

```
#define MOSQ_ACL_NONE 0x00
#define MOSQ_ACL_READ 0x01
#define MOSQ_ACL_WRITE 0x02
#define MOSQ_ACL_SUBSCRIBE 0x04
```

I have not had time to figure out what the difference is 2 and 4, here is the text from the official document mosquito:

```
 *  MOSQ_ACL_SUBSCRIBE when a client is asking to subscribe to a topic string.
 *                     This differs from MOSQ_ACL_READ in that it allows you to
 *                     deny access to topic strings rather than by pattern. For
 *                     example, you may use MOSQ_ACL_SUBSCRIBE to deny
 *                     subscriptions to '#', but allow all topics in
 *                     MOSQ_ACL_READ. This allows clients to subscribe to any
 *                     topic they want, but not discover what topics are in use
 *                     on the server.
 *  MOSQ_ACL_READ      when a message is about to be sent to a client (i.e. whether
 *                     it can read that topic or not).
 *  MOSQ_ACL_WRITE     when a message has been received from a client (i.e. whether
 *                     it can write to that topic or not).
```


In the following example, the table has an `INT(1)` column `rw` containing `1` for
readonly topics, and `2` for read-write, and `4` for subscribe topics:

```sql
SELECT topic FROM acls WHERE (username = '%s') AND (rw >= %d)
```

Sample Mosquitto configuration (e.g., `mosquitto.conf`) for the `mysql` back-end:

```
auth_plugin /home/jpm/mosquitto-auth-plug/auth-plug.so
auth_opt_host localhost
auth_opt_port 3306
auth_opt_dbname test
auth_opt_user jjj
auth_opt_pass supersecret
auth_opt_userquery SELECT pw FROM users WHERE username = '%s'
# auth_opt_userquery SELECT pwhash FROM user WHERE username = '%s' AND clientid = '%s'
auth_opt_superquery SELECT COUNT(*) FROM users WHERE username = '%s' AND super = 1
auth_opt_aclquery SELECT topic FROM acls WHERE (username = '%s') AND (rw >= %d)
auth_opt_anonusername AnonymouS
```

Assuming the following database tables:

```
mysql> SELECT * FROM users;
+----+----------+---------------------------------------------------------------------+-------+
| id | username | pw                                                                  | super |
+----+----------+---------------------------------------------------------------------+-------+
|  1 | jjolie   | PBKDF2$sha256$901$x8mf3JIFTUFU9C23$Mid2xcgTrKBfBdye6W/4hE3GKeksu00+ |     0 |
|  2 | a        | PBKDF2$sha256$901$XPkOwNbd05p5XsUn$1uPtR6hMKBedWE44nqdVg+2NPKvyGst8 |     0 |
|  3 | su1      | PBKDF2$sha256$901$chEZ4HcSmKtlV0kf$yRh2N62uq6cHoAB6FIrxIN2iihYqNIJp |     1 |
+----+----------+---------------------------------------------------------------------+-------+

mysql> SELECT * FROM acls;
+----+----------+-------------------+----+
| id | username | topic             | rw |
+----+----------+-------------------+----+
|  1 | jjolie   | loc/jjolie        |  1 |
|  2 | jjolie   | $SYS/something    |  1 |
|  3 | a        | loc/test/#        |  1 |
|  4 | a        | $SYS/broker/log/+ |  1 |
|  5 | su1      | mega/secret       |  1 |
|  6 | nop      | mega/secret       |  1 |
+----+----------+-------------------+----+
```

the above SQL queries would enable the following combinations (the `*` at
the beginning of the line indicates a _superuser_)

```
  jjolie     PBKDF2$sha256$901$x8mf3JIFTUFU9C23$Mid2xcgTrKBfBdye6W/4hE3GKeksu00+
	loc/a                                    DENY
	loc/jjolie                               PERMIT
	mega/secret                              DENY
	loc/test                                 DENY
	$SYS/broker/log/N                        DENY
  nop        <nil>
	loc/a                                    DENY
	loc/jjolie                               DENY
	mega/secret                              PERMIT
	loc/test                                 DENY
	$SYS/broker/log/N                        DENY
  a          PBKDF2$sha256$901$XPkOwNbd05p5XsUn$1uPtR6hMKBedWE44nqdVg+2NPKvyGst8
	loc/a                                    DENY
	loc/jjolie                               DENY
	mega/secret                              DENY
	loc/test                                 PERMIT
	$SYS/broker/log/N                        PERMIT
* su1        PBKDF2$sha256$901$chEZ4HcSmKtlV0kf$yRh2N62uq6cHoAB6FIrxIN2iihYqNIJp
	loc/a                                    PERMIT
	loc/jjolie                               PERMIT
	mega/secret                              PERMIT
	loc/test                                 PERMIT
	$SYS/broker/log/N                        PERMIT
```

The `mysql` back-end will re-connect to the MySQL server when the connection has been lost.
If you wish, you can disable this by configuring:

```
auth_opt_mysql_opt_reconnect false
auth_opt_mysql_auto_connect false
```

### LDAP auth

The LDAP plugin currently does authentication only; authenticated users are allowed
to publish/subscribe at will.

The user that connects to the broker is searched for in the LDAP directory indicated
via the `ldap_uri` configuration parameter. This LDAP search MUST return exactly one
entry. The user's password is then used with the DN of the that entry to bind to the
directory. If that LDAP bind succeeds, the user is authenticated. In all other cases,
authentication fails.


| Option         | default           |  Mandatory  | Meaning     |
| -------------- | ----------------- | :---------: | ----------  |
| binddn         |                   |     Y       | the DN of an object which may search users |
| bindpw         |                   |     Y       | its password                               |
| ldap_uri       |                   |     Y       | an LDAP uri with filter                    |
| ldap_acl_deny  | false             |             | return DENY instead of ALLOW to ACL checks |

Example configuration:

```
auth_plugin /path/to/auth-plug.so
auth_opt_backends ldap
auth_opt_binddn cn=manager,dc=mens,dc=de
auth_opt_bindpw s3crit
auth_opt_ldap_uri ldap://127.0.0.1/ou=Users,dc=mens,dc=de?cn?sub?(&(objectclass=inetOrgPerson)(uid=@))
auth_opt_ldap_acl_deny false
```

With the `ldap_acl_deny` we return DENY instead of ALLOW for every ACL check. This makes it possible to chain other backends with ldap backend, and use LDAP for authentification and, e.g., MySQL for ACL checking.

### CDB auth

| Option         | default           |  Mandatory  | Meaning     |
| -------------- | ----------------- | :---------: | ----------  |
| cdbname        |                   |     Y       | path to .cdb |

### SQLITE auth

| Option          | default           |  Mandatory  | Meaning     |
| --------------- | ----------------- | :---------: | ----------  |
| dbpath          |                   |     Y       | path to database |
| sqliteuserquery |                   |     Y       | SQL for users |

Example:

```
auth_opt_sqliteuserquery SELECT pw FROM users WHERE username = ?
```

### Redis auth


```
auth_opt_redis_userquery GET %s
auth_opt_redis_aclquery GET %s-%s
```

In `auth_opt_redis_userquery` the `%s` parameter is the _username_, whereas in `auth_opt_redis_aclquery`, the first `%s` is the _username_ and the second is the _topic_. When using ACLs, _topic_ must be an exact match - wildcards are not supported.

If no options are provided, then the plugin will default to not using an ACL and using the above userquery.


| Option         | default           |  Mandatory  | Meaning     |
| -------------- | ----------------- | :---------: | ----------  |
| redis_host     | localhost         |             | hostname / IP address
| redis_port     | 6379              |             | TCP port number |

### HTTP auth

The `http` back-end is for auth by custom HTTP API.

The following `auth_opt_` options are supported by the `http` back-end:

| Option            | default           |  Mandatory  | Meaning     |
| ----------------- | ----------------- | :---------: | ----------  |
| http_ip           |                   |      Y      | IP address, will skip DNS lookup |
| http_port         | 80                |             | TCP port number                 |
| http_hostname     |                   |             | hostname for HTTP header        |
| http_getuser_uri  |                   |      Y      | URI for checking username/password |
| http_superuser_uri|                   |      Y      | URI for checking superuser         |
| http_aclcheck_uri |                   |      Y      | URI for checking acl               |
| http_with_tls     | false             |             | Use TLS on connect              |
| http_basic_auth_key|                  |             | Basic Authentication Key        |
| http_retry_count  | 3                 |             | Number of retries done if backend is unavailable |

If the configured URLs return an HTTP status code == `2xx`, the authentication /
authorization succeeds. If the status code == `4xx`, authentication /
authorization fails. For a status code == `5xx` or server `Unreachable`, the HTTP request
will be retried up to `http_retry_count`. If all tries fail and if no other backend succeeded,
then an error is returned and the client is disconnected.

| URI-Param         | username | password | clientid | topic | acc |
| ----------------- | -------- | -------- | -------- | :---: | :-: |
| http_getuser_uri  |   Y      |   Y      |   N      |   N   |  N  |
| http_superuser_uri|   Y      |   N      |   N      |   N   |  N  |
| http_aclcheck_uri |   Y      |   N      |   Y      |   Y   |  Y  |

Mosquitto configuration for the `http` back-end:

```
auth_opt_backends http
auth_opt_http_ip 127.0.0.1
auth_opt_http_port 8089
#auth_opt_http_hostname example.org
auth_opt_http_getuser_uri /auth
auth_opt_http_superuser_uri /superuser
auth_opt_http_aclcheck_uri /acl
```

A very simple example service using Python and [bottle](https://bottlepy.org/docs/dev/) can be found in [examples/http-auth-be.py](examples/http-auth-be.py).

The _http_ plugin can utilize environment variables which are exported before it (i.e., Mosquitto) is started by adding configuration settings like

```
auth_opt_<interface>_<method>_params <key>=<evn_name>[,<key>=<evn_name>]*
```

For example, set the following:

```bash
export DOMAIN=example.com
export PORT=8080
```

and add the following settings to `mosquitto.conf`:

```
auth_opt_http_getuser_params domain=DOMAIN,port=PORT
auth_opt_http_superuser_params domain=DOMAIN,port=PORT
auth_opt_http_aclcheck_params domain=DOMAIN,port=PORT
```



### JWT auth

The `jwt` back-end is for auth by [JWT-webtokens](https://jwt.io/). The JWT and HTTP configurations are identical, so please read the `http`-section above.

The `username` field is interpreted as the token-field and passed to the http-server in an Authorization-header.
```
Authorization: Bearer %token
```

**Note**: Some clients require the `password` field to be populated. This field is ignored by the JWT-backend, so feel free to input some gibberish.



### PostgreSQL auth

The `postgres` back-end, like `mysql`, is currently the most feature-complete: it supports
distinct SQL queries for obtaining passwords, checking for _superusers_, and verifying ACLs,
each configurable to suit your schema.

The following `auth_opt_` options are supported by the `postgres` back-end:

| Option         | default           |  Mandatory  | Meaning                  |
| -------------- | ----------------- | :---------: | ------------------------ |
| host           | localhost         |             | hostname/address
| port           | 5432              |             | TCP port
| user           |                   |             | username
| pass           |                   |             | password
| dbname         |                   |     Y       | database name
| userquery      |                   |     Y       | SQL for users
| superquery     |                   |             | SQL for superusers
| aclquery       |                   |             | SQL for ACLs
| sslcert        |                   |             | SSL/TLS Client Cert.
| sslkey         |                   |             | SSL/TLS Client Cert. Key

The SQL query for looking up a user's password hash is mandatory. The query
**must** return a single row only (any other number of rows is considered to be
"user not found"), and it **must** return a single column only with the PBKDF2
password hash. A single `$1` in the query string is replaced by the
username attempting to access the broker.

```sql
SELECT pass FROM account WHERE username = $1 limit 1
```

The SQL query for checking whether a user is a _superuser_ - and thus
circumventing ACL checks - is optional. If it is specified, the query **must**
return a single row with a single value: 0 is false and 1 is true. We recommend
using a `SELECT COALESCE(COUNT(*),0) FROM ...` for this query as it satisfies
both conditions. A single `$1` in the `auth_opt_superquery` string is replaced by the
username attempting to access the broker. The following example uses the
same `account` table, but it could just as well reference a distinct table
or view.

```sql
SELECT COALESCE(COUNT(*),0) FROM account WHERE username = $1 AND super = 1
```

The SQL query for checking ACLs is optional, but if it is specified, the
`postgres` back-end can try to limit access to particular topics or topic branches
depending on the value of a database table. The query MAY return zero or more
rows for a particular user, each containing EXACTLY one column containing a
topic (wildcards are supported). A single `$1` in the query string is
replaced by the username attempting to access the broker, and a single `$2` is
replaced with an integer, `1` signifying a read-only access attempt
(SUB) or `2` signifying a read-write access attempt (PUB).

In the following example, the table has a column `rw` containing 1 for
readonly topics, 2 for writeonly topics and 3 for readwrite topics:

```sql
SELECT topic FROM acl WHERE (username = $1) AND rw >= $2
```

Sample Mosquitto configuration for the `postgres` back-end:

```
auth_plugin /home/jpm/mosquitto-auth-plug/auth-plug.so
auth_opt_host localhost
auth_opt_port 5432
auth_opt_dbname test
auth_opt_user jjj
auth_opt_pass supersecret
auth_opt_userquery SELECT pw FROM account WHERE username = $1 limit 1
auth_opt_superquery SELECT COALESCE(COUNT(*),0) FROM account WHERE username = $1 AND mosquitto_super = 1
auth_opt_aclquery SELECT topic FROM acls WHERE (username = $1) AND (rw & $2) > 0
auth_opt_sslcert /etc/postgresql/ssl/client.crt
auth_opt_sslkey /etc/postgresql/ssl/client.key
```
Assuming the following database tables:

```
=> SELECT * FROM account;
+----+----------+---------------------------------------------------------------------+-------+
| id | username | pw                                                                  | super |
+----+----------+---------------------------------------------------------------------+-------+
|  1 | jjolie   | PBKDF2$sha256$901$x8mf3JIFTUFU9C23$Mid2xcgTrKBfBdye6W/4hE3GKeksu00+ |     0 |
|  2 | a        | PBKDF2$sha256$901$XPkOwNbd05p5XsUn$1uPtR6hMKBedWE44nqdVg+2NPKvyGst8 |     0 |
|  3 | su1      | PBKDF2$sha256$901$chEZ4HcSmKtlV0kf$yRh2N62uq6cHoAB6FIrxIN2iihYqNIJp |     1 |
+----+----------+---------------------------------------------------------------------+-------+

=> SELECT * FROM acls;
+----+----------+-------------------+----+
| id | username | topic             | rw |
+----+----------+-------------------+----+
|  1 | jjolie   | loc/jjolie        |  1 |
|  2 | jjolie   | $SYS/something    |  1 |
|  3 | a        | loc/test/#        |  1 |
|  4 | a        | $SYS/broker/log/+ |  1 |
|  5 | su1      | mega/secret       |  1 |
|  6 | nop      | mega/secret       |  1 |
+----+----------+-------------------+----+
```

the above SQL queries would enable the following combinations (the `*` at
the beginning of the line indicates a _superuser_)

```
  jjolie     PBKDF2$sha256$901$x8mf3JIFTUFU9C23$Mid2xcgTrKBfBdye6W/4hE3GKeksu00+
  loc/a                                    DENY
  loc/jjolie                               PERMIT
  mega/secret                              DENY
  loc/test                                 DENY
  $SYS/broker/log/N                        DENY
  nop        <nil>
  loc/a                                    DENY
  loc/jjolie                               DENY
  mega/secret                              PERMIT
  loc/test                                 DENY
  $SYS/broker/log/N                        DENY
  a          PBKDF2$sha256$901$XPkOwNbd05p5XsUn$1uPtR6hMKBedWE44nqdVg+2NPKvyGst8
  loc/a                                    DENY
  loc/jjolie                               DENY
  mega/secret                              DENY
  loc/test                                 PERMIT
  $SYS/broker/log/N                        PERMIT
* su1        PBKDF2$sha256$901$chEZ4HcSmKtlV0kf$yRh2N62uq6cHoAB6FIrxIN2iihYqNIJp
  loc/a                                    PERMIT
  loc/jjolie                               PERMIT
  mega/secret                              PERMIT
  loc/test                                 PERMIT
  $SYS/broker/log/N                        PERMIT
```

_Note that the above sample `auth_opt_aclquery` is sensitive to [new permission values used in Mosquitto 1.5.](#https://github.com/jpmens/mosquitto-auth-plug/issues/356)_

You can either adapt to the updated binary-style permissions 
(`2` for write, `5` for read+subscribe, `7` for read/write),
modify your query to work around them, or modify the constants in the Mosquitto source.

## MongoDB auth
The `mongo` back-end works with superuser and ACL checks. Additional build dependencies are https://github.com/mongodb/mongo-c-driver `>=1.4.0`
and https://github.com/mongodb/libbson `>=1.4.0`.

You should set up a users collection (required) and a topic lists collection (optional) with the following format:

#### Users collection

Each user document must have a username, a hashed password, and at least one of:

 - A superuser prop, allowing full access to all topics
 - An embedded array or sub-document to use as an ACL (see 'ACL format')
 - A foreign key pointing to another document containing an ACL (see 'ACL format')

You may use any combination of these options; authorisation will be granted if any check passes.

The user document has the following format (note that the property names are configurable variables, see 'Configuration').

```
{
    [user_username_prop]: string, // Username as given in the MQTT connect request
    [user_password_prop]: string, // A PBKDF2 hash, see 'Passwords' section
    [user_topiclist_fk_prop]: int | oid | string, // reference to a document in collection_topics)
    [user_topics_prop]: string[] | { [topic: string]: "r"|"w"|"rw" }, // see 'ACL format'
    [user_superuser_prop]: int | boolean // optional, superuser if truthy
}
```

As an example using default options, a user document with an embedded ACL might look like:

```json
{
    "username": "user1",
    "password": "PBKDF2$sha256$901$8ebTR72Pcmjl3cYq$SCVHHfqn9t6Ev9sE6RMTeF3pawvtGqTu",
    "superuser": false,
    "topics": {
        "public/#": "r",
	"client/user1/#": "rw"
    }
}
```

#### Topic lists collection (optional)

If the user document references a separate topics document, that document should exist and must have the format:

```
{
    [topiclist_key_prop]: int | oid | string, // unique id, as referenced by users[user_topiclist_fk_prop],
    [topiclist_topics_prop]: string[] | { [topic: string]: "r"|"w"|"rw" } // see 'ACL format'
}
```

This strategy will be especially suitable if you have a complex ACL shared between many users.

#### ACL format

Topics may be given as either an array of topic strings, eg `["topic1/#", "topic2/+"]`, in which case all topics will
be read-write, or as a sub-document mapping topic names to the strings `"r"`, `"w"`, `"rw"`, eg
`{ "article/#":"r", "article/+/comments":"rw", "ballotbox":"w" }`.

#### Configuration

The following `auth_opt_mongo_` options are supported by the mongo back-end:

| Option                 | default       | Meaning               |
| ---------------------- | ------------- | --------------------- |
| uri                    | mongodb://localhost:27107 | [MongoDB connection string] (database part is ignored)
| database               | mqGate                    | Name of the database containing users (and topiclists)
| user_coll              | users                     | Collection for user documents
| topiclist_coll         | topics                    | Collection for topiclist documents (optional if embedded topics are used)
| user_username_prop     | username                  | Username property name in the user document
| user_password_prop     | password                  | Password property name in the user document
| user_superuser_prop    | superuser                 | Superuser property name in the user document
| user_topics_prop       | topics                    | Name of a property on the user document containing an embedded topic list
| user_topiclist_fk_prop | topics                    | Property used as a foreign key to reference a topiclist document
| topiclist_key_prop     | _id                       | Unique key in the topiclist document pointed to by user_topiclist_fk_prop
| topiclist_topics_prop  | topics                    | Property containing topics within the topiclist document

Mosquitto configuration for the `mongo` back-end:
```
auth_plugin /home/jpm/mosquitto-auth-plug/auth-plug.so
auth_opt_mongo_uri mongodb://localhost:27017
```
## Files auth

The `files` backend attempts to re-implement the files behavior in vanilla Mosquitto, however the user's password file contains PBKDF2 passwords instead of passwords hashed with the `mosquitto-passwd` program; you would use our `np` utility or similar to create the PBKDF2 hashes.

The configuration directives for the `Files` backend are as follows:

```
auth_opt_backends files
auth_opt_password_file file.pw
auth_opt_acl_file file.acl
```

with examples of these files being:

#### `password_file`

```
# comment
jpm:PBKDF2$sha256$901$UGfDz79cAaydRsEF$XvYwauPeviFd1NfbGL+dxcn1K7BVfMeW
jane:PBKDF2$sha256$901$wvvH0fe7Ftszt8nR$NZV6XWWg01dCRiPOheVNsgMJDX1mzd2v
```

#### `acl_file`

```
user jane
topic read #

user jpm
topic dd

```

The syntax for the ACL file is that as described in `mosquitto.conf(5)`.

## PSK auth

If [Mosquitto] has been built with PSK support, and _auth-plug_ has been built
with `BE_PSK` defined, it supports authenticating PSK connections over TLS, as
long as Mosquitto is appropriately configured.

The way this works is that the `psk` back-end actually uses one of _auth-plug_'s
other databases (`mysql`, `sqlite`, `cdb`, etc.) to obtain the pre-shared key
from the "users" query, and it uses the same database's back-end for performing
authorization (aka ACL checks).

Consider the following `mosquitto.conf` snippet:

```
...
auth_opt_psk_database mysql
...
listener 8885
psk_hint hint1
tls_version tlsv1
use_identity_as_username true
```

TLS PSK is available on port 8885 and is activated with, say,

```
mosquitto_pub -h localhost -p 8885 -t x -m hi --psk-identity ps2 --psk 020202
```

The `use_identity_as_username` option has _auth-plug_ see the name `ps2` as the
username, and this is given to the database back-end (here: `mysql`) to look up
the password as defined for the `mysql` back-end. _auth-plug_ uses its `getuser()` query
to read the clear-text (not PKBDF2) hex key string which it returns to Mosquitto
for authentication. If authentication passes, the connection is established.

For authorization, _auth_plug_ uses the identity as the username and the topic to
perform ACL-checking as described earlier.

The following log-snippet serves as an illustration:

```
New connection from ::1 on port 8885.
|-- psk_key_get(hint1, ps1) from [mysql] finds PSK: 1
New client connected from ::1 as mosqpub/90759-tiggr.ww. (c1, k60).
Sending CONNACK to mosqpub/90759-tiggr.ww. (0)
|-- user ps1 was authenticated in back-end 0 (psk)
|--   mysql: topic_matches(x, x) == 1
|-- aclcheck(ps1, x, 2) AUTHORIZED=1 by psk
Received PUBLISH from mosqpub/90759-tiggr.ww. (d0, q0, r0, m0, 'x', ... (2 bytes))
Received DISCONNECT from mosqpub/90759-tiggr.ww.
```

In the case of this MySQL example, we added the clear text of the PSK key to the database:

```
mysql> INSERT INTO user (username, pwhash, superuser) VALUES ('mylistener', 'F0BEEF', 0);
```

## Passwords

A user's password is stored as a [PBKDF2] hash in the back-end. An example
"password" is a string with five pieces in it, delimited by `$`, inspired by
[this][1].

```
PBKDF2$sha256$901$8ebTR72Pcmjl3cYq$SCVHHfqn9t6Ev9sE6RMTeF3pawvtGqTu
--^--- --^--- -^- ------^--------- -------------^------------------
  |      |     |        |                       |
  |      |     |        |                       +-- : hashed password
  |      |     |        +-------------------------- : salt
  |      |     +----------------------------------- : iterations
  |      +----------------------------------------- : hash function
  +------------------------------------------------ : marker
```

Note that the `salt` by default will be taken as-is (thus it will not be
base64 decoded before the validation). In case your own implementation uses
the raw bytes when hashing the password and base64 is only used for display
purpose, compile this project with the `-DRAW_SALT` flag (you could add this
in the `config.mk` file to `CFG_CFLAGS`).

## Creating a user

A trivial utility to generate hashes is included as `np`. Copy and paste the
whole string generated into the respective back-end.

```bash
$ np
Enter password:
Re-enter same password:
PBKDF2$sha256$901$Qh18ysY4wstXoHhk$g8d2aDzbz3rYztvJiO3dsV698jzECxSg
```

For example, in [Redis][Redis-Ext]:

```
$ redis-cli
> SET n2 PBKDF2$sha256$901$Qh18ysY4wstXoHhk$g8d2aDzbz3rYztvJiO3dsV698jzECxSg
> QUIT
```

## Configuring Mosquitto

```
listener 1883

auth_plugin /path/to/auth-plug.so
auth_opt_redis_host 127.0.0.1
auth_opt_redis_port 6379

# Usernames with this fnmatch(3) (a.k.a glob(3))  pattern are exempt from the
# module's ACL checking
auth_opt_superusers S*
```

## ACL

In addition to the ACL checking which might be performed by a back-end,
there's a more "static" checking which can be configured in `mosquitto.conf`.

Note that if ACLs are being verified by the plugin, this also applies to
Will topics (_last will and testament_). Failing to correctly set up
an ACL for these, will cause a broker to silently fail with a 'not
authorized' message.

Users can be given "superuser" status (i.e. they may access any topic)
if their username matches the _glob_ specified in `auth_opt_superusers`.

In our example above, any user with a username beginning with a capital `"S"`
is exempt from ACL-checking.

## PUB/SUB

At this point you ought to be able to connect to [Mosquitto] using, e.g., the Mosquitto client: 

```
mosquitto_pub  -t '/location/n2' -m hello -u n2 -P secret
```

## Requirements

* A [Mosquitto] broker
* OpenSSL (tested with 1.0.0c, but should work with earlier versions)

Some of the back-ends require a server instance or client libraries. For example:
* for [redis]: a [Redis][Redis-Ext] server and [hiredis], the Minimalistic C client for Redis
* for [cdb]: [TinyCDB](http://www.corpit.ru/mjt/tinycdb.html) by Michael Tokarev (included in `contrib/`)
* for [postgres]: the latest `dev` version of `postgresql-server`

## Credits

* Uses `base64.[ch]` (and yes, I know OpenSSL has base64 routines, but no thanks). These files are
>  Copyright (c) 1995, 1996, 1997 Kungliga Tekniska Hgskolan (Royal Institute of Technology, Stockholm, Sweden).
* Uses [uthash] by Troy D. Hanson.


 [Mosquitto]: http://mosquitto.org
 [Redis-Ext]: http://redis.io
 [pbkdf2]: http://en.wikipedia.org/wiki/PBKDF2
 [1]: https://exyr.org/2011/hashing-passwords/
 [hiredis]: https://github.com/redis/hiredis
 [uthash]: http://troydhanson.github.io/uthash/
 [MongoDB connection string]: https://docs.mongodb.com/manual/reference/connection-string/
 
 [mysql]: #mysql-auth
 [postgres]: #postgresql-auth
 [cdb]: #cdb-auth
 [sqlite]: #sqlite-auth
 [redis]: #redis-auth
 [psk]: #psk-auth
 [ldap]: #ldap-auth
 [http]: #http-auth
 [jwt]: #jwt-auth
 [mongo]: #mongodb-auth
 [files]: #files-auth

## Possibly related

 * [docker-mosquitto](https://github.com/jllopis/docker-mosquitto) - easy installation of this plugin
 * [mosquitto_pyauth](https://github.com/mbachry/mosquitto_pyauth)
 * [mosquitto-auth-plugin-http](https://github.com/hadleyrich/mosquitto-auth-plugin-http)
 * [lua_auth_plugin](https://github.com/DenkiYagi/lua_auth_plugin)

## Press

 * [How to make Access Control Lists (ACL) work for Mosquitto MQTT Broker with Auth Plugin](http://my-classes.com/2015/02/05/acl-mosquitto-mqtt-broker-auth-plugin/)
 * [PostgreSQL-based MQTT access control](https://mberka.com/web/postgresql-based-mqtt-access-control)
 * [Raspberry Pi: How to install MQTT broker and mosquitto auth plugin](http://wei48221.blogspot.com/2017/08/raspberry-pi-how-to-install-mqtt-broker.html)
 * [Securing MQT connection using Mosquitto Auth Plugin - HTTP API](http://www.yasith.me/2016/04/securing-mqtt-connection-using.html)
