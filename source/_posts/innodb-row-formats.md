---
title: InnoDB 行格式
date: 2020-06-15 11:43:08
tags:
- MySQL
- InnoDB
categories:
- coding
---

InnoDB 行格式决定了 InnoDB 数据表中每一行的数据是如何实际存储的。本文对 InnoDB 行格式进行一些探索。

<!--more-->

InnoDB 目前支持的行格式有：
- REDUNDANT
- COMPACT
- DYNAMIC
- COMPRESSED

MySQL 5.7 InnoDB 默认的行格式是 DYNAMIC，可以通过系统变量  `innodb_default_row_format` 查看当前数据库默认的行格式：
```sql
SELECT @@innodb_default_row_format;
```

## 行格式 Dynamic
实际存储中每一行数据从低位到高位依次由以下几部分组成：
- 长度可变字段的实际长度记录：长度不固定
- 值为 null 的列的标识记录：长度不固定，至少为 1 字节
- record header：长度固定为 5 字节
- 主键：长度不固定
- transaction ID：长度固定为6 字节
- roll pointer ：长度固定为 7 字节
- 各个列的实际数据：长度不固定

### 案例1

```sql
create table t1 (
    id int unsigned primary key,
    a varchar(32),
    b varchar(32),
    c varchar(32)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 ROW_FORMAT=DYNAMIC;

insert into t1 values(1, 'china', 'guangdong', 'guangzhou');
insert into t1 values(2, 'china', null, null);
insert into t1 values(3, null, null, null);
```

用 UltraEdit 打开 t1.ibd 文件，找到以上插入的 3 行记录的实际存储数据：
![](https://raw.githubusercontent.com/leejianyang/pic/master/row-format/01.png)

蓝色的是主键为 1 的行；绿色的是主键为 2 的行；红色是主键为 3 的行。逐行整理一下：

第一行记录的数据如下：
```
09 09 05  从低位到高位跟实际的列的顺序是反过来的，第3个可变长度的列的实际长度是9字节，第2个可变长度的列的长度是9字节，第1个可变长度的列的长度是5字节
00                          表示全部列都不含 null
00 00 10 00 2F              header 部分，占5个字节
00 00 00 00 01              主键，也就是id列，值为1
00 00 00 1D B7 4E           transaction ID
C1 00 00 01 FB 01 10        roll pointer
63 68 69 6E 61              第2列的数据：china
67 75 61 6E 67 64 6F 6E 67  第3列的数据：guangdong
67 75 61 6E 67 7A 68 6F 75  第3列的数据：guangzhou
```

紧接着第一行的数据后面就是第二行的数据：
```
05 第一个长度可变列（也就是列a）的实际长度是5字节
06 转换成二进制得到 0000 0110，这里也是从低位到高位反过来排得，意味着第2个和第3个可为null的列（b列和c列）的值是null，第1个可为null的列（a列）的值不为null
00 00 18 00 1C        header 部分
00 00 00 02           主键，也就是id列，值为2
00 00 00 1D B7 4F     transaction ID
C2 00 00 01 AD 01 10  roll pointer
63 68 69 6E 61        第2列的数据：china
剩下的b列和c列就不用实际存储null的
```

最后是第三行的数据：
```
因为所有可变长度列的长度都为0，所以是没有用于表示可变长度列长度的数据的
07                    转换成二进制得到 0000 1110，也就是表示 abc 三个列的值都是 null
00 00 20 FF A4	    header 部分
00 00 00 03           主键，也就是id列，值为3
00 00 00 1D B7 54     transaction ID
C5 00 00 01 FE 01 10  roll pointer
除主键外所有列的值都是null，所以剩下没有数据了
```

### 案例2：没有创建主键

```sql
create table t2 (
    a varchar(32),
    b varchar(32)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 ROW_FORMAT=DYNAMIC;

insert into t2 values('guangzhou', null);
```

用 UltraEdit 打开 t2.ibd 文件，找到对应的行数据

![](https://github.com/leejianyang/pic/raw/master/row-format/02.png)

整理如下：
```
09                              第一个可变长度的列的实际长度为9个字节
02                              第2个可为null的列的实际值为 null
00 00 10 FF F1                  header
00 00 00 00 09 00               由于没有可用的主键，自动生成了长度为6个字节的row id
00 00 00 1D B7 59               transaction ID
CA 00 00 01 B2 01 10            roll pointer
67 75 61 6E 67 7A 68 6F 75      列a的数据：guangzhou
```

### 案例3：CHAR 类型的存储

实际上，char 类型的字段长度也是可变的。例如表定义的时候，将某个列的类型设为 `char(10)`，但这个10指的是10个字符，并不代表10个字节。如果表的字符集是 utf8mb4 的话，意味着这个列的长度最短为 10 字节，最长可达到 40 字节。

```sql
CREATE TABLE `t3` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `a` char(10) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 ROW_FORMAT=DYNAMIC;

INSERT INTO t3 VALUES ('guangzhou');
INSERT INTO t3 VALUES ('广州广州');
```

实际存储成这个样子：
![](https://github.com/leejianyang/pic/raw/master/row-format/03.png)

红色是第一行，蓝色是第二行数据。

第一行整理如下：
```
0A                               第一个可变长度列的长度为10，虽然a列是char(10)，但是也需要记录实际长度
没有允许为null的列，所以没有代表null值列的数据
00 00 10 00 21                    header
00 00 00 01                       主键
00 00 00 1D B7 5E                 transaction ID
CF 00 00 01 C5 01 10              roll pointer
67 75 61 6E 67 7A 68 6F 75 20     a列的数据，不足10字节，使用十六进制数据20来占位
```

第二行整理如下：
```
0C                                     第一个可变长度列的长度为10，这里值的是a列，它的实际长度是12字节
00 00 18 FF D1                         header
00 00 00 02                            主键
00 00 00 1D B7 5F                      transaction ID
D0 00 00 01 C6 01 10                   roll pointer
E5 B9 BF E5 B7 9E E5 B9 BF E5 B7 9E    a列的数据，「广州广州」的utf8编码，占12字节
```

由此可见，char字段也属于长度可变的。以使用字符集 utf8mb4 为例，char(N) 的长度最少是 N字节，最长是 4*N 字节；如果实际数据长度小于N，就使用十六进制值为20的值来补全 N 位；如果实际数据长度大于N，那么就按实际数据来存储，不用加上20来填充。

### Overflow Pages

如果有一些列的数据长度过大，那么这列的数据就不会被完整地保存在当前数据页中，而是全部或部分保存到别的地方，这样的称为「off-page columns」。因为一来页的大小是固定的，一般为 16kb，数据太大的话一个页放不下；二来的话，如果一个页里面的数据行太少，会影响 b+ 树结构的效率。不同的行格式对 overflow page 的处理方式是不一样的，如果选择 Dynamic 行格式的话，将进行如下处理：
- 如果字段长度小于 40 bytes，那么数据依然在当前页存储
- 如果字段长于 40 bytes，也不一定会选择进行 off-page，会根据当前页的实际情况而定
- 如果选择进行 off-page 存储，那么只保存一个 20 bytes 的指针，该指针指向实际的存储位置

## 其它行格式
行格式 Compact 和 Dynamic 差不多，不同之处是进行 off-page 存储的时候，Compact 行格式会存储 off-page columns 的前 768 字节，再加上长度为 20 bytes 的指针。Compressed 行格式的存储也跟 Dynamic 差不多，但会进行数据压缩，省内存省硬盘但更耗费 CPU。Redundant 行格式比较古老，现在很少用了，暂且忽略不提。

---
参考资料：
- [stackoverflow: What is off page in Mysql?](https://stackoverflow.com/questions/48298502/what-is-off-page-in-mysql)
- [MySQL 手册: 14.11 InnoDB Row Formats](https://dev.mysql.com/doc/refman/5.7/en/innodb-row-format.html)
