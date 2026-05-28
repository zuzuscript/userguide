# Chapter 19: Tabular Data with Databases and CSV

Chapter 18 packaged reusable code so other people could install it.

Now we look at a common kind of data those tools often handle: tables.
This chapter is about rows, columns, imports, exports, and queries.

Zia the sleepy raccoon coder wants to keep track of coffee shops. The
important facts are:

- the shop name,
- price,
- coffee quality,
- coziness,
- vibes,
- and whether the shop allows naps.

That is naturally tabular data. It can live in a SQL database, a CSV file,
or both.

ZuzuScript gives you two standard tools for this:

- `std/db` for SQL databases,
- `std/data/csv` for CSV, TSV, and other delimited text files.

Database examples require a runtime with database support. File-backed CSV
examples also require filesystem support. In a browser, in-memory CSV
parsing may be available even when local file access is not.


## 19.1 The coffee shop table

The examples in this chapter use one table:

```text
coffee_shops
  id            integer primary key
  name          text
  neighbourhood text
  price         integer
  coffee        integer
  coziness      integer
  vibes         integer
  allows_naps   integer
```

The ranking columns use small integers. Higher is better.

For broad SQL portability, `allows_naps` is stored as an integer:

- `1` means true,
- `0` means false.

The code can still treat it as a Boolean-style flag because ZuzuScript
treats the number 0 as false and all other numbers as true.


## 19.2 Temporary SQLite databases

Import `DB` from `std/db`:

```zzs
from std/db import DB;
```

Use `DB.temp()` when you want a temporary in-memory SQLite database. It is
excellent for examples, tests, scratch scripts, and data transformations
that do not need to survive after the process exits.

```zzs
from std/db import DB;

let dbh := DB.temp();

dbh.prepare("""
	create table coffee_shops (
		id integer primary key,
		name text not null,
		neighbourhood text,
		price integer,
		coffee integer,
		coziness integer,
		vibes integer,
		allows_naps integer
	)
""").execute();
```

`prepare(sql)` returns a statement handle. `execute()` runs the prepared
statement.

Use placeholders for values:

```zzs
let insert := dbh.prepare("""
	insert into coffee_shops
	(id, name, neighbourhood, price, coffee, coziness, vibes, allows_naps)
	values (?, ?, ?, ?, ?, ?, ?, ?)
""");

insert.execute( 1, "Moonbean", "North Arcade", 3, 5, 4, 5, 1 );
insert.execute( 2, "Steam & Syntax", "Old Station", 4, 5, 3, 4, 0 );
insert.execute( 3, "Hibernate House", "Canal Walk", 2, 3, 5, 4, 1 );
```

Placeholders keep data separate from SQL syntax. Prefer them over building
SQL by concatenating user-provided strings.


## 19.3 Querying rows

To read rows, prepare a `select`, execute it, and fetch the results.

```zzs
let query := dbh.prepare("""
	select name, price, coffee, coziness, vibes
	from coffee_shops
	where allows_naps = ?
	order by coffee desc, coziness desc, price asc
""");

query.execute(1);

for ( const row in query ) {
	say `${row{name}}: coffee ${row{coffee}}, coziness ${row{coziness}}`;
}
```

A statement can be used directly in a `for` loop. The loop receives
dictionary rows.

If you want all remaining rows at once, use `all_dict()`:

```zzs
let best := dbh.prepare("""
	select name, neighbourhood
	from coffee_shops
	where allows_naps = ?
	order by vibes desc, coffee desc
""");

best.execute(1);
let rows := best.all_dict();

say rows[0]{name};
```

Use typed fetches when you want numeric-looking database values coerced
back to numbers:

```zzs
let scores := dbh.prepare("""
	select name, price, coffee, allows_naps
	from coffee_shops
	order by id
""");

scores.execute();
let typed_rows := scores.all_typed_dict();

say typed_rows[0]{coffee} + 1;
```

The typed methods include:

- `next_typed_array()`,
- `next_typed_dict()`,
- `all_typed_array()`,
- and `all_typed_dict()`.


## 19.4 Updating and deleting rows

Updates use the same prepare-and-execute pattern.

Zia discovers that Steam & Syntax has added a nap corner:

```zzs
dbh.prepare("""
	update coffee_shops
	set coziness = ?, allows_naps = ?
	where name = ?
""").execute( 4, 1, "Steam & Syntax" );
```

Zia also decides that any shop with weak coffee and no nap policy should
not stay on the shortlist:

```zzs
dbh.prepare("""
	delete from coffee_shops
	where coffee < ? and allows_naps = ?
""").execute( 3, 0 );
```

(If she could delete them from real life too, she would.)

For several similar inserts, `execute_batch` prepares once and runs many
rows:

```zzs
dbh.execute_batch(
	"""insert into coffee_shops
		(id, name, neighbourhood, price, coffee, coziness, vibes,
		allows_naps)
		values (?, ?, ?, ?, ?, ?, ?, ?)""",
	[
		[ 4, "The Quiet Mug", "Library Lane", 3, 4, 5, 3, 1 ],
		[ 5, "Noisy Bean", "Market Square", 2, 4, 1, 2, 0 ],
	],
);
```

If a group of changes must succeed or fail together, use a transaction:

```zzs
let dbh := DB.temp( { auto_commit: true } );

dbh.prepare(
	"create table coffee_shops (id integer primary key, name text)",
).execute();

dbh.begin();
dbh.prepare(
	"insert into coffee_shops (id, name) values (?, ?)",
).execute( 1, "Moonbean" );
dbh.commit();
```

If something goes wrong before `commit()`, call `rollback()`.


## 19.5 Opening an existing SQLite database

Use `DB.open(path)` for a SQLite database file.

```zzs
from std/db import DB;
from std/io import Path;

let path := new Path("coffee-shops.sqlite");
let dbh := DB.open(path);
```

If the file already contains tables, you can query them immediately:

```zzs
let query := dbh.prepare("""
	select name, coffee, allows_naps
	from coffee_shops
	order by coffee desc
""");

query.execute();

for ( const shop in query ) {
	say shop{name};
}
```

If the file does not exist yet, SQLite creates it. That means the same API
works for first setup and for later reopening.

```zzs
let setup_db := DB.open( new Path("coffee-shops.sqlite") );

setup_db.prepare("""
	create table if not exists coffee_shops (
		id integer primary key,
		name text not null,
		neighbourhood text,
		price integer,
		coffee integer,
		coziness integer,
		vibes integer,
		allows_naps integer
	)
""").execute();
```

`DB.open` also accepts a string path:

```zzs
let reopened := DB.open("coffee-shops.sqlite");
```


## 19.6 Connecting to MySQL and PostgreSQL

Use `DB.connect(dsn)` when you are connecting through a database driver
rather than opening a SQLite file.

The DSN format is the driver's connection string. For MySQL:

```zzs
from std/db import DB;
from std/proc import Env;

let user := Env.get("COFFEE_DB_USER", "zia");
let password := Env.get("COFFEE_DB_PASSWORD", "");

let mysql := DB.connect(
	"dbi:mysql:database=coffee_tracker;"
	_ "host=127.0.0.1;"
	_ "port=3306;"
	_ `user=${user};`
	_ `password=${password}`,
);
```

For PostgreSQL:

```zzs
from std/db import DB;
from std/proc import Env;

let user := Env.get("COFFEE_DB_USER", "zia");
let password := Env.get("COFFEE_DB_PASSWORD", "");

let pg := DB.connect(
	"dbi:Pg:dbname=coffee_tracker;"
	_ "host=/var/run/postgresql;"
	_ `user=${user};`
	_ `password=${password}`,
);
```

Once connected, the normal `prepare`, `execute`, and fetch methods are the
same. This example uses the PostgreSQL handle, but the SQL also works with
the MySQL handle shown above:

```zzs
let shop_db := pg;

shop_db.prepare( "drop table if exists coffee_shops" ).execute();
shop_db.prepare("""
	create table coffee_shops (
		id integer primary key,
		name text not null,
		neighbourhood text,
		price integer,
		coffee integer,
		coziness integer,
		vibes integer,
		allows_naps integer
	)
""").execute();

shop_db.prepare("""
	insert into coffee_shops
	(id, name, neighbourhood, price, coffee, coziness, vibes, allows_naps)
	values (?, ?, ?, ?, ?, ?, ?, ?)
""").execute( 1, "Moonbean", "North Arcade", 3, 5, 4, 5, 1 );

shop_db.prepare("""
	update coffee_shops
	set allows_naps = ?
	where name = ?
""").execute( 1, "Moonbean" );

let query := shop_db.prepare("""
	select name, coffee
	from coffee_shops
	order by coffee desc
""");
query.execute();

say query.next_dict(){name};

shop_db.prepare(
	"delete from coffee_shops where name = ?",
).execute( "Moonbean" );
```

Real deployments need the relevant database driver, network or socket
access, credentials, and permissions to create or modify tables.


## 19.7 Parsing CSV with headers

Import `CSV` from `std/data/csv`:

```zzs
from std/data/csv import CSV;
```

When `headers: true`, the first row supplies dictionary keys:

```zzs
from std/data/csv import CSV;

let csv := new CSV(
	headers: true,
	types: {
		price: "integer",
		coffee: "integer",
		coziness: "integer",
		vibes: "integer",
		allows_naps: "boolean",
	},
);

let shops := csv.decode(
	"name,neighbourhood,price,coffee,coziness,vibes,allows_naps\n"
	_ "Moonbean,North Arcade,3,5,4,5,true\n"
	_ "Hibernate House,Canal Walk,2,3,5,4,true\n"
);

say shops[0]{name};
say shops[0]{coffee} + shops[0]{coziness};
```

The `types` option converts selected fields while parsing. Boolean fields
come back as Boolean-style numeric values.

CSV parsing handles quoted commas:

```zzs
let quoted := ( new CSV( headers: true ) ).decode(
	"name,neighbourhood\n"
	_ "\"Steam, Syntax & Snacks\",Old Station\n",
);

say quoted[0]{name};
```


## 19.8 Parsing CSV without headers

When there is no header row, set `headers: false`. Rows are arrays:

```zzs
let csv := new CSV(
	headers: false,
);

let rows := csv.decode(
	"Moonbean,North Arcade,3,5,4,5,1\n"
	_ "Noisy Bean,Market Square,2,4,1,2,0\n",
);

say rows[0][0];       // Moonbean
say rows[0][3];       // 5
```

If you want dictionary rows from a headerless file, provide column names:

```zzs
let csv := new CSV(
	headers: false,
	columns: [
		"name",
		"neighbourhood",
		"price",
		"coffee",
		"coziness",
		"vibes",
		"allows_naps",
	],
);

let rows := csv.decode(
	"Moonbean,North Arcade,3,5,4,5,1\n",
);

say rows[0]{neighbourhood};
```


## 19.9 Tabs, pipes, and other delimiters

CSV is often used as a general name for delimited text. The separator does
not have to be a comma.

For tab-separated values, set `sep_char` to `"\t"`:

```zzs
let tsv := new CSV(
	headers: true,
	sep_char: "\t",
);

let rows := tsv.decode(
	"name\tprice\tcoffee\tallows_naps\n"
	_ "Moonbean\t3\t5\t1\n"
	_ "Noisy Bean\t2\t4\t0\n",
);

say rows[0]{name};
```

For pipe-delimited text:

```zzs
let pipe := new CSV(
	headers: true,
	sep_char: "|",
);

let rows := pipe.decode(
	"name|price|coffee|allows_naps\n"
	_ "Hibernate House|2|3|1\n",
);

say rows[0]{allows_naps};
```

`CSV.sniff(text)` can guess a likely separator and whether the data has a
header row:

```zzs
let guess := ( new CSV() ).sniff(
	"name\tprice\n"
	_ "Moonbean\t3\n",
);

say guess{sep_char};       // tab
say guess{headers};        // likely true
```


## 19.10 Loading CSV into a database table

`std/data/csv` can import a delimited file directly into a database table.

The file APIs use `Path` objects:

```zzs
from std/data/csv import CSV;
from std/db import DB;
from std/io import Path;

let input := Path.tempfile();
input.spew_utf8(
	"name,neighbourhood,price,coffee,coziness,vibes,allows_naps\n"
	_ "Moonbean,North Arcade,3,5,4,5,1\n"
	_ "Hibernate House,Canal Walk,2,3,5,4,1\n",
);

let dbh := DB.temp();
let csv := new CSV( headers: true );

let inserted := csv.load_table(
	input,
	dbh,
	"coffee_shops",
	{
		create_table: true,
		column_types: {
			name: "text",
			neighbourhood: "text",
			price: "integer",
			coffee: "integer",
			coziness: "integer",
			vibes: "integer",
			allows_naps: "integer",
		},
	},
);

say inserted;
```

`load_table` returns the number of rows it attempted to insert.

You can also append headerless data into an existing table:

```zzs
let more := Path.tempfile();
more.spew_utf8(
	"The Quiet Mug,Library Lane,3,4,5,3,1\n"
	_ "Noisy Bean,Market Square,2,4,1,2,0\n",
);

( new CSV() ).load_table(
	more,
	dbh,
	"coffee_shops",
	{
		headers: false,
		columns: [
			"name",
			"neighbourhood",
			"price",
			"coffee",
			"coziness",
			"vibes",
			"allows_naps",
		],
	},
);
```


## 19.11 Dumping database rows to CSV

Use `dump_table` to export a whole table:

```zzs
let output := Path.tempfile();

csv.dump_table( output, dbh, "coffee_shops" );

say output.slurp_utf8();
```

Use `dump_query` to export only selected rows or columns:

```zzs
let sleepy_output := Path.tempfile();

csv.dump_query(
	sleepy_output,
	dbh,
	"""select name, neighbourhood, coffee, coziness
		from coffee_shops
		where allows_naps = ?
		order by coffee desc, coziness desc""",
	[ 1 ],
	{
		headers: true,
	},
);

say sleepy_output.slurp_utf8();
```

That gives Zia a report of shops where a coffee, a quiet corner, and a nap
are all plausible.


## 19.12 Choosing the right shape

Use SQL when you need:

- filtering and sorting without reading every row yourself,
- updates and deletes by condition,
- transactions,
- multiple related tables,
- or a durable local or server-backed store.

Use CSV when you need:

- a human-editable file,
- interchange with spreadsheets,
- quick imports or exports,
- or simple row-by-row processing.

Use both when the workflow asks for it:

1. import a CSV list of coffee shops,
2. query and update it in SQL,
3. export a ranked shortlist back to CSV.

That is the main pattern for tabular data in ZuzuScript: use `CSV` at the
edges and `DB` when you need the database to answer questions.

Tables are often local, but scripts also need to talk to remote services.
Chapter 20 moves from files and databases to the web client side: URLs,
HTTP requests, response bodies, and downloads.
