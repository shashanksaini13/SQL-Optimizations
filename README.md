# README

This README includes a guide I wrote on how to optimize PostrgreSQL query performance for Army MUGS (electric sensor) units, a description of my API and its uses, and my query performance benchmarking script for MUGS units. Unfortunately most code/files I wrote cannot be included, as many are classified government property. Code for my API and an example on how to use my sampling algorithm are included in the repository, however.

## Database Optimization:

### Step 1: Configure work_mem and shared_buffers

The work\_mem parameter in PostgreSQL determines the base maximum amount of memory to be used by a query operation. Once this base amount is exceeded, temporary disk files must be made for the rest of the operation to proceed. These disk files are slow and inefficient, and should theoretically never be made if work\_mem is configured correctly. The default value for work_mem is 4MB, the change recommended is 100MB. 

The shared\_buffers parameter in PostgreSQL sets the amount of memory the database server uses for shared memory buffers. In general, it determines transactions per second per user when multiple users are connected to the database at once. The recommended value for shared_buffers is 25% of the available memory in the MUGS unit, which is around 256MB.

Connect to the MUGS unit in your terminal, and open the postgresql.conf file. One way to do this is run:

`locate postgesql.conf`

You will see the path to this file. Use an editor such as VIM to open it. Once open, set the work\_mem parameter to '100MB' and the shared_buffers parameter to '256MB' and save. Next, you will need to restart the PSQL server with these two commands:

`sudo systemctl stop postgresql`
`sudo systemctl start postgresql`

In order to check if work_mem was properly configured, enter the PSQL command line with:

`psql`

Once connected, enter:

`SHOW work_mem;`

If "100MB" is not returned, the configuration has not succeeded. 

### Step 2: Create indices

In order to access data in the database, PostgreSQL performs index scans, which means that in each row retrieval, data must be fetched from both the index and the heap. Accessing the heap is slow and may be random, thus should be limited. To solve this problem, index-only scans should be implemented, which do not require heap access. First, enter the PSQL command line and main database using

`psql`
`\c main`

Next, indices must be created on commonly ran queries. I have provided six example keys. Within PSQL, enter:

`CREATE UNIQUE INDEX phasea_time_frequency_key ON public.phasea USING btree("time, frequency);`
`CREATE UNIQUE INDEX phaseb_time_frequency_key ON public.phaseb USING btree("time, frequency);`
`CREATE UNIQUE INDEX phasec_time_frequency_key ON public.phasec USING btree("time, frequency);`
`CREATE UNIQUE INDEX phasea_time_powerfactor_key ON public.phasea USING btree("time, power_factor);`
`CREATE UNIQUE INDEX phaseb_time_powerfactor_key ON public.phaseb USING btree("time, power_factor);`
`CREATE UNIQUE INDEX phasec_time_powerfactor_key ON public.phasec USING btree("time, power_factor);`

If different common queries are ran, visit postgresql.org/docs/current/indexes-index-only-scans.html for more information on how to create these indexes.

### Step 3: Tune autovacuum settings

Vacuuming in postgresql removes dead tuples and eliminates the need to check whether indexes (from step 2) allign with their values in the heap. As stated previously, checking the heap is very slow. Aggressive autovacuum settings fix this. 

Connect to the MUGS unit in your terminal, and open the postgresql.conf file. One way to do this is run:

`locate postgesql.conf`

You will see the path to this file. Use an editor such as VIM to open it. Set these parameters:

`autovacuum_vacuum_cost_delay = 10ms`
`autovacuum_vacuum_cost_limit = 1000`
`autovacuum_vacuum_threshold = 25`
`autovacuum_vacuum_scale_factor = 0.1`
`autovacuum_analyze_threshold = 10`
`autovacuum_analyze_scale_factor = 0.05`
`autovacuum_max_workers = 6`

If queries are still running slow, run

`vacuum;`

in the PSQL command line and wait.

## Description of API:

This API is used to make interaction with the database easy. Description of methods:

`set_conn(con):`
Pass in a database connection, and the API will set its connection to it. For example,
`set_conn("postgres://postgres:postgres@localhost:5432")`

`get_db():`
Returns an array of database names available to connect to.

`set_db(name):`
Pass in a database name, and the API will connect to it. For example,
`set_db("main")`

`get_tables:`
Returns an array of names of tables in the selected database.

`get_fields(table):`
Pass in a table name, and the API will return an array of fields (column names) in the table. For example,
`get_fields("phasea")`

`db_retrieve(table, field, start, end, auto = True, reso = 0):`
Pass in a table name, field, start time, and end time. The API will write a PostgreSQL query and run it, and return the data requested. Optionally pass in a Boolean value (True = Automatic resolution, False = User chooses resolution). If False is passed in, a value for reso must be passed in (resolution of data). For example,
`db_retrieve('phasea', 'frequency', '2022-06-01T18:45:59.879Z', '2022-06-15T19:45:59.879Z')`
`db_retrieve('phasea', 'frequency', '2022-06-01T18:45:59.879Z', '2022-06-15T19:45:59.879Z', False, '3600.000s')`

Automatic resolution will sample the data with equal distribution and choose an averaging interval automatically. 50 points will be generated.

## Benchmarking script:

commascript.py and humanscript.py are two benchmarking scripts for MUGS units with similar purposes. commascript.py request the user for a table name, data type, and number of days for the query to range. It then executes the query and writes Execution Time, Preparation Time, CPU1 and CPU2 Usage, RAM Usage and Row Count to a CSV file. humanscript.py does the same, except writes to a text file which is easier to read. Run new.py before running commascript.py to reset the CSV file. 
