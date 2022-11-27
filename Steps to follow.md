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
- the above is the standard sql for creating table, but if we have 100 different stock we have to replace the table name for every stock.Now we are using PL/SQL. so we shall try to do this table creation in a smarter way using loops.

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
- once the table is created , we have to import data, by following steps
- ![image](https://user-images.githubusercontent.com/82328858/204153388-fba6bc64-7812-490b-b93c-23d51a6277ab.png)
- ![image](https://user-images.githubusercontent.com/82328858/204153406-54e27d3d-b0b4-47a4-91d5-7f5cbb3df69b.png)
- ![image](https://user-images.githubusercontent.com/82328858/204153440-85753c21-9a59-4a7e-b5a2-1c03cfc515da.png)
- ![image](https://user-images.githubusercontent.com/82328858/204153459-0b1a47fd-49c1-48cc-961c-1e5d7fa96f97.png)
- ![image](https://user-images.githubusercontent.com/82328858/204153470-cf6825b3-b62c-4bea-b718-fc0817f21a77.png)
- ![image](https://user-images.githubusercontent.com/82328858/204153490-17d34ccd-fc3e-45cc-a8b9-e3c7184d343f.png)
- ![image](https://user-images.githubusercontent.com/82328858/204153506-91a9bf58-ac72-4b5d-b928-628685144322.png)
- ![image](https://user-images.githubusercontent.com/82328858/204153520-9c0d57e7-456c-4849-80ff-734a84c02438.png)
- ![image](https://user-images.githubusercontent.com/82328858/204153530-189df053-6a53-4677-a57e-41a825c81406.png)

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
