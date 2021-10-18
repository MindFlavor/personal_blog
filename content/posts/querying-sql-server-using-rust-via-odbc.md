---
title: "Querying SQLServer using Rust via ODBC"
date: 2021-05-13
draft: false
description: "A simple tutorial on how to get interact with SQL Server from a Rust application leveraging the Microsoft ODBC Driver for SQL Server"
tags: [ "sqlserver", "rust" ]
cover:
  image: /images/querying-sql-server-using-rust-via-odbc/cover.jpg
  alternate: something good
---

SQL Server is very easy to interact with in .Net thanks to ADO.Net. There is no Rust port however. So how can you interact with SQL Server from a Rust application? 
As of now, the easiest is to use the official ODBC driver (get it [here](https://docs.microsoft.com/en-us/sql/connect/odbc/download-odbc-driver-for-sql-server?view=sql-server-ver15)). It's available for every major platform so your Rust code will remain cross-platform. As for Rust, there is an excellent crate called [odbc](https://crates.io/crates/odbc) that makes working with FFI easier.

## ODBC

The steps required to query a SQL Server instance are the following:

1. Create the ODBC environment.
2. Pick a driver and connect to the SQL Server instance.
3. Create a statement.
4. Execute the statement and read the resulting result set, one row at at time.

### Create the environment

The ODBC crate makes creating the environment trivial:

```rust
let mut env = create_environment_v3().unwrap();
```

While we are at it, we can ask the environment for the list of available ODBC drivers. There is a function called `drivers()` that does just that:

```rust
let drivers = env.drivers()?; 
println!("drivers == {:?}", drivers);
```

You should see the ODBC Driver for SQL Server amongst the available drivers. If not, make sure it's properly installed following the aforementioned [link](https://docs.microsoft.com/en-us/sql/connect/odbc/download-odbc-driver-for-sql-server?view=sql-server-ver15). 

In my case I only have the ODBC Driver 17 for SQL Server installed:

```
drivers == [DriverInfo { description: "ODBC Driver 17 for SQL Server", attributes: {"Driver": "/opt/microsoft/msodbcsql17/lib64/libmsodbcsql-17.7.so.2.1", "Description": "Microsoft ODBC Driver 17 for SQL Server", "UsageCount": "1"} }]
```

Note the description part, it's the name we will pass to the ODBC connection string to bind it.

### Connect to the instance

Now that we know the driver it's just a matter of composing the connection string. [Here](https://docs.microsoft.com/en-us/sql/integration-services/import-export-data/connect-to-an-odbc-data-source-sql-server-import-and-export-wizard?view=sql-server-ver15#odbc_connstring) you can find the complete docs but, for our sakes, we just want to specify the driver, the instance and the initial catalog. Integrated authentication will take care of passing our credentials to SQL Server.

The connection string in my case will be

```
Driver={ODBC Driver 17 for SQL Server};Server=localhost;Database=tersa;Trusted_Connection=yes;
```

You might want to replace `Trusted_Connection=yes;` with `Uid=myUsername;Pwd=myPassword;` if you want to use the SQL Server authentication.

So to get a connection we just need to call the `connect_with_connection_string` function of our environment:

```rust
let conn = env.connect_with_connection_string("Driver={ODBC Driver 17 for SQL Server};Server=localhost;Database=tersa;Trusted_Connection=yes;")?;
```

> *Note*: The documentation explicitly states that our connection must not outlive the generating environment. See [https://docs.rs/odbc/0.17.0/odbc/struct.Environment.html#method.connect_with_connection_string](https://docs.rs/odbc/0.17.0/odbc/struct.Environment.html#method.connect_with_connection_string).

So far, so good! Let's move to the query!

### Create the statement

We need to create a statement, link it to our connection and execute it. 

```rust
let stmt = Statement::with_parent(&conn)?; 
let stmt_prepared = stmt.prepare("SELECT * FROM Cluster")?; 
```

### Execute the statement

Just call the, uh, `execute` function of our prepared statement:

```rust
let execution = stmt_prepared.execute()?; 
```

The `execution` is an enum that can either be `Data` or `NoData`. We need to discriminate between the two:

```rust
match execution {
    ResultSetState::NoData(_stmt) => { 
        println!("no data!");
    }
    ResultSetState::Data(_stmt) => { 
        println!("data!");
    }
}
```

> *Note*: `Data` and `NoData` are misleading. You should always receive a `Data` even if the returned recordset is empty. In other words, `NoData` does not mean "zero rows returned"!

The statement with data can now be iterated to get each row, using the `fetch` function. In this example we are printing the second column of each row, interpreted as as `String`: 

```rust
while let Some(mut cursor) = stmt.fetch()? { 
    println!("{:?}", cursor.get_data::<String>(2)?);
}
```              

Putting all together:

```rust
// Step 1.
let mut env = create_environment_v3().unwrap();

// Step 2.
let conn = env.connect_with_connection_string( 
    "Driver={ODBC Driver 17 for SQL Server};Server=localhost;Database=tersa;Uid=sa;Pwd=PasaCulo00"
)?;

// Step 3.
let stmt = Statement::with_parent(&conn)?; 
let stmt_prepared = stmt.prepare("SELECT * FROM Cluster")?; 

// Step 4.
let execution = stmt_prepared.execute()?; 

match execution {
    ResultSetState::NoData(_stmt) => { 
        println!("no data!");
    }
    ResultSetState::Data(mut stmt) => {
        println!("data!");

        while let Some(mut cursor) = stmt.fetch()? { 
            println!("{:?}", cursor.get_data::<String>(2)?);
        }
    }
}
```

## Parameters

Concatenating strings in T-SQL is dangerous: [Bobby Tables](https://xkcd.com/327/) will haunt you and make sure to screw you at the worst possible time. For this reason you can - and should - use parameters instead. 

In ODBC is very easy to pass parameters to a statement, just make sure to do it before calling `prepare`. In this example we are passing a `INT` parameter:

```rust
let stmt = odbc::Statement::with_parent(&conn)?;  

// This is our parameter (remember, the binding starts from one, not zero!)
let capacity = 400;  
let stmt = stmt.bind_parameter(1, &capacity)?;  

let stmt_prepared = stmt.prepare("INSERT INTO Cluster([Name], [Capacity]) VALUES('Trekkies', ?)")?;  

let execution = stmt_prepared.execute()?; 
```

> *Note*: `bind_parameter` takes a reference of our parameter value. To avoid making Rust's borrow checker angry it's best to bind the parameters as close to the statement execution as possible.

At this point you might be wondering why I have not parametrized the value `Trekkies` as well. There is a known issue on how the ODBC driver for Linux and macOS handles UTF-8 that makes impossible to bind string values (check out [here](https://docs.microsoft.com/en-us/sql/connect/odbc/linux-mac/known-issues-in-this-version-of-the-driver?view=sql-server-2017#known-issues)). It works in Windows so if you are only targeting Windows this issue does not apply to you.

## Connection pooling

The SQL Server ODBC Driver supports connection pooling. Connection pooling allows connection reuse, greatly decreasing the overhead of small queries. 

We can test the difference using this sample code:

```rust
use crate::odbc_safe::Odbc3;
use odbc::*;

fn execute_one(
    env: &Environment<Odbc3>,
) -> std::result::Result<(), Box<dyn std::error::Error + Send + Sync>> {
    let conn = env.connect_with_connection_string(
        "Driver={ODBC Driver 17 for SQL Server};Server=<your_instance>;Database=<your_db>;Uid=<your_user_id>;Pwd=<your_password>",
    )?;

    let stmt = Statement::with_parent(&conn)?;
    let stmt_prepared = stmt.prepare("SELECT * FROM Cluster")?;

    let execution = stmt_prepared.execute()?;

    match execution {
        ResultSetState::NoData(_stmt) => {
            panic!();
        }
        ResultSetState::Data(mut stmt) => while let Some(_cursor) = stmt.fetch()? {},
    }

    Ok(())
}

fn main() -> std::result::Result<(), Box<dyn std::error::Error + Send + Sync>> {
    const ITERATIONS: u128 = 1000;

    let env = create_environment_v3().unwrap();

    let now = std::time::Instant::now();
    for _ in 0..ITERATIONS {
        execute_one(&env)?;
    }
    println!(
        "elapsed microseconds for {} iterations = {}",
        ITERATIONS,
        now.elapsed().as_micros()
    );
    println!(
        "micros per query {}",
        now.elapsed().as_micros() / ITERATIONS
    );

    Ok(())
}
```

You can see for yourself, with a local connection it's an order of magnitude faster!

Care must be taken because the ODBC connection pooling behaves differently from the ADO.Net one. Amongst other things, ODBC connection pooling does not reset the connection when you close it (i.e. it won't call `sp_reset_connection` behind the scenes). For this reason, unless you account for it during application development, it's best to not enable it. 

In Linux/MacOS, the setting is controlled by these lines in the `odbcinst.ini`:

```ini
[ODBC]
Pooling=Yes

[ODBC Driver 17 for SQL Server]
CPTimeout=<int value>
```

## Change environment options

ODBC allows to customize the environment you are using by calling the `SQLSetEnvAttr` function ([SQLSetEnvAttr Function on Microsoft docs](https://docs.microsoft.com/en-us/sql/odbc/reference/syntax/sqlsetenvattr-function?view=sql-server-ver15)). This is useful for example if you want to change the connection pooling mode (per driver, per environment, driver aware or disable altogether).

The odbc crate does not, however, expose a safe interface for calling it. 

Fear not, FFI comes to the rescue! The odbc crate reexports the relevant function in the [`ffi` module](https://docs.rs/odbc/0.17.0/odbc/ffi/index.html). The signature is this one:

![](/images/querying-sql-server-using-rust-via-odbc/00.png)

It's definitely not Rust-y at all. The Microsoft docs tell us we need pass:

1. A valid environment handle or `NULL` if we want to be a process level change.
2. The attribute we want to change (`odbc::ffi::EnvironmentAttribute::SQL_ATTR_CONNECTION_POOLING`)
3. The value we want to set as pointer. 
4. (ignored)

In Rust it becomes unsafe:

```rust
let p_val = 1 as *mut std::ffi::c_void;
let null_handle = 0 as *mut odbc::ffi::Env;

unsafe {
    let ret = odbc::ffi::SQLSetEnvAttr(
        null_handle,
        odbc::ffi::EnvironmentAttribute::SQL_ATTR_CONNECTION_POOLING,
        p_val,
        0
    );

    println!("ret == {:?}", ret);
}
``` 

---

Happy Coding,

**Francesco**

