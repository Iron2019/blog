# function_score 查询
function_score 查询 是用来控制评分过程的终极武器，它允许为每个与主查询匹配的文档应用一个函数， 以达到改变甚至完全替换原始查询评分 _score 的目的。

## 按受欢迎度提高权重
假设有一个blog网站，我们想受用户欢迎的blog展示到尽可能上的位置，但是全文搜索排序顺序还是要按照相关度来作为主要排序依据，可以通过点赞数来实现受欢迎程度。基本的文档格式如下：
```
00000001 PUT /blogposts/post/1
00000002 {
00000003   "title":   "About popularity",
00000004   "content": "In this post we will talk about...",
00000005   "votes":   6
00000006 }
```
那么，我们在搜索时，可以用function_score与field_value_factor结合使用，即将相关度与点赞数结合，示例如下：
```
00000001 GET /blogposts/post/_search
00000002 
00000003 {
00000004   "query": {
00000005 	"function_score": {
00000006 	  "query": { 
00000007 		"multi_match": {
00000008 		  "query":    "popularity",
00000009 		  "fields": [ "title", "content" ]
00000010 		}
00000011 	  },
00000012 	  "field_value_factor": { 
00000013 		"field": "votes" 
00000014 	  }
00000015 	}
00000016   }
00000017 }
```
**第5行** `function_score` 查询将主查询和函数包括在内。
***
**第6行** 主查询优先执行。
***
**第12行** `field_value_factor` 函数会被应用到每个与主 `query` 匹配的文档
***
**第13行** 每个文档的 `votes` 字段都 必须 有值供 `function_score` 计算。如果 没有 文档的 `votes` 字段有值，那么就 必须 使用 `missing `属性 提供的默认值来进行评分计算。
***
在前面示例中，每个文档的最终评分 `_score` 都做了如下修改：
> new_score = old_score * number_of_votes

然而这并不会带来出人意料的好结果，全文评分 _score 通常处于 0 到 10 之间，如下图 图 29 “受欢迎度的线性关系基于 _score 的原始值 2.0” 中，有 10 个赞的博客会掩盖掉全文评分，而 0 个赞的博客的评分会被置为 0 。

***受欢迎度的线性关系基于 _score 的原始值 2.0***

![线性关系图](https://www.elastic.co/guide/cn/elasticsearch/guide/cn/images/elas_1701.png)

**modifier**

一种更好的方式是使用`modifier`平滑`votes`的值。换句话说，我们希望最开始的一些赞更重要，但是其重要性随着数字的增加而降低。0个赞与1个赞的区别应该比10个赞和11个赞的区别大很多。

对于上述情况，典型的`modifier`应用是使用`log1p`参数值，公式如下：
```
new_score=old_score *log(1+number_of_votes)
```

log对数函数是`votes`赞字段的评分曲线更平滑，如图["受欢迎度的对数关系基于`_score`的原始值`2.0`"](https://www.elastic.co/guide/cn/elasticsearch/guide/cn/images/elas_1702.png):

***欢迎度的对数关系基于`_score`的原始值`2.0`***

!["受欢迎度的对数关系基于`_score`的原始值`2.0`"](https://www.elastic.co/guide/cn/elasticsearch/guide/cn/images/elas_1702.png)

带`modifier`参数的请求如下：
```
00000001 GET /blogposts/post/_search
00000002 {
00000003   "query": {
00000004     "function_score": {
00000005       "query": {
00000006         "multi_match": {
00000007           "query":    "popularity",
00000008           "fields": [ "title", "content" ]
00000009         }
00000010       },
00000011       "field_value_factor": {
00000012         "field":    "votes",
00000013         "modifier": "log1p" 
00000014       }
00000015     }
00000016   }
00000017 }
```

**第13行**  modifierwei log1p.

修饰语`modifier`的值可以为:`none`(默认状态),`log`,`log1p`,`log2p`,`ln`,`ln1p`,`ln2p`,`square`,`sqrt`以及`reciprocal`。

**factor**

可以通过将`votes`字段与`factor`的积来调节受欢迎程度效果的高低：
```
00000001 GET /blogposts/post/_search
00000002 {
00000003   "query": {
00000004     "function_score": {
00000005       "query": {
00000006         "multi_match": {
00000007           "query":    "popularity",
00000008           "fields": [ "title", "content" ]
00000009         }
00000010       },
00000011       "field_value_factor": {
00000012         "field":    "votes",
00000013         "modifier": "log1p",
00000014         "factor":   2 
00000015       }
00000016     }
00000017   }
00000018 }
```

**第14行** 双倍效果.

添加了`factor`会使公式变成这样：
```
new_score = old_score * log(1 + factor * number_of_votes)
```

factor 值大于 1 会提升效果， factor 值小于 1 会降低效果，如图

***受欢迎度的对数关系基于多个不同因子***

!["受欢迎度的对数关系基于多个不同因子"](https://www.elastic.co/guide/cn/elasticsearch/guide/cn/images/elas_1703.png)

**boost_mode**

将全文评分与`field_value_factor`函数乘积的效果仍然可能太大，我们可以通过控制参数`boost_mode`来控制函数与查询评分`_score`合并后的结果，参数接受的值为：
> `mutiply`
     评分`_score`与函数值的乘积(默认)
>`sum`
     评分`_score`与函数值的和
>`min`
    评分`_score`与函数值的较小值
>`max`
	评分`_score`与函数值的较大值
>`replace`
	函数值取代评分`_score`
	


