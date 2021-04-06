# 实验三：创建分区表

姓名：张剑峰 学号：201810513329

## 实验目的

掌握分区表的创建方法，掌握各种分区方式的使用场景。

## 实验内容

* 本实验使用 3 个表空间：USERS,USERS02,USERS03。在表空间中创建两张表：订单表(orders)与订单详表(order_details)。
* 使用你自己的账号创建本实验的表，表创建在上述 3 个分区，自定义分区策略。
* 你需要使用 system 用户给你自己的账号分配上述分区的使用权限。你需要使用 system 用户给你的用户分配可以查询执行计划的权限。
* 表创建成功后，插入数据，数据能并平均分布到各个分区。每个表的数据都应该大于 1 万行，对表进行联合查询。
* 写出插入数据的语句和查询数据的语句，并分析语句的执行计划。
* 进行分区与不分区的对比实验。

## 实验步骤

1.首先创建自己的账号 zjf_201810513329，然后以 system 身份登录:

```sql
    ALTER USER zjf_201810513329 QUOTA UNLIMITED ON USERS;
    ALTER USER zjf_201810513329 QUOTA UNLIMITED ON USERS02;
    ALTER USER zjf_201810513329 QUOTA UNLIMITED ON USERS03;
```

![执行结果](./01.png)

2.用自己的账号zjf_201810513329登录,并运行脚本文件 test3.sql:该脚本在账号下创建了两个分区表，orders（一万行数据），order_details（三万行数据）。

```sql
[student@deep02 ~]$cat test3.sql
[student@deep02 ~]$sqlplus zjf_201810513329/123@localhost/pdborcl
SQL>@test3.sql
SQL>exit
```

![执行结果](./02.png)

创建orders表代码如下：

```sql
  SQL>CREATE TABLESPACE users02 DATAFILE
  '/home/student/你的目录/pdbtest_users02_1.dbf'
    SIZE 100M AUTOEXTEND ON NEXT 50M MAXSIZE UNLIMITED，
  '/home/student/你的目录/pdbtest_users02_2.dbf' 
    SIZE 100M AUTOEXTEND ON NEXT 50M MAXSIZE UNLIMITED
  EXTENT MANAGEMENT LOCAL SEGMENT SPACE MANAGEMENT AUTO;
  
  SQL>CREATE TABLESPACE users03 DATAFILE
  '/home/student/你的目录/pdbtest_users02_1.dbf'
    SIZE 100M AUTOEXTEND ON NEXT 50M MAXSIZE UNLIMITED，
  '/home/student/你的目录/pdbtest_users02_2.dbf'
    SIZE 100M AUTOEXTEND ON NEXT 50M MAXSIZE UNLIMITED
  EXTENT MANAGEMENT LOCAL SEGMENT SPACE MANAGEMENT AUTO;
  
  SQL> CREATE TABLE orders
  (
   order_id NUMBER(10, 0) NOT NULL
   , customer_name VARCHAR2(40 BYTE) NOT NULL
   , customer_tel VARCHAR2(40 BYTE) NOT NULL
   , order_date DATE NOT NULL
   , employee_id NUMBER(6, 0) NOT NULL
   , discount NUMBER(8, 2) DEFAULT 0
   , trade_receivable NUMBER(8, 2) DEFAULT 0
  )
  TABLESPACE USERS
  PCTFREE 10 INITRANS 1
  STORAGE (   BUFFER_POOL DEFAULT )
  NOCOMPRESS NOPARALLEL
  PARTITION BY RANGE (order_date)
  
   PARTITION PARTITION_BEFORE_2016 VALUES LESS THAN (
   TO_DATE(' 2016-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
   'NLS_CALENDAR=GREGORIAN'))
   NOLOGGING
   TABLESPACE USERS
   PCTFREE 10
   INITRANS 1
   STORAGE
  (
   INITIAL 8388608
   NEXT 1048576
   MINEXTENTS 1
   MAXEXTENTS UNLIMITED
   BUFFER_POOL DEFAULT
  )
  NOCOMPRESS NO INMEMORY  
  , PARTITION PARTITION_BEFORE_2017 VALUES LESS THAN (
  TO_DATE(' 2017-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN'))
  NOLOGGING
  TABLESPACE USERS02
  ...
  );
```

创建order_details表的部分语句如下：

```sql
  SQL> CREATE TABLE order_details 
  (
  id NUMBER(10, 0) NOT NULL 
  , order_id NUMBER(10, 0) NOT NULL
  , product_id VARCHAR2(40 BYTE) NOT NULL 
  , product_num NUMBER(8, 2) NOT NULL 
  , product_price NUMBER(8, 2) NOT NULL 
  , CONSTRAINT order_details_fk1 FOREIGN KEY  (order_id)
  REFERENCES orders  (  order_id   )
  ENABLE 
  ) 
  TABLESPACE USERS 
  PCTFREE 10 INITRANS 1 
  STORAGE (   BUFFER_POOL DEFAULT ) 
  NOCOMPRESS NOPARALLEL
  PARTITION BY REFERENCE (order_details_fk1)
  (
  PARTITION PARTITION_BEFORE_2016 
  NOLOGGING 
  TABLESPACE USERS --必须指定表空间,否则会将分区存储在用户的默认表空间中
  ...
  ) 
  NOCOMPRESS NO INMEMORY, 
  PARTITION PARTITION_BEFORE_2017 
  NOLOGGING 
  TABLESPACE USERS02
  ...
  ) 
  NOCOMPRESS NO INMEMORY  
  );
```

插入数据语句：

```sql
  for i in 1..10000
    loop
      if i mod 6 =0 then
        dt:=to_date('2015-3-2','yyyy-mm-dd')+(i mod 60); --PARTITION_2015
      elsif i mod 6 =1 then
        dt:=to_date('2016-3-2','yyyy-mm-dd')+(i mod 60); --PARTITION_2016
      elsif i mod 6 =2 then
        dt:=to_date('2017-3-2','yyyy-mm-dd')+(i mod 60); --PARTITION_2017
      elsif i mod 6 =3 then
        dt:=to_date('2018-3-2','yyyy-mm-dd')+(i mod 60); --PARTITION_2018
      elsif i mod 6 =4 then
        dt:=to_date('2019-3-2','yyyy-mm-dd')+(i mod 60); --PARTITION_2019
      else
        dt:=to_date('2020-3-2','yyyy-mm-dd')+(i mod 60); --PARTITION_2020
      end if;
      V_EMPLOYEE_ID:=CASE I MOD 6 WHEN 0 THEN 11 WHEN 1 THEN 111 WHEN 2 THEN 112
                                  WHEN 3 THEN 12 WHEN 4 THEN 121 ELSE 122 END;
      --插入订单
      v_order_id:=i;
      v_name := 'aa'|| 'aa';
      v_name := 'zhang' || i;
      v_tel := '139888883' || i;
      insert /*+append*/ into ORDERS (ORDER_ID,CUSTOMER_NAME,CUSTOMER_TEL,ORDER_DATE,EMPLOYEE_ID,DISCOUNT)
        values (v_order_id,v_name,v_tel,dt,V_EMPLOYEE_ID,dbms_random.value(100,0));
      --插入订单y一个订单包括3个产品
      v:=dbms_random.value(10000,4000);
      v_name:='computer'|| (i mod 3 + 1);
      insert /*+append*/ into ORDER_DETAILS(ID,ORDER_ID,PRODUCT_NAME,PRODUCT_NUM,PRODUCT_PRICE)
        values (v_order_detail_id,v_order_id,v_name,2,v);
      v:=dbms_random.value(1000,50);
      v_name:='paper'|| (i mod 3 + 1);
      v_order_detail_id:=v_order_detail_id+1;
      insert /*+append*/ into ORDER_DETAILS(ID,ORDER_ID,PRODUCT_NAME,PRODUCT_NUM,PRODUCT_PRICE)
        values (v_order_detail_id,v_order_id,v_name,3,v);
      v:=dbms_random.value(9000,2000);
      v_name:='phone'|| (i mod 3 + 1);
  
      v_order_detail_id:=v_order_detail_id+1;
      insert /*+append*/ into ORDER_DETAILS(ID,ORDER_ID,PRODUCT_NAME,PRODUCT_NUM,PRODUCT_PRICE)
        values (v_order_detail_id,v_order_id,v_name,1,v);
      --在触发器关闭的情况下，需要手工计算每个订单的应收金额：
      select sum(PRODUCT_NUM*PRODUCT_PRICE) into m from ORDER_DETAILS where ORDER_ID=v_order_id;
      if m is null then
       m:=0;
      end if;
      UPDATE ORDERS SET TRADE_RECEIVABLE = m - discount WHERE ORDER_ID=v_order_id;
      IF I MOD 1000 =0 THEN
        commit; --每次提交会加快插入数据的速度
      END IF;
    end loop;
  end;
```

## 查询语句

查询数据条数：

```sql
  select count(*) from zjf_201810513329.orders;
  select count(*) from zjf_201810513329.order_details;
```

![执行结果](./03.png)

查询2017-1-1至2018-6-1的订单：

  ```sql
  select * from zjf_201810513329.orders where order_date
  between to_date('2017-1-1','yyyy-mm-dd') and to_date('2018-6-1','yyyy-mm-dd');
  ```

![执行结果](./04.png)

查询2017-1-1至2018-6-1的订单详情：

```sql
  select a.ORDER_ID,a.CUSTOMER_NAME,
  b.product_name,b.product_num,b.product_price
  from zjf_201810513329.orders a,zjf_201810513329.order_details b where
  a.ORDER_ID=b.order_id and
  a.order_date between to_date('2017-1-1','yyyy-mm-dd') and to_date('2018-6-1','yyyy-mm-dd');
```

![执行结果](./05.png)

## 查看数据库的使用情况

以下样例查看表空间的数据库文件，以及每个文件的磁盘占用情况。

$ sqlplus system/123@pdborcl

```sql
SQL>SELECT tablespace_name,FILE_NAME,BYTES/1024/1024 MB,MAXBYTES/1024/1024 MAX_MB,autoextensible FROM dba_data_files  WHERE  tablespace_name='USERS';
SQL>SELECT a.tablespace_name "表空间名",Total/1024/1024 "大小MB",
 free/1024/1024 "剩余MB",( total - free )/1024/1024 "使用MB",
 Round(( total - free )/ total,4)* 100 "使用率%"
 from (SELECT tablespace_name,Sum(bytes)free
        FROM   dba_free_space group  BY tablespace_name)a,
       (SELECT tablespace_name,Sum(bytes)total FROM dba_data_files
        group  BY tablespace_name)b
 where  a.tablespace_name = b.tablespace_name;
```

- autoextensible是显示表空间中的数据文件是否自动增加。

- MAX_MB是指数据文件的最大容量。

- 运行结果截图：

  ![执行结果](./06.png)

  ![执行结果](./07.png)

## 实验总结

经过这次实验，我学习到了如何在虚拟机上创建分区表的方法和插入相关数据的语法。明白了在创建分区表之前要先创建好分区存储位置，即分配分区存储空间。然后我还了解了如何在自己的用户下进行数据库和表的相关操作，比如运行sql文件等。最后还学会了怎么去查看数据库的使用情况。