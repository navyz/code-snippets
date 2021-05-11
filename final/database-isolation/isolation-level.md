For the sake of my rememberable

1. Transaction

I'm not sure about you but i got confused when i first read the below sentence

"A phantom read occurs when, in the course of a transaction, new rows are added or removed by another transaction to the records being read."

The confusion is that: what happend for a "non-transaction" thing. Why peope keep talking about transaction but not consider the interaction between a transaction and a non-transaction; or even between 2 non transactions?

Finally, i understand.

"For database, every operation is transactional". Even if we write a simple sql statement "select * from a-table" and not start any transaction, a transaction will be create for you. It's called "auto commit".
There is a term of non-transactional, but that for the application world, when you don't want to group a bunch of operations to live and die together.

2. Isolation level

Isolation level to define the interaction behavior between two or more CONCURRENT transactions.

People keep taking about database isolation level all day long, and i still not sure how to do that

Assumps that i have two transactions run concurrently, one is read-only, and the other insert new records . Which isolation level should i put for each transaction to make sure it behave as i expect?

For example, if i put the SERIALIZE for my write transaction (which not allow phantom read), and READ-COMMMITED (allow phantom read) for my read transaction, can i see the phantom records?

To make it clear, only can test it out myself. Luckly, they invent a greate tool called docker, which is very good tool to do this.


3. Preparation
- I am using postgresql. The other database engine may behave differently. The basic concept should stay i belive.
- All the below steps are done by command line. So just copy and run
- To begin, you just need to install docker and a internet connection, then it's good to go

4. Preparation steps
- install docker
- pull postgres docker image: docker pull postgres
- run: docker run --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword -d postgres
5. Connect to docker and run psql: 
  + docker ps
  + docker exec -it <container_id> bash
  + psql -h localhost -p 5432 -U postgres -W
  + enter password: mysecretpassword


6. Scenario 1: dirty read, non-repeatable read and phantom read
a. prepare
- open 4 terminal windows, call t1, t2, t3, t4, t5 (execute step-5 four times)
- in t1, create a new table
CREATE TABLE books (
  id              SERIAL PRIMARY KEY,
  title           VARCHAR(100) NOT NULL,
  year  		int
);

- insert one sample record

insert into books (title, year) values ('Book1', 2001);
insert into books (title, year) values ('Book2', 2002);

b. Verify

i.
in t1, t2, t3, t4, execute the below sql statement respectively
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED ;
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ ;
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE ;

ii.
in t1, t2, t3, t4, execute the below sql statement
select * from books

2 records should be return

iii.
in t5, execute the below statement

BEGIN;
update books set title='Book1-A' where year = 2001;
delete books where year = 2002; insert into books ('title', 'year') values ('Book3', '2003' 


do not commit yet, come back to t1, t2, t3, t4 and try to select again

select * from books;

notice that there are no change in the result since the transaction in step iii have not commit yet. Mean, "dirty read" is not allowed in postgresql, regardless of whatlevel of transaction

iv.
now come back to t5 and run
COMMIT;

select * from books;

You should see the latest update are reflected.


v.

Let go back to t1, t2, t3, t4 and execute the select statement
select * from books;

Notice that the updated/deleted/inserted records are reflected for t1 & t2 but not t3 and t4
this prove that "non-repeatable read" and "phantom-read" are not posible for isolation level "REPEATABLE READ" and "SERIALIZE"

7. Scenario 2: serialization anomaly

Follow step 5, open two psql terminal windows

In one of the terminal, prepare the data


CREATE TABLE mytable (
  id              SERIAL PRIMARY KEY,
  class           int NOT NULL,
  value  		  int NOT NULL
);

insert into mytable (class, value) values (1, 10);
insert into mytable (class, value) values (1, 20);
insert into mytable (class, value) values (2, 100);
insert into mytable (class, value) values (2, 200);


in t1
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE ;
int2
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE ;
back to t1
select sum(value) from mytable where class = 1;
back to t2
select sum(value) from mytable where class = 2;
back to t1
insert into mytable (class, value) values (2, 300);
back to t2
insert into mytable (class, value) values (1, 30);
back to t1

Now, try to commit in both t1 and t2. Notice that only the first one will be success, the second one will be failed because it violate the constraint of serialization isolation 

A serializable execution is defined to be an execution of the operations of concurrently executing SQL-transactions that produces the same effect as some serial execution of those same SQL-transactions. A serial execution is one in which each SQL-transaction executes to completion before the next SQL-transaction begins.

let say we commit t1 first, and then t2. t1 will success because since it start transaction, there is no change commited yet. t2 is a bit tricky, it saw there is changes, it it try to assump if the transaction is execute serially, there are two posibality
- t1 > t2: t2 will behave differently (sum=600) than as the current situaation (sum = 300)
- t2 > t1: t1 will behave differently (sum=60) than as the current situation (sum = 30)
since there is no order comply, t2 must failed


Now, we try the same experiment, only change the isolation level of t1 to REPEATABLE READ, and keep t2 as SERIALIZABLE as below. There are two scenarios:

- If we commit t1 first, and then t2.
  + t1 will be succcess for sure, because there is no changes committed since it started transaction
  + t2 is SERIALIABLE, it will try to calculate all the posible order
     t1 > t2: t2 will behave differently (sum=600) than the current situation (sum=300)
     t2 > t1: t2 will behave the same as of now (sum=300). t1 will behave differently, but who care! it's repeatable read.
  + since there is one acceptable order solution for t2, it will commit successfully.

- If we commit t2 first, and then t1. 
  + t2 will be succcess for sure, because there is no changes committed since it started transaction
  + t1 will also be success because it's repeatable read. No need to care must a about serial order consideration

 






 





