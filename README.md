# MySQL数据库中变长数据的存储
## InnoDB简介
InnoDB是MySQL默认的存储引擎。InnoDB在数据库中读取数据的方式：**将数据划分为若干页，再读区数据时以页作为磁盘和内存交互的基本单位，页的大小一般为16KB**。InnoDB存储引擎的常用数据记录格式有四种，分别是：**Compact**,**Redundant**,**Dynamic**,**Compressed**,可以在创建数据库表时指定记录格式。
## 变长数据的存储
以**Compact**行格式为例