+++
title = "使用SQL 数据库建模树形结构"
date =  2020-12-20

[taxonomies]
tags = ["sql", "database", "tree"]
+++



一般建模树形结构，比较直观的方式是在表中引入一个单独的字段，track 本记录的父记录id，比如典型的帖子回复：

```sql
create database if not exists tree_test default charset 'utf8' collate 'utf8_bin';

use tree_test;

create table if not exists Posts (
    post_id    serial primary key,
    publish_at  datetime not null default current_timestamp,
    title      text  not null,
    content    text  not null
);

create table if not exists  Comments (
  comment_id   serial primary key,
  parent_id    bigint unsigned,
  post_id      bigint unsigned not null,
  author       varchar(100),
  comment_at   datetime not null  default current_timestamp,
  comment      text not null,

  foreign key (parent_id) references Comments(comment_id),
  foreign key (post_id) references Posts(post_id)
);

insert into Posts(title, content) 
values('When a mouse over a file is enough to crash your system', 'There is no previous analysis for this vulnerability at the time of writing. We showed how we could analyze it precisely with REVEN, minimized the PoC and explained the influence of each faulty byte. In particular, we used the taint feature many times to quickly go through many memory manipulation and find the origin of some values.'), 
('An unsafety bug in rust\'s stdlib', 'The str::repeat function in the standard library allows repeating a string a fixed number of times, returning an owned version of the final string. The capacity of the final string is calculated by multiplying the length of the string being repeated by the number of copies. This calculation can overflow, and this case was not properly checked for.');

insert into Comments(post_id, parent_id, author, comment) values (1,null, 'Fran', 'What\'s the cause of this bug?');
insert into Comments(post_id, parent_id, author, comment) values (1,1, 'Ollie', 'I think it\'s a null pointer.');
insert into Comments(post_id, parent_id, author, comment) values(1, 2, 'Fran',  'No, I checked for that.');
insert into Comments(post_id, parent_id, author, comment) values(1, 1, 'Kukla', 'We need to check for invalid input.');
insert into Comments(post_id, parent_id, author, comment) values(1, 4, 'Ollie', 'Yes, that’s a bug.');
insert into Comments(post_id, parent_id, author, comment) values(1, 4, 'Fran', 'Yes, please add a check.');
insert into Comments(post_id, parent_id, author, comment) values(1, 6, 'Kukla', 'That fixed it.');
```
Comments 的外键parent_id  与 主键comment_id 形成了一对多的自引用关系， 形成一个树； 上面插入的样本数据，形成的树的图形：

{{ image(src="https://oscimg.oschina.net/oscnet/up-f1d6ff8553bd28043509e5cb6e2f87e223e.png", position="left") }}

可以比较方便的获取节点与其直接后代的关系：
```sql
SELECT c1.comment_id as pid , group_concat(c2.comment_id) as cids
FROM Comments c1 left join Comments c2
  ON c2.parent_id = c1.comment_id  group by  pid ;
```

{{ image(src="https://oscimg.oschina.net/oscnet/up-14137bdf53cceb73954444f778a2dca3a6e.png", position="left") }}

为了获取节点的后裔(包括非直接后代)， 需要多次的join：
```sql
select  c1.comment_id, c2.comment_id, c3.comment_id, c4.comment_id
from  Comments c1  
  left join Comments c2
    on c2.parent_id = c1.comment_id  
  left  join Comments c3
    on  c3.parent_id = c2.comment_id  
  left  join Comments c4
    ON c4.parent_id = c3.comment_id;   
```

{{image(src="https://oscimg.oschina.net/oscnet/up-c122c24456d902973fafbb89b03ba394d41.png", position="left") }}


但是这种方式比较受限，如果不能提前知道树的最大深度，就不知道需要写多少次join 操作，[如果使用mysql 8.0 以上版本的数据库](http://mysqlserverteam.com/mysql-8-0-labs-recursive-common-table-expressions-in-mysql-ctes-part-three-hierarchies/ "如果使用mysql 8.0 以上版本的数据库")， MariaDB 10.2.2, 或者是postgresql  支持[Recursive Common Table Expressions ](https://mariadb.com/kb/en/recursive-common-table-expressions-overview/ "Recursive Common Table Expressions "),  很多操作就方便很多:

获取整颗树的层级:
```sql
WITH  recursive CommentTree 
    (comment_id, parent_id, post_id, author, comment_at, comment, depth)
AS (
    SELECT *, 0 AS depth FROM Comments
    WHERE parent_id IS NULL
  UNION all
    SELECT c.*, ct.depth+1 AS depth FROM CommentTree ct JOIN Comments c ON (ct.comment_id = c.parent_id)
)  SELECT * FROM CommentTree;
```

{{image(src="https://oscimg.oschina.net/oscnet/up-294bcf46dee192f7625cbbe467074ce85bc.png", position="left") }}

获取以id 4 为根的子树：
```sql
WITH  recursive CommentTree 
    (comment_id, parent_id, post_id, author, comment_at, comment, depth)
AS (
    SELECT *, 1 AS depth FROM Comments
    where comment_id = 4
  UNION all
    SELECT c.*, ct.depth+1 AS depth FROM CommentTree ct JOIN Comments c ON (ct.comment_id = c.parent_id)
)  SELECT * FROM CommentTree;
```
获取根节点:
```sql
select  * from Comments where parent_id is null;
```

获取叶子节点:
```sql
select * from Comments as p
where  not exists (
   select *  from Comments as  c  where   p.comment_id = c.parent_id
);
```
节点7 下插入一个孩子节点：
```sql
insert into Comments(post_id, parent_id, author, comment) values(1, 7 , 'Fran', 'great job.');
```
移动一颗子树:
```sql
update Comments set parent_id = 3 where comment_id = 6;
```

删除整颗子树, 比如删除以4为根的子树， 需要找到4的所有后裔，从底部向上的按序删除，以防止违反外键约束：
```
delete from Comments where comment_id in ( 7 );
delete from Comments where comment_id in ( 5, 6 );
delete from Comments where comment_id = 4;
```

如果数据库支持cte， 而且Comments 表的外键这样定义: 
```sql
foreign key (parent_id) references Comments(comment_id) on delete cascade,
```

可以一条语句删除子树：
```sql
WITH  recursive CommentTree      (comment_id, parent_id, post_id, author, comment_at, comment, depth)
AS (    
	   SELECT *, 1 AS depth FROM Comments     where comment_id = 4  
	  UNION all     
	  SELECT c.*, ct.depth+1 AS depth FROM CommentTree ct JOIN Comments c ON (ct.comment_id = c.parent_id)
)  delete from Comments where  comment_id in (select comment_id from  CommentTree order by comment_id );
```

删除非叶子节点，该节点的所有孩子节点，提升为当前节点的父节点的直接孩子， 比如删除节点4， 节点5，6 的父节点变成节点1：
```sql
update   Comments as c1  join  Comments as c2 on c1.comment_id = c2.comment_id and c2.parent_id = 4 
set  c1.parent_id = (select parent_id from (select parent_id from Comments  where  comment_id = 4) as t);

delete from Comments where comment_id = 4;
```

上面的 update 语句看起来有些怪异，实际逻辑上是为了表达这个语句:

```sql
update  Comments set parent_id = 1 where comment_id in (select  comment_id from Comments where parent_id = 4);
```
mysql 会报错误码1093， 不支持update 同一个表时候，还在select 此表.

还有一种使用额外一张表，存储节点层级关系的方法:
```
create database if not exists test_tree default charset 'utf8' collate 'utf8_bin';

use test_tree;

create table if not exists Posts (
    post_id    serial primary key,
    publish_at  datetime not null default current_timestamp,
    title      text  not null,
    content    text  not null
);

create table if not exists  Comments (
  comment_id   serial primary key,
  post_id      bigint unsigned not null,
  author       varchar(100),
  comment_at   datetime not null  default current_timestamp,
  comment      text not null,

  foreign key (post_id) references Posts(post_id)
);

create table TreePaths (
  ancestor    bigint unsigned not null,
  descendant  bigint unsigned not null,
  path_length bigint unsigned not null,

  primary key(ancestor, descendant),
  foreign key (ancestor) references Comments(comment_id),
  foreign key (descendant) references Comments(comment_id)
);



insert into Posts(title, content) 
values('When a mouse over a file is enough to crash your system', 'There is no previous analysis for this vulnerability at the time of writing. We showed how we could analyze it precisely with REVEN, minimized the PoC and explained the influence of each faulty byte. In particular, we used the taint feature many times to quickly go through many memory manipulation and find the origin of some values.'), 
('An unsafety bug in rust\'s stdlib', 'The str::repeat function in the standard library allows repeating a string a fixed number of times, returning an owned version of the final string. The capacity of the final string is calculated by multiplying the length of the string being repeated by the number of copies. This calculation can overflow, and this case was not properly checked for.');


insert into Comments(post_id, author, comment) values(1, 'Fran', 'What\'s the cause of this bug?');  -- 1
insert into TreePaths (ancestor, descendant, path_length)  
 select t.ancestor, 1, t.path_length + 1
  from TreePaths AS t
  where t.descendant = -1
 union all
 select 1, 1, 0;

insert into Comments(post_id, author, comment) values(1, 'Ollie', 'I think it\'s a null pointer.');   -- 2
insert into TreePaths (ancestor, descendant, path_length)
  select t.ancestor, 2, t.path_length + 1
  from TreePaths AS t
  where t.descendant = 1
 union all
  select 2, 2, 0;


insert into Comments(post_id, author, comment) values(1,  'Fran',  'No, I checked for that.');   -- 3
insert into TreePaths (ancestor, descendant, path_length)
  select t.ancestor, 3, t.path_length + 1
  from TreePaths AS t
  where t.descendant = 2
 union all
  select 3, 3, 0;


insert into Comments(post_id, author, comment) values(1,  'Kukla', 'We need to check for invalid input.');  -- 4
insert into TreePaths (ancestor, descendant, path_length)
  select t.ancestor, 4, t.path_length + 1
  from TreePaths AS t
  where t.descendant =1
 union all
  select 4, 4, 0;

insert into Comments(post_id, author, comment) values(1,  'Ollie', 'Yes, that’s a bug.');   -- 5
insert into TreePaths (ancestor, descendant, path_length)
  select t.ancestor, 5, t.path_length + 1
  from TreePaths AS t
  where t.descendant =4
 union all
  select 5, 5, 0;

insert into Comments(post_id, author, comment) values(1,  'Fran', 'Yes, please add a check.');    -- 6
insert into TreePaths (ancestor, descendant, path_length)
  select t.ancestor, 6, t.path_length + 1
  from TreePaths AS t
  where t.descendant =4
 union all
  select 6, 6, 0;


insert into Comments(post_id, author, comment) values(1,  'Kukla', 'That fixed it.');  -- 7
insert into TreePaths (ancestor, descendant, path_length)
  select t.ancestor, 7, t.path_length + 1
  from TreePaths AS t
  where t.descendant =6
 union all
  select 7, 7, 0;
```
TreePaths 就是用来专门存放节点之间层级关系的表，ancestor 字段是祖先节点， descendant是后裔节点， path_length是此祖先到此后裔的距离， 上面的样本数据插入后，形成的图及TreePaths表中数据：
{{ image(src="https://oscimg.oschina.net/oscnet/up-7fe9858ac777581dff21c50d56cf775acd6.png", position="left") }}

{{ image(src = "https://oscimg.oschina.net/oscnet/up-2c9b5055d9491bb0eaa00fd39069e909ac1.png", position="left") }}

像(1,1,0) , (2,2,0), (3,3,0), ....... (7,7,0) 这样的元组代表图中 自引用节点；节点1 到节点1 的距离是0，   节点1到节点2的距离是1， 2到3的距离是1，  1到5是2， 1到7是3.   TreePaths 存储的是某节点在树中下降的可达路径.

这样的构造是怎么形成的，以节点6下面添加节点7为例:
```sql
  select t.ancestor, t.descendant, t.path_length
  from TreePaths AS t
  where t.descendant =6;
```
输出:

{{ image(src="https://oscimg.oschina.net/oscnet/up-62885da8252bc59a62065bb278db4980020.png", position="left") }}

节点6 下面添加节点7，  6 的祖先1，4及6 自身都变得路径可达 节点7， 1到7 的路径长度就是1到6的长度加1， 4到7 的路径长度就是4到6的长度加1， 6到7 的路径长度就是6到6的长度加1， 再加上7到7本身可达，路径长度是0，产生了插入TreePaths的sql:
```sql
insert into TreePaths (ancestor, descendant, path_length)
  select t.ancestor, 7, t.path_length + 1
  from TreePaths AS t
  where t.descendant =6
 union all
  select 7, 7, 0;
```


获取节点4的后裔:

```sql
select c.*
from Comments as c
  join TreePaths as t on c.comment_id = t.descendant
where t.ancestor = 4;
```

获取节点6的祖先:

```sql
select c.*
from Comments as c
  join TreePaths as t on c.comment_id = t.ancestor
where t.descendant = 6;
```

删除叶子节点7：

```sql
delete from TreePaths where descendant = 7;
```


删除以4为根的子树:

```sql
delete from TreePaths
where descendant in (
     select descendant
     from TreePaths
     where ancestor = 4
);
```

子树移动，比如以6 为根的子树移动到节点3下面，首先断掉6的祖先到6 及6的孩子节点7的连接(1,6), (4,6),  (1,7), (4,7)， 保留，6到自身，6到其孩子7的连接(6,6), (6,7)：

```sql
delete from TreePaths
where descendant in (select descendant
		     from TreePaths
		     where ancestor = 6)
  and ancestor in (select ancestor
		   from TreePaths
		   where descendant = 6
		     and ancestor != descendant);
```

生成新的路径组合，(1,6),(2,6),(3,6),  (1,7),(2,7), (3,7) 映射子树的新位置：
```sql
insert into TreePaths (ancestor, descendant)
  select supertree.ancestor, subtree.descendant
  from TreePaths as supertree
    cross join TreePaths as subtree
  where supertree.descendant = 3
    and subtree.ancestor = 6;
```

