---
title: 二级索引与联合索引
date: 2020-06-15
categories: mysql
---

> 上周在公司内部分享了关于索引的一些相关概念，在会议最后，有两位同事想我提出了若干问题，这里就单独抽出来聊一聊
> 原问题：二级高效还是联合高效，还是根据场景来定？

谈到索引，主要分几大类，**聚簇索引**、**二级索引**、**联合索引**，先回顾一下这三种索引对应的概念

### 聚簇索引
聚簇索引就是以主键值作为索引而建立的一颗 b+ 树，子节点包含**记录完整的数据**，举例说明
```
- 创建表 issues
CREATE TABLE `issues`(
   `id` INT UNSIGNED AUTO_INCREMENT,
   `title` varchar(255),
   `author_id` int,
   PRIMARY KEY ( `id` )
)ENGINE=InnoDB DEFAULT CHARSET=utf8;

insert into issues(`title`, `author_id`) values('test_01', 3)
insert into issues(`title`, `author_id`) values('test_02', 10)
insert into issues(`title`, `author_id`) values('test_03', 20)
insert into issues(`title`, `author_id`) values('test_04', 5)
```
画成 mysql 结构的**聚簇索引 (由id建立起来的索引)** b+ 树后，结构是如此的，底下的`页10`、`页22`、`页8`都称为树的子节点，他们内部存储的都是数据表记录，注意，它是**按主键顺序或者rowid(当没有显式定义主键或唯一索引就会默认用 rowid 作为主键，当然了这个是隐藏列，平时我们是看不到的) 的大小顺序组成的..**，而页 10086 也即是我们平时所说的索引页，一颗树中可能有许许多多的索引页，简单画了下大概结构如下：

![输入图片说明](/images/mysql_index/01.png)

所以，总结如下：聚簇索引就是由 id 作为索引建立起来的 b+ 树，它是默认就存在的，换言之，不需要你手动创建，innodb引擎会自动帮你创建，如果觉得这棵树有点别扭，可以参考下这句话 **数据即索引，索引即数据**


### 二级索引
在聚簇索引之上创建的索引称之为辅助索引，辅助索引访问数据总是需要二次查找。**辅助索引叶子节点存储的不再是行所有内容，而是主键值 + 你建立的索引**。通过辅助索引首先找到的是主键值，再通过主键值找到数据行的数据叶，再通过数据叶中的Page Directory找到数据行。

对比上面的图，我们可以试着创建一个二级索引
```
alter table issues add index index_by_author_id(author_id);
```

然后b+树的索引结构就会转变为，如图所示
不同点在于**子节点不再存储完成的列记录**，对比可见 title 字段已经没有记录在这些子节点出现了
![输入图片说明](/images/mysql_index/02.png)

所以就会有如下区别
```
- 1. 此 sql 会首先在二级索引(index_by_author_id)中先找到对应记录，找到后发现子节点竟然没有 title 字段
- 2. 找到对应记录后，再次到聚簇索引进行查找 title 记录
select title from issues where author_id = 2;


- 此 sql 会首先在二级索引(index_by_author_id)中先找到对应记录，找到后发现子节点有 id 和 title 字段，直接返回
select id, author_id from issues where author_id = 2;
```


### 联合索引
联合索引最为常用，比如我要为**author_id 和 title**建立联合索引，字段有先后次序之分哦，联合索引(author_id, title) 和 (title, author_id) 是不同滴，(语法省略..)，比如建立 (author_id, title)，此索引包含两层含义：
1. 先把各个记录和页按照 author_id 列进行排序。
2. 在记录的 author_id 列相同的情况下，采用 title 列的大小进行排序

![输入图片说明](/images/mysql_index/03.png)

在查找过程中，其实主要区别也是和二级索引的类似，**如果 select 的字段没有命中索引，那么也是要到聚簇索引进行二次查找的**，所以回到最开始的问题，哪个高效？其实硬要选出一个的话，那么当然是二级索引较为高效，因为它的建立索引的字段列少，对比联合索引可以减少循环判断，但是最主要的还是取决于**是否需要回表**，即 select 语句是否有查询非索引的列...


