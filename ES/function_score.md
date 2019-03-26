# function_score 查询
function_score 查询 是用来控制评分过程的终极武器，它允许为每个与主查询匹配的文档应用一个函数， 以达到改变甚至完全替换原始查询评分 _score 的目的。

## 按受欢迎度提高权重
```
	{
	  "query": {
		"function_score": { 
		  "query": { 
			"multi_match": {
			  "query":    "popularity",
			  "fields": [ "title", "content" ]
			}
		  },
		  "field_value_factor": { 
			"field": "votes" 
		  }
		}
	  }
	}
```