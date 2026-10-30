# sqlquery — mysqlnd 查询字段追踪扩展

## 功能

在 mysqlnd 层拦截 mysqli / PDO_MySQL 的查询与结果行读取。

- 配置文件：`/tmp/sql_query.json`
- 配置为空、不存在或格式错误时：完全静默，不写任何输出，不影响 PHP 程序。
- 当 SQL 的**发起位置**（PHP 文件 + 行号）与配置匹配时，在请求结束前把 SQL、实际结果字段元数据、实际返回的行值写入：
  - `/tmp/sql/<output>`
- 同一次请求内目标行被多次调用时，每次调用单独输出一条记录（`call_index`）。

## 配置格式

```json
{
  "file": "/app/xvwa/vulnerabilities/stored_xss/home.php",
  "line": 56,
  "output": "result_line56.json"
}
```

兼容字段别名：

- `file` / `path` / `filename`
- `line` / `lineno` / `line_number`
- `output` / `out` / `output_file`

`output` 只取文件名部分，统一放在 `/tmp/sql/`；没有 `.json` 后缀会自动补上。

## 输出格式

```json
{
  "target_file": "...",
  "target_line": 56,
  "query_count": 2,
  "queries": [
    {
      "call_index": 1,
      "sql": "select user,comment,date from comments",
      "kind": "query",
      "matched_at_prepare": false,
      "row_count": 4,
      "affected_rows": 0,
      "execute_time_ms": 0.21,
      "fetch_time_ms": 0.0017,
      "fields": [
        {"name": "user", "org_name": "user", "table": "comments", "db": "xvwa", "type": 253, "length": 120}
      ],
      "rows": [
        ["admin", "Keep posting your comments here ", "10 Aug 2015"]
      ]
    }
  ]
}
```

写入语句没有结果集，输出中 `fields`/`rows` 为空，并带 `affected_rows`、`last_insert_id`。

## 实现要点

- 在 `MINIT` 中通过 mysqlnd 的 methods xetters 包装：
  - `mysqlnd_stmt::prepare`
  - `mysqlnd_stmt::execute`
  - `mysqlnd_conn_data::query`
  - `mysqlnd_res::fetch_row`
- 匹配时遍历完整 PHP 用户态调用栈，而不是只看最内层帧。
  - 例如 WordPress 的 SQL 实际由 `wpdb::_do_query()` 发出，
  - 但用户指定的 `WP_Post::get_instance()` 第 289 行在更上层，需要沿栈匹配。
- 每次匹配的调用创建独立的 `sqlq_ctx`，按调用顺序保存在 `sqlq_records`。
  - 同一个 prepared statement 多次 `execute()` 不会互相覆盖。
  - 同一个连接上的多条查询不会只保留最后一条。
- 通过 `zend_execute_internal` 额外记录 `PDO::prepare()` 的调用位置。
  - PDO_MySQL 默认开启 `ATTR_EMULATE_PREPARES`，`prepare()` 本身不访问数据库；
  - 实际查询发生在 `PDOStatement::execute()` 时；
  - 因此只有记录 prepare 位置，才能按 `$stmt = $conn->prepare(...)` 这一行匹配。
- 行值在 `mysqlnd_res::fetch_row()` 返回后从 `result->row_data` 复制，不改变 PDO/mysqli 的读取游标。
- 输出在 `RSHUTDOWN` 统一写盘。

## 覆盖范围

已覆盖：

- PDO 模拟 prepare / execute（默认）
- PDO 原生 prepare / execute
- PDO `query()` / `exec()`
- mysqli `query()` / `real_query()`
- mysqli 原生 prepare + `bind_result()` / `get_result()`
- mysqli 非缓冲查询 `MYSQLI_USE_RESULT`
- INSERT / UPDATE / DELETE 等写入

未覆盖 / 有限支持：

- `mysqli_multi_query()` 的后续结果集不会拆成多条语句
- 异步查询、LOAD DATA、local infile 等边缘路径

## 编译

```bash
docker exec php84-build bash -lc '
  cd /php-build/php-8.4.25/ext/sqlquery &&
  phpize &&
  ./configure --with-php-config=$(which php-config) &&
  make -j4
'
```

产物：`/php-build/php-8.4.25/ext/sqlquery/modules/sqlquery.so`

## 部署到测试容器

```bash
docker cp /home/jmliang/php-build/php-8.4.25/ext/sqlquery/modules/sqlquery.so \
  0fe828e90292:/usr/local/lib/php/extensions/no-debug-non-zts-20240924/sqlquery.so

docker exec --user root 0fe828e90292 bash -lc '
  echo extension=sqlquery.so > /usr/local/etc/php/conf.d/zz-sqlquery.ini
  apache2ctl graceful
'
```

## 验证

```bash
docker exec --user root 0fe828e90292 bash -lc '
  printf "%s" "{\"file\":\"/app/xvwa/vulnerabilities/stored_xss/home.php\",\"line\":56,\"output\":\"result_line56.json\"}" > /tmp/sql_query.json
  curl -s -o /dev/null http://127.0.0.1/xvwa/vulnerabilities/stored_xss/index.php
  cat /tmp/sql/result_line56.json
'
```

## 写入字段提取

写入语句没有结果集，扩展额外在 SQL 层解析写入字段：

- 文本协议：解析 `INSERT` / `REPLACE` / `UPDATE` 的表、列名、值和 WHERE。
- 原生 prepared：读取 `stmt->data->param_bind`，用实际参数替换 `?`。

输出字段：

```json
"write": {
  "operation": "INSERT",
  "table": "comments",
  "columns": ["user", "comment", "date"],
  "rows": [["alice", "hello", "30 Oct 2026"]]
}
```

原生 prepared 额外带 `parameters`。解析为 best-effort，覆盖
`INSERT ... VALUES`、`INSERT ... SET`、`UPDATE ... SET`、`DELETE ... WHERE` 等常见形式。
