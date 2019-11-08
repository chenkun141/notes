## 参考网站 ##
[https://es.xiaoleilu.com/index.html](https://es.xiaoleilu.com/index.html)
## Elasticsearch入门使用 ##
Elasticsearch是一款稳定高效的分布式搜索和分析引擎,它的底层基于lucene,并提供了友好的Restful api来对数据进行操作,还有比较重要的一点是,Elasticsearch开箱即可用,上手也比较容易.  
目前Elasticsearch在搭建企业级搜索(如日志搜索,商品搜索等)平台中很广泛,官网也提供了不少案例,比如:

- GitHub使用Elasticsearch检索超过800万的代码库
- eBay使用Elasticsearch搜索海量的商品数据
- Netflix使用Elasticsearch来实现高效的消息传递系统

本文主要介绍Elasticsearch的基本概念和入门使用

## 安装 ##
在安装Elasticsearch之前,请确保你的计算机已经安装了Java.目前Elasticsearch 的最新版是5.2,需要安装Java 8,如果你用的是老版本的Elasticsearch ,如2.x版,可用Java 7 ,但还是推荐使用Java 8.

下载最新版本的Elasticsearch ,使用wget下载,如下:

	wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.2.2.tar.gz

下载完成后进行解压:

	tar -zxvf elasticsearch-5.2.2.tar.gz

## 运行 ##
首先,我们进入到刚刚解压出来的目录中:

	cd elasticsearch-5.2.2

接着 使用如下命令启动Elasticsearch

	bin/elasticsearch

此时,如果正常的话,你可以在终端看到类似如下的输出:

	[2017-03-04T23:25:09,961][INFO ][o.e.n.Node               ] [] initializing ...
	[2017-03-04T23:25:10,073][INFO ][o.e.e.NodeEnvironment    ] [yO11WLM] using [1] data paths, mounts [[/ (/dev/disk0s2)]], net usable_space [141.1gb], net total_space [232.9gb], spins? [unknown], types [hfs]
	[2017-03-04T23:25:10,074][INFO ][o.e.e.NodeEnvironment    ] [yO11WLM] heap size [1.9gb], compressed ordinary object pointers [true]
	[2017-03-04T23:25:10,095][INFO ][o.e.n.Node               ] node name [yO11WLM] derived from node ID [yO11WLMOQDuAOpZbYZYjzw]; set [node.name] to override
	[2017-03-04T23:25:10,100][INFO ][o.e.n.Node               ] version[5.2.2], pid[7607], build[db0d481/2017-02-09T22:05:32.386Z], OS[Mac OS X/10.11.5/x86_64], JVM[Oracle Corporation/Java HotSpot(TM) 64-Bit Server VM/1.8.0_102/25.102-b14]
	[2017-03-04T23:25:11,363][INFO ][o.e.p.PluginsService     ] [yO11WLM] loaded module [aggs-matrix-stats]
	...


上面的命令是在前台运行的,如果想在后台以守护进程模式运行,可以加**-d**参数.  
Elasticsearch启动后,也启动了两个端口9200和9300:  

- 9200端口:HTTP RESTFUL接口的通讯端口
- 9300端口:TCP通讯端口,用于集群间节点通讯和与Java客户端通信的端口

现在,让我们做一些测试.在浏览器访问链接**http://localhost:9200/ **,或使用curl命令:

	curl 'http://localhost:9200/?pretty'

我们可以看到类似如下的输出:

	{
	  "name" : "yO11WLM",
	  "cluster_name" : "elasticsearch",
	  "cluster_uuid" : "yC8BGwzlSnu_zGbKL918Xg",
	  "version" : {
	    "number" : "5.2.1",
	    "build_hash" : "db0d481",
	    "build_date" : "2017-02-09T22:05:32.386Z",
	    "build_snapshot" : false,
	    "lucene_version" : "6.4.1"
	  },
	  "tagline" : "You Know, for Search"
	}

## 概念 ##
在进一步使用Elasticsearch之前,我们先了解几个关键概念.

在逻辑层面:

- Index(索引):这里的Index是名词,一个Index就像是传统关系数据库的Database,它是Elasticsearch用来存储数据的逻辑区域
- Document(文档):Elasticsearch使用json文档来表示一个对象,就像是关系数据库中一个table中的一行数据
- Type(类型):文档归属于一种Type,就像是关系数据库中的一个Table
- Field(字段):每个文档包含多个字段,类似关系数据库中一个Table的列

我们用一个表格来做类比,如下:
  

|Elasticsearch |	MySQL |  
|--|--|  
|Index|	Database|  
|Type|	Table|  
|Document|	Row|  
|Field	|Column|  

在物理层面:

- Node(节点):node是一个运行这的Elasticsearch实例,一个node就是一个单独的server
- Cluster(集群):cluster是多个node集群
- Shard(分片):数据分片,一个index可能会存在于多个shard

## 使用 ##
接下来,我们看看如何建立索引,创建文档等,就好比在Mysql中进行诸如创建数据库,插入数据等操作.

## 添加文档 ##
下面,我们将创建一个存储电影信息的Document:  

- Index的名称为movie
- Type为adventure
- Document有两个字段:name和actors

我们使用Elasticsearch提供的Restful api 来执行上述操作,如图所示:  
![](https://i.imgur.com/EqANJH9.png)  

- 用url表示一个资源,比如/movie/adventure/1就表示一个index为movie,type为adventure,id为1的ducument
- 用http方法操作资源,如使用get获取资源,使用post,put新增或更新资源,使用delete删除资源等

我们可以使用curl命令来执行上述操作:

	curl -i -X PUT "localhost:9200/movie/adventure/1" -d '{"name": "Life of Pi", "actors": ["Suraj", "Irrfan"]}'

不过,本文推荐使用httpie,类似curl,但比curl更好用,将上面的命令换成httpie,如下:

	http put :9200/movie/adventure/1 name="Life of Pi" actors:='["Suraj", "Irrfan"]'

上面的命令结果如下:

	HTTP/1.1 201 Created
	Location: /movie/adventure/1
	content-encoding: gzip
	content-type: application/json; charset=UTF-8
	transfer-encoding: chunked
	
	{
	    "_id": "1",
	    "_index": "movie",
	    "_shards": {
	        "failed": 0,
	        "successful": 1,
	        "total": 2
	    },
	    "_type": "adventure",
	    "_version": 1,
	    "created": true,
	    "result": "created"
	}

可以看到,我们已经成功创建一个_index的movie,_type为adventure,_id为1的文档.  

我们通过get请求来查看这个文档的信息:

	http :9200/movie/adventure/1

结果如下:

	HTTP/1.1 200 OK
	content-encoding: gzip
	content-type: application/json; charset=UTF-8
	transfer-encoding: chunked
	
	{
	    "_id": "1",
	    "_index": "movie",
	    "_source": {
	        "actors": [
	            "Suraj",
	            "Irrfan"
	        ],
	        "name": "Life of Pi"
	    },
	    "_type": "adventure",
	    "_version": 1,
	    "found": true
	}

可以看到,原始的文档数据存在了_source字段中.

如果我们的数据没有id,也可以让Elasticsearch自动为我们生成,此时要使用post请求,形式如下:

	POST /movie/adventure/
	{
	    "name": "Life of Pi"
	}

## 更新整个文档 ##
当我们使用put方法指明文档的_index,_type和_id时,如果_id已存在,则新文档会替换旧文档,此时文档的_version会增加1,并且_created字段为false.比如:

	http put :9200/movie/adventure/1 name="Life of Pi"

结果如下:

	HTTP/1.1 200 OK
	content-encoding: gzip
	content-type: application/json; charset=UTF-8
	transfer-encoding: chunked
	
	{
	    "_id": "1",
	    "_index": "movie",
	    "_shards": {
	        "failed": 0,
	        "successful": 1,
	        "total": 2
	    },
	    "_type": "adventure",
	    "_version": 2,
	    "created": false,
	    "result": "updated"
	}

使用get请求查看新文档的数据:

	http :9200/movie/adventure/1

结果如下:

	HTTP/1.1 200 OK
	content-encoding: gzip
	content-type: application/json; charset=UTF-8
	transfer-encoding: chunked
	
	{
	    "_id": "1",
	    "_index": "movie",
	    "_source": {
	        "name": "Life of Pi"
	    },
	    "_type": "adventure",
	    "_version": 2,
	    "found": true
	}

可以看到,actors这个字段已经不存在了,文档的_version变成了2.  
因此,为了避免在误操作的情况下,原文档被替换,我们可以使用_create这个api,表示只在文档不存在的情况下才创建新文档(返回201 created),如果文档存在则不做任何操作(返回409 conflict),命令如下:

	http put :9200/movie/adventure/1/_create name="Life of Pi"

由于文档id存在,会返回409 conflict.

## 局部更新 ##
在有些情况下,我们只想更新文档的局部,而不是整个文档,这是我们可以使用_update这个api.  

现在,待更新的文档信息如下:

	{
	    "_id": "1",
	    "_index": "movie",
	    "_source": {
	        "name": "Life of Pi"
	    },
	    "_type": "adventure",
	    "_version": 2,
	    "found": true
	}

最简单的update请求接受一个局部文档参数**doc**,它会合并到现有文档中:将对象合并在一起,存在的标量字段被覆盖,新字段被添加.

形式如下:

	POST /movie/adventure/1/_update
	{
	   "doc": {
	      "name": "life of pi"
	   }
	}

由于有嵌套字段,我们可以这样使用http(这里需要注意使用post方法):

	echo '{"doc": {"actors": ["Suraj", "Irrfan"]}}' | http post :9200/movie/adventure/1/_update

上面的命令中,我们添加了一个新的字段:actors,结果如下:

	HTTP/1.1 200 OK
	content-encoding: gzip
	content-type: application/json; charset=UTF-8
	transfer-encoding: chunked
	
	{
	    "_id": "1",
	    "_index": "movie",
	    "_shards": {
	        "failed": 0,
	        "successful": 1,
	        "total": 2
	    },
	    "_type": "adventure",
	    "_version": 3,
	    "result": "updated"
	}

可以看到,_version增加了1,result的结果是updated.

## 检索文档 ##
### 检索某个文档 ###
要检索某个文档很简单,我们只需要使用get请求并指出文档的index,type,id就可以了,比如:

	http :9200/movie/adventure/1/

响应内容会包含文档的元信息,文档的原始数据存在_source字段中.  
我们也可以直接检索出文档的_source字段,如下:  

	http :9200/movie/adventure/1/_source

### 检索所有文档 ###
我们可以使用-search这个api检索出所有的文档,命令如下:

	http :9200/movie/adventure/_search

返回结果如下:

	{
	    "_shards": {
	        "failed": 0,
	        "successful": 5,
	        "total": 5
	    },
	    "hits": {
	        "hits": [
	            {
	                "_id": "1",
	                "_index": "movie",
	                "_score": 1.0,
	                "_source": {
	                    "actors": [
	                        "Suraj",
	                        "Irrfan"
	                    ],
	                    "name": "Life of Pi"
	                },
	                "_type": "adventure"
	            }
	        ],
	        "max_score": 1.0,
	        "total": 1
	    },
	    "timed_out": false,
	    "took": 299
	}

可以看到,hits这个object包含了hits数组,total等字段,其中,hits数组包含了所有的文档,这里只有一个文档,total表明了文档的数量,默认情况下会返回前10个结果.我们也可以设定**from/size**参数来获取某一范围的文档 比如:

	http :9200/movie/adventure/_search?from=1&size=5

当不指定from和size时,会使用默认值,其中from的默认值是0,size的默认值是10.

### 检索某些字段 ###
有时候,我们只需检索文档的个别字段,这时可以使用**_source**参数,多个子弹可以使用逗号分隔,如下所示:

	http :9200/movie/adventure/1?_source=name
	http :9200/movie/adventure/1?_source=name,actors

### query string 搜索 ###
query string 搜索以**q=field:value**的形式进行查询,比如查询name字段含有life的电影:

	http :9200/movie/adventure/_search?q=name:life

### DSL搜索 ###
上面的query string搜索比较轻量级,只适用于简单的场合.Elasticsearch提供了更为强大的DSL(Domain Specific Language)查询语言,适用于复杂的搜索场景,比如全文搜索.我们可以将上面的query string 搜索转换为DSL搜索,如下:

	GET /movie/adventure/_search
	{
	    "query" : {
	        "match" : {
	            "name" : "life"
	        }
	    }
	}

如果使用httpie,可以这样:

	echo '{"query": {"match": {"name": "life"}}}' | http get :9200/movie/adventure/_search

如果使用curl,可以这样:

	curl -X GET "127.0.0.1:9200/movie/adventure/_search" -d '{"query": {"match": {"name": "life"}}}'

## 文档是否存在 ##
使用head方法查看文档是否存在:

	http head :9200/movie/adventure/1

如果文档存在则返回200,否则返回404.

## 删除文档 ##
使用delete方法删除文档:

	http delete :9200/movie/adventure/1

## 小结 ##
- Elasticsearch通过简单的Restful api 来隐藏lucene的复杂性,从而让全文搜索变得简单
- 在创建文档时,我们可以用post方法指定将文档添加到某个**_index/_type**下,来让Elasticsearch自动生成唯一的**_id**,而用put方法指定将文档的**_index/_type/_id**

