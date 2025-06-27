## 📌 Business Problem
There is a Data of List of Users and the Login-IN details.

### Dataset Sample
**Logins *

![image](https://github.com/user-attachments/assets/ad6c6004-96e8-4a07-a0b8-a493c5945783)

**Users *

![image](https://github.com/user-attachments/assets/ebe282b7-9ddc-404b-a7b1-f45719e37858)

1. Management wants to see all the users that didnot login in the past 5 months

````sql
select *
from users where not exists
(
  select *
  from logins
  where cast(LOGIN_TIMESTAMP as date) between  cast(datetrunc(month,DATEADD(MONTH,-3,'2024-06-28 15:45:00.000')) as date) and '2024-06-28'
  and users.USER_ID=logins.USER_ID
)

````
![image](https://github.com/user-attachments/assets/54001810-1351-428d-944c-c083c2c2594d)

2. For Business Unit Quaterly Analysis ,calculate how many users and how many sessions were at each Quarter

````sql
select
cast(datetrunc(QUARTER,LOGIN_TIMESTAMP) as date) as First_day_of_the_Quater,
DATEPART(QUARTER,login_timestamp) as Quater,
count(distinct USER_ID) as user_cnt,
count(SESSION_ID) as session_cnt
from logins
group by cast(datetrunc(QUARTER,LOGIN_TIMESTAMP) as date),DATEPART(QUARTER,login_timestamp)
order by cast(datetrunc(QUARTER,LOGIN_TIMESTAMP) as date) desc

````

![image](https://github.com/user-attachments/assets/277daf9b-45f1-4db7-8479-ed9451a1ee08)

3. Display User_id that Login in Jan 2024 and didnot login in Nov 2023

````sql
select distinct l1.USER_ID
from logins as l1
where DATEPART(MONTH,LOGIN_TIMESTAMP)=1 and DATEPART(YEAR,LOGIN_TIMESTAMP)=2024 and not exists
(
  select USER_ID
  from logins as l2
  where DATEPART(MONTH,LOGIN_TIMESTAMP)=11 and DATEPART(YEAR,LOGIN_TIMESTAMP)=2023
  and l1.USER_ID=l2.USER_ID
)

````

4. Add from 2 % Change in sessions from last Quater

````sql
select *,
cast((session_cnt-lag_session_cnt)*100.0/lag_session_cnt as decimal(10,2)) as Per_Change
from
(
  select *,
  lag(session_cnt) over(order by First_day_of_the_Quater) as lag_session_cnt
  from
  (
    select cast(datetrunc(QUARTER,LOGIN_TIMESTAMP) as date) as First_day_of_the_Quater,
    DATEPART(QUARTER,login_timestamp) as Quater,
    count(distinct USER_ID) as user_cnt,
    count(SESSION_ID) as session_cnt
    from logins
    group by cast(datetrunc(QUARTER,LOGIN_TIMESTAMP) as date),DATEPART(QUARTER,login_timestamp)
    --order by cast(datetrunc(QUARTER,LOGIN_TIMESTAMP) as date) desc
  )base
)base

````
5. Display The USer That had the highest session score for each day

````sql
select *
from
(
  select *,
  ROW_NUMBER() over(partition by date_ order by seesion_score desc) as rw
  from
  (
    select
    cast(login_timestamp as date) as date_,
    USER_ID,
    sum(SESSION_SCORE) as seesion_score
    from logins
    group by cast(login_timestamp as date),USER_ID
  )base
)base
where rw=1

````
6. Return Users that had a session on every single day since their first login

````sql
select *
from
(
  select USER_ID,count(distinct cast(login_timestamp as date)) as No_of_days,
  DATEDIFF(DAY,min(login_timestamp),max(login_timestamp))+1 as Required_days
  from logins
  group by USER_ID
)base
where No_of_days=Required_days
;

````

7. On What Date there were NO LOGIN at all

````sql
with cte as
(
select 
cast(min(login_timestamp) as date) as min_date,
cast(max(login_timestamp) as date) as max_date
from logins
),
cte1 as
(
select min_date ,max_date
from cte
union all
select dateadd(day,1,min_date) as min_date,max_date
from cte1
where min_date<=max_date
)

select * from 
cte1 as c where not exists
(
select cast((login_timestamp) as date)
from logins l 
where c.min_date=cast((l.login_timestamp) as date)
)
Option (MAXRECURSION 1000)

````
