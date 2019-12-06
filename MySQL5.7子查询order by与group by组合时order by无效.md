# MySQL 5.7子查询order by与group by组合时order by无效

## 起因 

今天在工作中遇到一个需求,简单点的来说就是查询该类别下时间最近的一条数据,

例如:可以把(类别)比作微信的服务号,每天这些服务号都给你发信息,需要做的就是把最近的一条展示到列表中,而且最新的服务号在排在最上面

接下面我们用的一个学生成绩的例子来模拟这样一个需求,然后引出今天说所的内容.如果说嫌麻烦可以直接看最后的解决办法

> 可以转化为相同的需求查询: `根据课程查询最高分数的学生然后在根据分数排序`

## 准备数据
```sql
-- 建表语句
CREATE TABLE `score` (
  `id`      int(10)     NOT NULL AUTO_INCREMENT COMMENT 'id',
  `name`    varchar(20) NOT NULL                COMMENT '姓名',
  `score`   int(10)     NOT NULL                COMMENT '成绩',
  `course`  varchar(20) NOT NULL                COMMENT '课程',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=10 DEFAULT CHARSET=utf8mb4 COMMENT='成绩表';
-- 初始化数据
INSERT INTO `score`(`id`, `name`, `score`, `course`) VALUES (1, '小明', 80, '数学');
INSERT INTO `score`(`id`, `name`, `score`, `course`) VALUES (2, '小红', 99, '数学');
INSERT INTO `score`(`id`, `name`, `score`, `course`) VALUES (3, '小强', 88, '语文');
INSERT INTO `score`(`id`, `name`, `score`, `course`) VALUES (4, '小红', 89, '英语');
INSERT INTO `score`(`id`, `name`, `score`, `course`) VALUES (5, '小强', 100, '数学');
INSERT INTO `score`(`id`, `name`, `score`, `course`) VALUES (6, '小明', 90, '英语');
INSERT INTO `score`(`id`, `name`, `score`, `course`) VALUES (7, '小红', 89, '语文');
INSERT INTO `score`(`id`, `name`, `score`, `course`) VALUES (8, '小强', 91, '英语');
INSERT INTO `score`(`id`, `name`, `score`, `course`) VALUES (9, '小明', 95, '语文');
```

## 测试

当然到这里可能有人说这多简单呀可以先子查询`group by course`在`order by score`(分组并排序)然后在排序嘛,

```sql
select * from (select id, name, score, course from score group by course order by score) as a order by score desc;
```
|  id   | name  | score  | course  |
|  ----  | ----  |  ----  | ----  |
|   4    |  小红 |   89  |	英语  |
|   3    |  小强 |   88  |	语文  |
|   1    |  小明 |   80  |	数学  |

可以看到这样数据完全不对,这时有人想子查询里加个`max(score)`,接着来试试看

```sql
select * from (select id, name, max(score) as score, course from score group by course order by score) as a order by a.score desc 
```
|  id   | name  | maxScore  | course  |
|  ----  | ----  |  ----  | ----  |
|   1    |  小明  |  100  |	数学  |
|   3    |  小强  |  95  |	语文  |
|   4    |  小红  |  91  |	英语  |

咦,这会看上去好像是可以了.但是细心可以发现数据其实是不对的.成绩虽然是对的,但是`id`为`1`的数据与实际并不相符

看到这里我们好像发现了其实`group by`在去`order by`是行不通的,`max()`只是拿出了最高的成绩其他数据并不相符

仔细想一下流程我们是先`group by`在去`order by`,我们能不能先去`order by`再去`group by`呢?

实际上mysql它并不允许这样做,那我们在子查询中`order by`再然后在`group by`呢?

```sql
select * from (select * from score order by score desc) as a GROUP BY course order by score desc;
```
|  id   | name  | score  | course  |
|  ----  | ----  |  ----  | ----  |
|   4	|  小红 |   89	|   英语  |
|   3	|  小强 |   88	|   语文  |
|   1	|  小明 |   80	|   数学  |

执行成功了,但是数据好像有不对.为什么我们这样排序了但是没有作用呢?

通过查阅资料发现这样在`MySQL 5.7`之后优化了子查询,导致我们的排序没有生效.可以通过在子查询中添加`limit`解决

```sql
select * from (select * from score order by score desc limit 999999) as a  GROUP BY course order by score desc
```
|  id   | name  | score  | course  |
|  ----  | ----  |  ----  | ----  |
|   5    |	小强 |	100   |	数学  |
|   9    |	小明 |	95   |	语文  |
|   8    |	小强 |	91   |	英语  |

这回,完美的解决问题!

## 总结

**由于MySQL5.7之后优化了子查询所以才会导致子查询中的排序在和`order by`组合是不生效**

> 解决办法就是在子查询中添加`limit`来使`order by`生效

[相关资料](https://yq.aliyun.com/articles/72503)