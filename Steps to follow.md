- first we import the csv data into oracle database in 6 different tables namely:
    - bajaj
    - eicher
    - hero
    - infosys
    - tcs
    - tvs
<br></br>
- Inorder to import data we have to create the tables.In this project we create one table for each stock.
- The strategy in this project is  using MA for closing price.So our table for each stock has only two columns `stk_date` and `close`

- creating table
    ```
    create  table EICHER(
    stk_date date,
    close number(10,2));
    ```
- the above is the standart sql for creating table, but if we have 100 different stock we have to replace the table name for every stock.Now we are using PL/SQL. so we shall try to do this table creation in a smarter way using loops.

- for that we shall create a procedure called `create_stock_table`
```
create or replace procedure create_stock_table(table_name in varchar2)
is
begin
    EXECUTE IMMEDIATE 'CREATE TABLE '||table_name||' (stk_date date,close number(10,2))';
end;
```
- how to execute the procedure we created above ?

```
begin
    create_stock_table('bajaj');
end;

```
- as simple as this

- now we shall create the 6 tables using loop.
```
declare
my_array sys.dbms_debug_vc2coll:= sys.dbms_debug_vc2coll('hero', 'bajaj', 'eicher', 'infosys','tvs','tcs');
begin
  for r in my_array.first..my_array.last
        loop
            create_stock_table(my_array(r));
       end loop;
       commit;
    end;
```
- Next step is to calculate moving average for the data we got.So we will create a new table as <Stock1> like bajaj1,infosys1,etc... with MA for 20 and 50 days

```
CREATE TABLE bajaj1 as (
SELECT stk_date, ROUND(Close,2)  "Close Price", 
ROUND(AVG(Close) OVER(ORDER BY stk_date ROWS 19 PRECEDING),2)  "20 Day MA",
ROUND(AVG(Close) OVER(ORDER BY stk_date ROWS 49 PRECEDING),2)  "50 Day MA"
FROM bajaj);
```
- Therefore the procedure to  create such a table is

```
create or replace procedure create_stock_table_with_ma(table_name in varchar2)
is
stmt varchar2(1000);
begin
    stmt:='CREATE TABLE '||table_name||'1 as (SELECT stk_date, ROUND(Close,2)  "Close Price", 
ROUND(AVG(Close) OVER(ORDER BY stk_date ROWS 19 PRECEDING),2)  "20 Day MA",
ROUND(AVG(Close) OVER(ORDER BY stk_date ROWS 49 PRECEDING),2)  "50 Day MA"
FROM '||table_name||')';
 execute immediate stmt;
    dbms_output.put_line('CREATE TABLE '||table_name||'1 (SELECT stk_date, ROUND(Close,2)  "Close Price", 
ROUND(AVG(Close) OVER(ORDER BY stk_date ROWS 19 PRECEDING),2)  "20 Day MA",
ROUND(AVG(Close) OVER(ORDER BY stk_date ROWS 49 PRECEDING),2)  "50 Day MA"
FROM '||table_name||'');
end;
```
- how to execute the procedure we created above ?

```
begin
    create_stock_table_with_ma('bajaj');
end;
```
- use loop to call the procedure for all the tables
```
declare
my_array sys.dbms_debug_vc2coll:= sys.dbms_debug_vc2coll('hero', 'bajaj', 'eicher', 'infosys','tvs','tcs');
begin
  for r in my_array.first..my_array.last
        loop
            create_stock_table_with_ma(my_array(r));
       end loop;
       commit;
    end;
```
- now lets unify all the tables

```
CREATE TABLE master_stocks AS
SELECT stk_date ,bajaj.Close  Bajaj, eicher.Close Eicher,
hero.Close Hero, infosys.Close Infosys, 
tcs.Close TCS, tvs.Close TVS
FROM bajaj
INNER JOIN eicher
USING(stk_date)
INNER JOIN hero
USING(stk_date)
INNER JOIN infosys
USING(stk_date)
INNER JOIN tcs
USING(stk_date)
INNER JOIN tvs
USING(stk_date);
```
- now lets create a table with signal whether to `buy/hold/sell` based on the MA values

```
create or replace procedure create_stock_table_with_signal(table_name in varchar2)
is
stmt varchar2(1000);
begin
stmt:='CREATE TABLE '||table_name||'2 AS (
SELECT stk_date, "Close Price",
(CASE	
	WHEN ROW_NUMBER() OVER(order by stk_date) <=50 THEN ''Hold''
	WHEN ("20 Day MA" - "50 Day MA" > 0 AND LAG("20 Day MA" - "50 Day MA") OVER(order by stk_date) > 0 ) THEN ''Hold''
    WHEN ("20 Day MA" - "50 Day MA" < 0 AND LAG("20 Day MA" - "50 Day MA") OVER(order by stk_date) < 0 ) THEN ''Hold''
    WHEN ("20 Day MA" - "50 Day MA" = 0 AND LAG("20 Day MA" - "50 Day MA") OVER(order by stk_date) = 0 ) THEN ''Hold''
    WHEN ("20 Day MA" - "50 Day MA" > 0 AND LAG("20 Day MA" - "50 Day MA") OVER(order by stk_date) < 0 ) THEN ''Buy''
    WHEN ("20 Day MA" - "50 Day MA" < 0 AND LAG("20 Day MA" - "50 Day MA") OVER(order by stk_date) > 0 ) THEN ''Sell''
END ) Signal
from '||table_name||'1)';
    dbms_output.put_line(stmt);
 execute immediate stmt;
end;

```
- how to call the procedure?
```
begin
    create_stock_table_with_signal('bajaj');
end;
```

- lets iterate and do the same for all the stocks

```
declare
my_array sys.dbms_debug_vc2coll:= sys.dbms_debug_vc2coll('hero', 'bajaj', 'eicher', 'infosys','tvs','tcs');
begin
  for r in my_array.first..my_array.last
        loop
            create_stock_table_with_signal(my_array(r));
       end loop;
       commit;
    end;
```

- we have populated the table <stock>2 with signal as buy/sell/hold
-