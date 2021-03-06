# clickhousedb_fdw - ClickHouse Foreign Data Wrapper for PostgreSQL
The clickhousedb_fdw is open-source (GPLv2 licensed). It is a Foreign Data Wrapper (FDW) for one of the fastest column store databases; "Clickhouse". This FDW allows you to SELECT from, and INSERT into, a ClickHouse database from within a PostgreSQL v11 server.

The FDW supports advanced features like aggregate pushdown and joins pushdown. These significantly improve performance by utilizing the remote server’s resources for these resource intensive operations.


##### 1. Supported PostgreSQL Versions.
PostgreSQL Version 11 [1].

##### 2. Installation
###### Prerequisite
The clickhousedb_fdw uses an ODBC driver provided by ClickHouse[2]. This driver can be downloaded from the ClickHouse site[3]. This driver requires unixodbc[4] which can be installed on any *nix based operating system using the platform’s package manager.   

###### Installing clickhousedb_fdw
There are two ways to install the clickhousedb_fdw. The first is by downloading the clickhousedb_fdw source code and compiling it. The second option is to download the deb or rpm packages. 

###### Download and compile
The latest release of clickhousedb_fdw can be downloaded from this GitHub page: 

      https://github.com/Percona/clickhousedb_fdw/releases

or it can be downloaded using the git: 

     git clone https://github.com/Percona/clickhousedb_fdw.git
     
After downloading the code, set the path for the PostgreSQL binary: 

    cd clickhousedb_fdw 
    make USE_PGXS=1
		make USE_PGXS=1 install


###### Install using deb / rpm packages.
Sudo yum install clickhousedb_fdw
Sudo apt-get install clickhousedb_fdw



##### Usage
You need to set up the sample database and tables in the ClickHouse database. Here we create a sample database name test_database and two sample tables tax_bills_nyc and tax_bills: 
 
    CREATE DATABASE test_database;

    USE test_database;


    CREATE TABLE tax_bills_nyc 
    (
        bbl Int64,
        owner_name String,
        address String,
        tax_class String,
        tax_rate String,
        emv Float64,
        tbea Float64,
        bav Float64,
        tba String,
        property_tax String,
        condonumber String,
        condo String,
        insertion_date DateTime MATERIALIZED now() 
    )
    ENGINE = MergeTree PARTITION BY tax_class ORDER BY (owner_name)


Download the sample data from the taxbills.nyc website and put the data in the table:

    curl -X GET 'http://taxbills.nyc/tax_bills_june15_bbls.csv' | clickhouse-client --input_format_allow_errors_num=10 --query="INSERT INTO test_database.tax_bills_nyc FORMAT CSV"
    
    CREATE TABLE tax_bills
    (
        bbl bigint, 
        owner_name text
    )
    ENGINE = MergeTree
    PARTITION BY bbl
    ORDER BY bbl;

##### PostgreSQL
Now the data is ready in the ClickHouse, the next step is to set up the PostgreSQL side. We need to create a ClickHouse foreign server, user mapping, and foreign tables.

    CREATE SERVER clickhouse_svr FOREIGN DATA WRAPPER clickhousedb_fdw OPTIONS(dbname 'test_database', driver '/home/vagrant/percona/clickhousedb_fdw/lib/clickhouse-odbc/driver/libclickhouseodbc.so', host '127.0.0.1');
    
    CREATE USER MAPPING FOR CURRENT_USER SERVER clickhouse_svr;
    postgres=# CREATE FOREIGN TABLE tax_bills_nyc 
    (
        bbl int8,
        owner_name text,
        address text,
        tax_class text,
        tax_rate text,
        emv Float,
        tbea Float,
        bav Float,
        tba text,
        property_tax text,
        condonumber text,
        condo text,
        insertion_date Time 
    ) SERVER clickhouse_svr;

    postgres=# SELECT bbl,tbea,bav,insertion_date FROM tax_bills_nyc LIMIT 5;
        bbl     | tbea  |  bav   | insertion_date 
    ------------+-------+--------+----------------
    4001940057 | 18755 | 145899 | 15:25:42
    1016830130 |  2216 |  17238 | 15:25:42
    4012850059 | 69562 | 541125 | 15:25:42
    1006130061 | 55883 | 434719 | 15:25:42
    3033540009 | 33100 | 257490 | 15:25:42
    (5 rows)
    
    CREATE TABLE tax_bills ( bbl bigint, owner_name text) ENGINE = MergeTree PARTITION BY bbl ORDER BY (bbl)
    
    INSERT INTO tax_bills SELECT bbl, tbea from tax_bills_nyc LIMIT 100;
    
    EXPLAIN VERBOSE SELECT bbl,tbea,bav,insertion_date FROM tax_bills_nyc LIMIT 5;
                                         QUERY PLAN                                         
    --------------------------------------------------------------------------------------------
    Limit  (cost=0.00..0.00 rows=1 width=32)
    Output: bbl, tbea, bav, insertion_date
     ->  Foreign Scan on public.tax_bills_nyc  (cost=0.00..0.00 rows=0 width=32)
             Output: bbl, tbea, bav, insertion_date
             Remote SQL: SELECT bbl, tbea, bav, insertion_date FROM test_database.tax_bills_nyc
    (5 rows)

#### Aggregate Pushdown.


Aggregate pushdown is a new feature of PostgreSQL FDW. There are currently very few foreign data wrappers that support aggregate pushdown: clickhousedb_fdw is one of them. Planner decides which aggregate to pushdown or not. Here is an example for both cases.

    postgres=# EXPLAIN VERBOSE SELECT count(bbl) FROM tax_bills_nyc LIMIT 5;
                                    QUERY PLAN                                                                
    ---------------------------------------------------------------------------
    Limit  (cost=0.00..0.01 rows=1 width=8)
    Output: (count(bbl))
    ->  Aggregate  (cost=0.00..0.01 rows=1 width=8)
             Output: count(bbl)
             ->  Foreign Scan on public.tax_bills_nyc  (cost=0.00..0.00 rows=0 width=8)
                Output: bbl, owner_name, address, tax_class, tax_rate, emv, tbea, bav, tba, property_tax, condonumber, condo, insertion_date
                   Remote SQL: SELECT bbl FROM test_database.tax_bills_nyc
    (7 rows)
    

        EXPLAIN VERBOSE SELECT count(bbl) FROM tax_bills_nyc;
                                QUERY PLAN                            
    ------------------------------------------------------------------
    Foreign Scan  (cost=1.00..-1.00 rows=1000 width=8)
    Output: (count(bbl))
    Relations: Aggregate on (tax_bills_nyc)
    Remote SQL: SELECT count(bbl) FROM test_database.tax_bills_nyc
    (4 rows)



###### Join Pushdown
Join pushdown is also a very new feature of PostgreSQL FDW’s. The clickhousedb_fdw also supports join pushdown.

    EXPLAIN VERBOSE SELECT t2.bbl, t2.owner_name, t1.bav FROM tax_bills_nyc t1 RIGHT OUTER JOIN tax_bills t2 ON (t1.bbl = t2.bbl);
                                                                        QUERY PLAN                                                                     
    
    -------------------------------------------------------------------------------------------------------------------------------------------------------
    ----
    Foreign Scan  (cost=1.00..-1.00 rows=1000 width=50)
    Output: t2.bbl, t2.owner_name, t1.bav
    Relations: (tax_bills t2) LEFT JOIN (tax_bills_nyc t1)
    Remote SQL: SELECT r2.bbl, r2.owner_name, r1.bav FROM  test_database.tax_bills r2 ALL LEFT JOIN test_database.tax_bills_nyc r1 ON (((r1.bbl = r2.bbl
    )))
    (4 rows)
    
 ##### Limitations
There are some limitations and ToDos. In some instances, more research is needed to implement cases such as these:

    1 - All configuration parameters of ClickHouse database.
    2 - Complex join pushdown support.
    3 - Delete from a ClickHouse partition


[1]: https://www.postgresql.org/
[2]: http://www.clickhouse.com
[3]: https://github.com/yandex/clickhouse-odbc/releases
[4]: https://github.com/percona/clickhousedb_fdw/issues/new
[5]: CONTRIBUTING.md


