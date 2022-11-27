- This is the table with 20 day MA and 50 day MA

![image](https://user-images.githubusercontent.com/82328858/204157825-e2451b41-2335-4177-970c-f452ad224c2e.png)

- MASTER STOCKS

![image](https://user-images.githubusercontent.com/82328858/204157882-ffea8307-a378-4bfa-914b-816699f89874.png)

- BAJAJ2 with  close price and signal

![image](https://user-images.githubusercontent.com/82328858/204157925-d18d4aed-b9f4-4456-824a-c760b8e5c95a.png)


- BAJAJ2 for BUY
![image](./data/bajaj2.png)

- stock report

![image](https://user-images.githubusercontent.com/82328858/204158018-0e3856f7-7b08-49e4-9340-dbf60acb1ad2.png)


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

- LAG provides access to a row at a given physical offset prior to that position. 

    - means lag makes previous value accessible
```
- for the 1st 50 rows make it as `HOLD`
- when (20d-50d)>0 and previous_day's (20d-50d)>0 -> `HOLD`
- when (20d-50d)>0 and previous_day's (20d-50d)>0 -> `HOLD`
- when (20d-50d)=0 and previous_day's (20d-50d)=0 -> `HOLD`
------------------------------------------

-  when (20d-50d)>0 and previous_day's (20d-50d)<0 -> `BUY` -> `Golden cross`

![image](./data/inner_join1.png)
    - we can see that 


- when (20d-50d)<0 and previous_day's (20d-50d)>0 -> `SELL` -> `Death Cross`

![image](./data/sell.png)

you can retrieve the table using the below query
```

select stk_date,"20 Day MA","20 Day MA","20 Day MA"-"50 Day MA" diff,bajaj1."Close Price" price,signal
FROM bajaj1 
inner JOIN bajaj2
USING(stk_date)

```