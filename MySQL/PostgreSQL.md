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