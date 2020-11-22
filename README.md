# MySQL数据库中变长数据的存储
## InnoDB简介
InnoDB是MySQL默认的存储引擎。InnoDB在数据库中读取数据的方式：**将数据划分为若干页，再读区数据时以页作为磁盘和内存交互的基本单位，页的大小一般为16KB**。InnoDB存储引擎的常用数据记录格式有四种，分别是：**Compact**,**Redundant**,**Dynamic**,**Compressed**,可以在创建数据库表时指定记录格式。
## 变长数据的存储
### Compact行格式
以**Compact**行格式为例，其存储格式如下：
![Compact](https://raw.githubusercontent.com/zyxmaxwell/MySQL-/master/image/169710e8fafc21aa.png?token=AHJTGSBH3ACFS3GG2ASWARC7YOIF4)
记录分为**记录的额外信息**和**记录的真实数据**两部分，额外信息是为了描述记录所需的一些信息。
###变长字段长度列表
MySQL数据库支持存储变长的数据类型，为了方便数据的读取，在数据库中变长字段的存储分为两部分：  
1.真实的数据内容  
2.占用的字节数  
以Compact格式为例，**把所有变长字段的真实数据占用的字节长度都存放在记录的开头部位，从而形成一个变长字段长度列表，各变长字段数据占用的字节数按照列的顺序逆序存放**。变长字段长度列表即是存储各变长字段数据占用的字节数，每个变长字段长度列表占用两个字符。
###NULL值列表
表中的某些列可能存储NULL值，如果把这些NULL值都放到记录的真实数据中存储会很占地方，所以Compact行格式把这些值为NULL的列统一管理起来，存储到NULL值列表中。  
将每个允许存储NULL的列对应一个二进制位，按照顺序逆序排列，二进制位表示的意义如下：  
1.二进制位的值为1时，代表该列的值为NULL。  
2.二进制位的值为0时，代表该列的值不为NULL。
###一个简单的例子
``` MySQL
mysql> USE xiaohaizi;
Database changed

mysql> CREATE TABLE record_format_demo (
    ->     c1 VARCHAR(10),
    ->     c2 VARCHAR(10) NOT NULL,
    ->     c3 CHAR(10),
    ->     c4 VARCHAR(10)
    -> ) CHARSET=ascii ROW_FORMAT=COMPACT;
```
![image]()
    
    

