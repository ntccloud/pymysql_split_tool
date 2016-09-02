# What is this?

Mysql query will become slow when table contains millions of rows. A simple way of solving this probolem is spliting the big table into small ones. We use some method to decide which table each row should be moved into. A widely used method is "modulus", e.g. group rows by [ID mod 100]. This tool will help you to do this job easily.

# Usage

* Just create a split task file and run this tool with it:

`python3 -m pymysql_shard_tool --action split --file  ./task.json`

* There are two type of action supported:

| action | description |
| -------- | -------- |
| split | Move rows from a big table into small tables. |
| remove | Remove rows from a big table after checking data integrity in small tables. |

* A quick look at a sample task file:

```json
{
  "src":{
    "mysql":{
      "host":"127.0.0.1",
      "port":"3306",
      "unix_socket":"/path/to/mysqld.sock",
      "user":"user_name",
      "passwd":"password"
    },
    "database":"name_of_database",
    "table":"name_of_the_big_table"
  },
  "dest":{
    "table":"small_table_[n]",
  },
  "rule":{
    "filter":"id>1000",
    "group_method":"modulus"
  }
}
```

### Supported fields in task file:

| field @ level-1 | field @ level-2 | field @ level-3 | description | sample |
| -------- | -------- | -------- | -------- | -------- |
| src |  |  | infomation of the being splitted table |  |
|  | mysql |  | mysql conneciton parameters |  |
|  |  | host | mysql host(domain name or ip) | "127.0.0.1" |
|  |  | port | mysql port | 3306 |
|  |  | unix_socket | connect mysql server with unix-socket | "/var/run/mysqld.sock" |
|  |  | user | mysql user name, need read privilege at least | "root" |
|  |  | passwd | mysql user password |  |
|  | database |  | name of database |  |
|  | table |  | name of table |  |
| dest |  |  | infomation of the new created small tables |  |
|  | mysql |  | mysql conneciton parameters. <br/>If not set, will use the same parameters as "src". <br/>Has same level-3 fileds like "src" |  |
|  | database |  | name of database where we create new small tables. <br/>If not set, will use the same database as "src". |  |
|  | table |  | name pattern of table. **[n]** will be replaced by an integer. | "small\_table\_[n]" |
|  | create_table_first |  | if set to 1, it will create the small table if not exists. <br/>default is 1. | 0 |
|  | create_table_sql |  | sql to create new table. <br/>If not set, will use 'show create table' to get a creation sql. | "create table [table_name] if not exists..." |
| rule |  |  | defines how it do the work |  |
|  | filter |  | a sql 'where' clause telling it which rows should be moved | "id>1000" |
|  | page_size |  | page size when select in paging mode | 1000 |
|  | page_sleep |  | sleep a few seconds between pages to avoid mysql server busy working | 1 |
|  | order_by |  | select order in **paging mode** | "id asc" |
|  | group_method |  | split rows by [modulus] or [devide], or [all] | "modulus" |
|  | group_column |  | column name which the method will used on. |
|  | group_int |  | an int array. if set, this tool will move data whose method result fall in this array.<br/>see this [detailed tips](#how_it_works)<br/>for filter [modulus] and [devide] only. | [1,7,[12,15],20] |
|  | check |  | for "remove" action only. how we check data integrity before removing data |  |
|  |  | checksum | using mysql checksum() function. default is 0. | 1 |
|  |  | count | using mysql count() function. default is 1 | 1 |

### how it works

For action "split", there are 4 different work flows:

* if `dest.mysql` is set, `rule.group_int` not set, it works in the following steps:

    1. make connection to "src" and "dest" mysql server.
    2. get data with this sql: select * from `src.database`.`src.table` where `rule.filter` order by `rule.order_by` limit offset,`rule.page_size`
    3. for each selected data, use `rule.group_method` on the `rule.group_column`, get a method result: `n`
    4. ensure table `dest.table_[n]` existance if `dest.create_table_first` exists.
    5. insert data into new table

* if `dest.mysql` is set, `rule.group_int` is set, it works like this:

	1. make connection to "src" and "dest" mysql server.
	2. generate an integer array according to `rule.group_int`.
	3. for each integer `n`, ensure table `dest.table_[n]` existance if `dest.create_table_first` exists.
	4. for each integer `n`, get data with this sql: select * from `src.database`.`src.table` where `rule.filter` and `rule.group_column` method = `n` order by `rule.order_by` limit offset,`rule.page_size`
    5. insert data into new table

* if `dest.mysql` is not set, `rule.group_int` not set, it works like this:

	1. make connection to "src" mysql server
	2. get group list with this sql: select n from `src.database`.`src.table` group by `rule.group_column` method result
	3. for each group n, copy data with this sql: replace into `dest.database`.`dest.table` select * from `src.database`.`src.table` where `rule.filter` where `rule.group_column` method result = `n`


* if `dest.mysql` is not set, `rule.group_int` is set, it works like this:

	1. make connection to "src" mysql server
	2. copy data with this sql: replace into `dest.database`.`dest.table` select * from `src.database`.`src.table` where `rule.filter`