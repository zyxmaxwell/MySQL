# MySQL数据库中变长数据的存储
## InnoDB简介
InnoDB是MySQL默认的存储引擎。InnoDB在数据库中读取数据的方式：**将数据划分为若干页，再读区数据时以页作为磁盘和内存交互的基本单位，页的大小一般为16KB**。InnoDB存储引擎的常用数据记录格式有四种，分别是：**Compact**,**Redundant**,**Dynamic**,**Compressed**,可以在创建数据库表时指定记录格式。
## 变长数据的存储
### Compact行格式
以**Compact**行格式为例，其存储格式如下：
![image1](https://raw.githubusercontent.com/zyxmaxwell/MySQL-/master/image/169710e8fafc21aa.png?token=AHJTGSBH3ACFS3GG2ASWARC7YOIF4)
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
mysql> USE example;
Database changed

mysql> CREATE TABLE record_format_demo (
    ->     c1 VARCHAR(10),
    ->     c2 VARCHAR(10) NOT NULL,
    ->     c3 CHAR(10),
    ->     c4 VARCHAR(10)
    -> ) CHARSET=ascii ROW_FORMAT=COMPACT;
```
在 record_format_demo表中的存在记录的两条数据如下：
![image2](https://raw.githubusercontent.com/zyxmaxwell/MySQL-/master/image/169710e95903144f.png?token=AHJTGSDGMOXSGU5MYMHLC2K7YOLUE)
第一条数据中  
变长字段长度列表代表c1,c2,c4的长度分别为4，3，1字节（变长字段数据采用从低到高的顺序存放），不存在空项。   
 第二条数据中    
 NULL字段的06的二进制表示为00000110（NULL值列表必须为整字节数）， 由于c2不能为NULL，第二条记录的NULL值列表表示c3，c4为空值，变长字段长度列表表示c1长4字节，c2长三字节。
##  数据页结构
###  页式存储
页是InnoDB管理存储空间的基本单位，页的基本结构可以分为多个部分，这里暂不做具体介绍。  
每当插入一条数据时，会在页的User_Recoders中申请一部分空间用来存储新插入的数据，在空间用完后再去申请新的页。
![image3](https://raw.githubusercontent.com/zyxmaxwell/MySQL-/master/image/16f13ee1e2dfac7c.png?token=AHJTGSDE56TJYGU2EFEL7FC7YOYUS)
###记录头信息
数据在页中存储时，除了前一部分讲的**记录的额外信息**，还会在前面加上**记录头字段**。
![image4]()
其中几个重要的字段含义:  
delete_mask:这个属性标记着当前记录是否被删除，占用1个二进制位，值为0的时候代表记录并没有被删除，为1的时候代表记录被删除掉了。  
min_rec_mask:B+树的每层非叶子节点中的最小记录都会添加该标记.    
heap_no:这个属性表示当前记录在本页中的位置。  
record_type:这个属性表示当前记录的类型，一共有4种类型的记录，0表示普通记录，1表示B+树非叶节点记录(目录项记录)，2表示最小记录，3表示最大记录  
next_record:这玩意儿非常重要，它表示从当前记录的真实数据到下一条记录的真实数据的地址偏移量。
###一个简单的例子
数据的存储
![image5](https://raw.githubusercontent.com/zyxmaxwell/MySQL-/master/image/16a95c1084c440b4.png?token=AHJTGSFPIFGYP2DIP6ZOKDS7YOYWW)
图中最小记录和最大记录是InnoDB自动生成的伪记录，分别代表最小记录和最大记录，存储在页中**的Infimum + Supremum**部分。
数据的删除
![image6]()
可以看到图中数据2删除后并没有清除空间，而是将delete_mask字段标为1，并将数据1指向数据3，数据2的next_record变为0。
### 页连接
数据库中页和记录的关系如下图
![image7](https://raw.githubusercontent.com/zyxmaxwell/MySQL-/master/image/16a01bd1b8eafbb4.png?token=AHJTGSA363L5UXETPSKAGL27YOY24)
可以看出页和页之间是会组成一个**双链表**
## B+树搜索
### 目录项记录
目录项与页是一一对应的，每个目录项包括：  
1、页的最小主键值。  
2、页号。  
目录项记录是存储目录项的数据页，并有以下特点：  
1、record_type值是1，而普通用户记录的record_type值是0  
2、只有主键值和页的编号两个列
简单的示例图如下：
![image8]()
### B+树
当表中的数据非常多则会产生很多存储目录项记录的页，这时为这些存储目录项记录的页再生成一个更高级的目录，这就产生了B+树。  
如图所示
![image9]()
### 二级索引
前面所讲的目录项记录和B+树都是以主键为索引的条件，当需要以主键外的属性建立索引时，需要建立不同的B+树索引。在这些B+树中，叶子结点仅包括索引项和主键值，拿到完整的数据信息需要依据主索引查找。


  
 
    

