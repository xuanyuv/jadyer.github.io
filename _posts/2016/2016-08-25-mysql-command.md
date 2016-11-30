---
layout: post
title: "MySQL命令小结"
categories: MySQL
tags: sql mysql schema tablespace
author: 玄玉
excerpt: 一些常用的MySQL命令，诸如元数据查询、统计、建表、修改表结构等等。
---

* content
{:toc}


## 建表

```sql
DROP TABLE IF EXISTS t_account_info;
CREATE TABLE t_account_info(
id          INT AUTO_INCREMENT PRIMARY KEY COMMENT '主键',
status      TINYINT(1)    NOT NULL COMMENT '账户状态：0--未认证，1--已认证，2--认证未通过',
password    CHAR(102)     NOT NULL COMMENT '登录密码',
email       VARCHAR(64)   NOT NULL COMMENT '登录邮箱',
resp_data   MEDIUMTEXT    NOT NULL COMMENT '接口应答报文',
money_max   DECIMAL(16,4) COMMENT '最高贷款额度，单位：元',
biz_time    DATETIME      NOT NULL COMMENT '业务时间',
send_time   DATETIME      DEFAULT NULL COMMENT '发送时间',
create_time TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
update_time TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
INDEX index_password(password),
UNIQUE INDEX unique_email_status(email, status)
)ENGINE=InnoDB DEFAULT CHARSET=UTF8 AUTO_INCREMENT=10001001 COMMENT='渠道账户信息表';
```

## 修改表结构

```sql
ALTER TABLE t_account COMMENT '账户信息表';
ALTER TABLE t_account MODIFY money_max DECIMAL(16,4) NOT NULL COMMENT '最高额度，单位：元';
ALTER TABLE t_account ADD COLUMN money_type TINYINT(1) COMMENT '金额类型：1--RMB，2--USD' AFTER id;
ALTER TABLE t_account DROP COLUMN money_type;

ALTER TABLE t_account ADD PRIMARY KEY(account_id);
ALTER TABLE t_account ADD INDEX index_password(password);
ALTER TABLE t_account ADD INDEX index_name_password(name, password);
ALTER TABLE t_account ADD UNIQUE INDEX index_name_email(name, email);

CREATE INDEX index_name_password ON t_account(name, password);
CREATE UNIQUE INDEX index_name_email ON t_account(name, email);

ALTER TABLE t_account DROP PRIMARY KEY;
ALTER TABLE t_account DROP INDEX index_name_password;
DROP INDEX index_name_password ON t_account;
```

## 修改表数据

```sql
-- 更新某字段值为另一个表的同名字段值
UPDATE t_user u, t_account a SET u.account_type=a.type WHERE u.account_id=a.id
```

## 查询元数据

```sql
-- 查询某张表的建表语句
SHOW CREATE TABLE t_admin;

-- 查询某张表的所有列名
SELECT COLUMN_NAME FROM information_schema.COLUMNS WHERE TABLE_NAME='t_admin';

-- 查询拥有某字段的所有表名
SELECT TABLE_NAME FROM information_schema.COLUMNS WHERE COLUMN_NAME='file_id';

-- 查询某数据库中的所有表名
SELECT TABLE_NAME FROM information_schema.TABLES WHERE TABLE_SCHEMA='数据库名';

-- 查询某张表存在的索引类型
SHOW INDEX FROM jadyer.t_admin;
SHOW INDEX FROM t_admin FROM jadyer;
SELECT INDEX_NAME, INDEX_TYPE FROM INFORMATION_SCHEMA.STATISTICS WHERE TABLE_NAME='t_admin';
```

## 统计昨日、今日、本周数据

```sql
-- 累计扫描量
SELECT t.tag, count(*) scanCounts FROM t_qq_qrcode t GROUP BY t.tag;

-- 今日扫描量
SELECT t.tag, count(*) scanCountsOfToday FROM t_qq_qrcode t
WHERE datediff(now(),t.create_time)=0 GROUP BY t.tag;

-- 昨日扫描量
SELECT t.tag, count(*) scanCountsOfYesterday FROM t_qq_qrcode t
WHERE datediff(now(),t.create_time)=1 GROUP BY t.tag;

-- 本周扫描量
SELECT t.tag, count(*) scanCountsOfThisWeek FROM t_qq_qrcode t
WHERE yearweek(date_format(t.create_time,'%Y-%m-%d'))=yearweek(now()) GROUP BY t.tag;

-- 指定日期的扫描量
SELECT t.tag, date_format(t.create_time, '%Y%m%d') theDate, count(*) scanCountsOfToday FROM t_qq_qrcode t
WHERE date_format(t.create_time, '%Y%m%d')='20160503' GROUP BY t.tag;
```

## 表中存在重复数据时的统计

```sql
-- 对于表中存在重复数据的，查詢出过滤掉重复数据后的
SELECT id FROM coop_push_user GROUP BY mobile HAVING count(mobile)=1;

-- 对于表中存在重复数据的，查詢重复数据中最旧的那条
SELECT id FROM coop_push_user GROUP BY mobile HAVING count(mobile)>1;

-- 对于表中存在重复数据的，查詢重复数据中最新的那条，对于其它无重复数据的則原样查出
SELECT mobile, status-1, create_time FROM coop_push_user
WHERE id in(SELECT max(id) FROM coop_push_user GROUP BY mobile);

-- 对于一对多的表统计，根据[一]把[多]里面的某个字段都查出来在一起
SELECT t.email, t.name, IF(t.type=1, '个人', IF(t.type=2,'企业','未知')) AS accountType,
GROUP_CONCAT(ac.channel_no) AS channelList,
aco.cooper_no
FROM t_account t
LEFT JOIN t_account_channel ac ON t.id=ac.account_id
LEFT JOIN t_account_cooper aco ON t.id=aco.account_id
GROUP by t.id;
```

## 同一张表分别统计后汇总结果

```sql
SELECT t1.totalApply, t2.totalSign, IF(t3.money IS NULL,0,t3.money) money FROM
(SELECT COUNT(t.id) totalApply FROM t_apply_info t WHERE t.apply_date=20160810) t1,
(SELECT COUNT(t.id) totalSign FROM t_apply_info t WHERE t.sign_date=20160810) t2,
(SELECT SUM(t.pay_money) totalMoney FROM t_apply_info t WHERE t.pay_date=20160810) t3;
```