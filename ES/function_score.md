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
