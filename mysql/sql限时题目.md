```sql
-- 记录遇到的4条sql限时题目

-- 题目
-- 有四个表,要求写出sql语句
-- 1、查询办税人所在的所有服务厅
-- 2、查询业务办理时间低于该业务平均办理时间的办税记录
-- 3、查询每位办税人的信息及到目前为止的总办理业务时长
-- 4、查询到办税厅办税超过12次的纳税人信息

-- 纳税人表
drop table IF exists `taxpayers`;
create table `taxpayers` (
  `id` int(11) not null auto_increment comment '纳税人id',
  `name` varchar(64) not null comment '纳税人姓名',
  `age`  int(6) not  null comment '纳税人年龄',
  `sex` int(1) not null  comment  '纳税人性别',
  primary key (`id`) using BTREE
);

-- 办税人表
drop table IF exists `staff`;
 create table `staff` (
  `id` int(11) not null auto_increment comment '办税人id',
  `name` varchar(64) not null comment '办税人名字',
  `off` varchar(64) not null  comment '办税人所在服务厅',
  primary key (`id`) using BTREE
);

-- 业务表
drop table IF exists `taxes`;
create table `taxes` (
  `id` int(11) not null auto_increment comment '可办税务id',
  `name` varchar(256) not null comment '可办税务名称',
  primary key (`id`) using BTREE
);

-- 办税记录表
drop table IF exists `processingTime`;
create table `processingTime` (
  `t_id` int(11) not null  comment '纳税人id',
  `ta_id` int(11) not null comment '办理税务id',
  `s_id` int(11) not null  comment '办税人id',
  `time` int(11) null default null comment '办理时间'
);

-- 1、查询办税人所在的所有服务厅

select distinct s.off from staff s ;

-- 2、查询业务办理时间低于该业务平均办理时间的办税记录

select p.* from processingTime p where  p.time < (select avg(p1.time) from processingTime p1 where p1.ta_id = p.ta_id);

-- 3、查询每位办税人的信息及到目前为止的总办理业务时长

select s.id,s.name,s.off,SUM(p.time) as 'total time' from staff s,processingTime p WHERE s.id = p.s_id GROUP BY p.s_id ;

-- 4、查询到办税厅办税超过12次的纳税人信息

select t.* from taxpayers t where t.id in (select p.t_id from processingTime p group by p.t_id having count(p.t_id) > 12);
```

