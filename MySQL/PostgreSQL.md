PG自定义排序

https://bbs.csdn.net/topics/390121828?_=1509796322

create table test(
id int2 PRIMARY key,
name varchar(20)
)

insert into test values(6, 'six');
insert into test values(3, 'three');
insert into test values(4, 'four');
insert into test values(2, 'two');

// order by position 按照某个字段固定来排序


select * from test order by position(name in 'four') desc, id desc;





MySQL自定义排序

select * from user order by field(username, 'roo111', 'root'), create_time desc;





**查询表信息**

select * from information_schema.columns where table_schema='public' and table_name='service_knowledge';

参考 https://www.cnblogs.com/yulinlewis/p/9986622.html







**数组varchar[]类型的值操作**

```
update service_knowledge set scenary = array_remove(scenary, 'app')  where tenant_id = '1' and 'Om6iezair' = any(scenary) and 'app' = any(scenary)
update service_knowledge set scenary = array_replace(scenary, 'Om6iezair', 'app') where tenant_id = '1' and 'Om6iezair' = any(scenary);
```

大概意思是scenary 如果包含了Om6iezair、app， 就把scenary中的app删掉。然后把Om6iezair 替换成app