## 参考网站 ##
[https://es.xiaoleilu.com/index.html](https://es.xiaoleilu.com/index.html)  
[https://www.cnblogs.com/lizichao1991/p/7809156.html](https://www.cnblogs.com/lizichao1991/p/7809156.html)  
[https://blog.csdn.net/laotoumo/article/details/53890279](https://blog.csdn.net/laotoumo/article/details/53890279)
## Elasticsearch入门使用 ##
Elasticsearch是一款稳定高效的分布式搜索和分析引擎,它的底层基于lucene,并提供了友好的Restful api来对数据进行操作,还有比较重要的一点是,Elasticsearch开箱即可用,上手也比较容易.  
目前Elasticsearch在搭建企业级搜索(如日志搜索,商品搜索等)平台中很广泛,官网也提供了不少案例,比如:

- GitHub使用Elasticsearch检索超过800万的代码库
- eBay使用Elasticsearch搜索海量的商品数据
- Netflix使用Elasticsearch来实现高效的消息传递系统

## 概念 ##
在进一步使用Elasticsearch之前,我们先了解几个关键概念.

- **Index(索引):**即索引，索引包含一堆有相似结构的文档数据。一般来讲，我们将数据类似一样或者相近的数据才包装为一个Index。就像是关系数据库中的一个Database(这个必须小写，也别用下划线开头)
- **Document(文档):**ES存储内容的基本单位，相当于一条数据（在项目中，它可以是一条订单数据、一条JSON数据等等)
- **Type(类型):**对Document的分类，在一个Index中，会根据一些特性的不同，建立不同的Type去进行逻辑数据分类。6.X以后的版本只允许有一个Type，这个需要注意.就像是关系数据库中的一个Table
- **Field(字段):**每个文档包含多个字段,类似关系数据库中一个Table的列
- **Shard(分片):**数据分片,一个index可能会存在于多个shard
- **Node(节点):**node是一个运行这的Elasticsearch实例,一个node就是一个单独的server
- **Cluster(集群):**cluster是多个node集群

我们用一个表格来做类比,如下:
	  
|Elasticsearch |	MySQL |  
|--|--|  
|Index|	Database|  
|Type|	Table|  
|Document|	Row|  
|Field	|Column|  


## 安装 ##
在安装Elasticsearch之前,请确保你的计算机已经安装了Java.

下载最新版本的Elasticsearch ,使用wget下载,如下:

	wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.4.2-linux-x86_64.tar.gz

下载完成后进行解压:

	tar -xzf elasticsearch-7.4.2-linux-x86_64.tar.gz

首先,我们进入到刚刚解压出来的目录中:

	cd elasticsearch-7.4.2/

接着 使用如下命令启动Elasticsearch

	./bin/elasticsearch

> 后台启动命令  **./bin/elasticsearch -d**

测试是否启动成功,打开另一个终端进行测试：访问链接 **http://localhost:9200/**,或使用curl命令:

	curl 'http://localhost:9200/?pretty'

我们可以看到类似如下的输出:

	{
	  "name" : "node01",
	  "cluster_name" : "elasticsearch",
	  "cluster_uuid" : "D22e0tvHSyy0_pRQmm03WQ",
	  "version" : {
	    "number" : "7.4.2",
	    "build_flavor" : "default",
	    "build_type" : "tar",
	    "build_hash" : "2f90bbf7b93631e52bafb59b3b049cb44ec25e96",
	    "build_date" : "2019-10-28T20:40:44.881551Z",
	    "build_snapshot" : false,
	    "lucene_version" : "8.2.0",
	    "minimum_wire_compatibility_version" : "6.8.0",
	    "minimum_index_compatibility_version" : "6.0.0-beta1"
	  },
	  "tagline" : "You Know, for Search"
	}

可以在配置文件config/elasticsearch.yml 修改name和cluster_name，注意配置中间留空格。修改后重新启动es，可以看到配置的name和cluster.name。

	{
	  "name" : "node01",
	  "cluster_name" : "chenkun",
	  "cluster_uuid" : "D22e0tvHSyy0_pRQmm03WQ",
	  "version" : {
	    "number" : "7.4.2",
	    "build_flavor" : "default",
	    "build_type" : "tar",
	    "build_hash" : "2f90bbf7b93631e52bafb59b3b049cb44ec25e96",
	    "build_date" : "2019-10-28T20:40:44.881551Z",
	    "build_snapshot" : false,
	    "lucene_version" : "8.2.0",
	    "minimum_wire_compatibility_version" : "6.8.0",
	    "minimum_index_compatibility_version" : "6.0.0-beta1"
	  },
	  "tagline" : "You Know, for Search"
	}

### 安装中遇到的错误&解决方案 ###
1. max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]  
原因： 意思是说你的进程不够用了  
解决方案： 切到root 用户：进入到security目录下的limits.conf；  
执行命令 vim /etc/security/limits.conf 在文件的末尾添加下面的参数值： 

		* soft nofile 65536
		* hard nofile 131072
		* soft nproc 2048
		* hard nproc 4096

2. max number of threads [3803] for user [es] is too low, increase to at least [4096]  
原因：意思就是说你的线程数不够用了  
解决方案： 切到root用户,执行命令 vi /etc/security/limits.d/20-nproc.conf 修改3803为4096:  


		*          soft    nproc     4096
		root       soft    nproc     unlimited


	如果还是失败（大多出现在 Centos7 以上），换下面这种：

		* hard nproc 4096
		* soft nproc 4096
		elk soft nproc 4096
		root soft nproc unlimited

3. max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]  
原因： 需要修改系统变量的最大值  
解决方案：切换到 root 用户修改配置 /etc/sysctl.conf 增加配置值：vm.max_map_count=655360  
执行命令 sysctl -p 这样就可以了，会显示如下信息

		[root@localhost ~]#    sysctl -p
		vm.max_map_count = 262144

4. system call filters failed to install; check the logs and fix your configuration or disable system call filters at your own risk  
问题原因：因为Centos6不支持SecComp，而ES5.2.1默认  bootstrap.system_call_filter为true进行检测，所以导致检测失败，失败后直接导致ES不能启动。详见 ：[https://github.com/elastic/elasticsearch/issues/22899](https://github.com/elastic/elasticsearch/issues/22899)  
解决方法：在elasticsearch.yml中配置bootstrap.system_call_filter为false，注意要在Memory下面:

		bootstrap.memory_lock: false
		bootstrap.system_call_filter: false

5. 安装es head 部分的错误参考网址进行解决

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

	curl -H "Content-Type: application/json" -i -XPOST "localhost:9200/mv/adv/1" -d '{"name": "Life of Pi", "actors": ["Suraj", "Irrfan"]}'


上面的命令结果如下:

	{
	  "_index": "mv",
	  "_type": "adv",
	  "_id": "1",
	  "_version": 1,
	  "result": "created",
	  "_shards": {
	    "total": 2,
	    "successful": 1,
	    "failed": 0
	  },
	  "_seq_no": 1,
	  "_primary_term": 1
	}

可以看到,我们已经成功创建一个_index的mv,_type为adv,_id为1的文档.  
> _id不填写,自动生成自增id,请求地址为 "localhost:9200/mv/adv"

我们通过get请求来查看这个文档的信息:

	http :9200/movie/adventure/1

结果如下:

	{
		"_index": "mv",
		"_type": "adv",
		"_id": "1",
		"_version": 1,
		"_seq_no": 1,
		"_primary_term": 1,
		"found": true,
		"_source": {
			"name": "Life of Pi",
			"actors": [
				"Suraj"
				,
				"Irrfan"
			]
		}
	}

可以看到,原始的文档数据存在了_source字段中.

## 更新文档 ##
当我们使用put方法指明文档的_index,_type和_id时,如果_id已存在,则新文档会替换旧文档,此时文档的_version会增加1,并且_created字段为false.比如:

	curl -H "Content-Type: application/json" -i -XPUT "localhost:9200/mv/adv/1" -d '{"name": "JACK AND ROSE"}'

结果如下:

	{
	  "_index": "mv",
	  "_type": "adv",
	  "_id": "1",
	  "_version": 2,
	  "result": "updated",
	  "_shards": {
	    "total": 2,
	    "successful": 1,
	    "failed": 0
	  },
	  "_seq_no": 2,
	  "_primary_term": 1
	}

使用get请求查看新文档的数据:

	curl -H "Content-Type: application/json" -i -XGET "localhost:9200/mv/adv/1"

结果如下:

	{
	  "_index": "mv",
	  "_type": "adv",
	  "_id": "1",
	  "_version": 2,
	  "_seq_no": 2,
	  "_primary_term": 1,
	  "found": true,
	  "_source": {
	    "name": "JACK AND ROSE"
	  }
	}

可以看到,actors这个字段已经不存在了,文档的_version变成了2.  
因此,为了避免在误操作的情况下,原文档被替换,我们可以使用_create这个api,表示只在文档不存在的情况下才创建新文档(返回201 created),如果文档存在则不做任何操作(返回409 conflict),命令如下:

	curl -H "Content-Type: application/json" -i -XPUT "localhost:9200/mv/adv/1/_create" -d '{"name": "JACK AND ROSE"}'
	

由于文档id存在,会返回409 conflict.

## 局部更新 ##
在有些情况下,我们只想更新文档的局部,而不是整个文档,这是我们可以使用_update这个api.  

现在,待更新的文档信息如下:

	{
	  "_index": "mv",
	  "_type": "adv",
	  "_id": "1",
	  "_version": 2,
	  "_seq_no": 2,
	  "_primary_term": 1,
	  "found": true,
	  "_source": {
	    "name": "JACK AND ROSE"
	  }
	}

最简单的update请求接受一个局部文档参数**doc**,它会合并到现有文档中:将对象合并在一起,存在的标量字段被覆盖,新字段被添加.

形式如下:

	curl -H "Content-Type: application/json" -i -XPOST "localhost:9200/mv/adv/1/_update" -d '{"doc": {"actors": ["Suraj", "Irrfan"]}}'

	
上面的命令中,我们添加了一个新的字段:actors,结果如下:

	{
	  "_index": "mv",
	  "_type": "adv",
	  "_id": "1",
	  "_version": 3,
	  "result": "updated",
	  "_shards": {
	    "total": 2,
	    "successful": 1,
	    "failed": 0
	  },
	  "_seq_no": 4,
	  "_primary_term": 1
	}

可以看到,_version增加了1,result的结果是updated.

## 检索文档 ##
### 检索某个文档 ###
要检索某个文档很简单,我们只需要使用get请求并指出文档的index,type,id就可以了,比如:

	curl -H "Content-Type: application/json" -i -XGET "localhost:9200/mv/adv/1"

响应内容会包含文档的元信息,文档的原始数据存在_source字段中.  
我们也可以直接检索出文档的_source字段,如下:  

	curl -H "Content-Type: application/json" -i -XGET "localhost:9200/mv/adv/1/_source"

### 检索所有文档 ###
我们可以使用-search这个api检索出所有的文档,命令如下:

	curl -H "Content-Type: application/json" -i -XGET "localhost:9200/mv/adv/_search"

返回结果如下:

	{
	  "took": 1,
	  "timed_out": false,
	  "_shards": {
	    "total": 1,
	    "successful": 1,
	    "skipped": 0,
	    "failed": 0
	  },
	  "hits": {
	    "total": {
	      "value": 3,
	      "relation": "eq"
	    },
	    "max_score": 1,
	    "hits": [
	      {
	        "_index": "mv",
	        "_type": "adv",
	        "_id": "eCNNpG4B2qkb5WRgughp",
	        "_score": 1,
	        "_source": {
	          "name": "Life of Pi",
	          "actors": [
	            "Suraj",
	            "Irrfan"
	          ]
	        }
	      },
	      {
	        "_index": "mv",
	        "_type": "adv",
	        "_id": "2",
	        "_score": 1,
	        "_source": {
	          "name": "JACK AND ROSE"
	        }
	      },
	      {
	        "_index": "mv",
	        "_type": "adv",
	        "_id": "1",
	        "_score": 1,
	        "_source": {
	          "name": "JACK AND ROSE",
	          "actors": [
	            "Suraj",
	            "Irrfan"
	          ]
	        }
	      }
	    ]
	  }
	}

可以看到,hits这个object包含了hits数组,total等字段,其中,hits数组包含了所有的文档,这里只有一个文档,total表明了文档的数量,默认情况下会返回前10个结果.我们也可以设定**from/size**参数来获取某一范围的文档 比如:

	curl -H "Content-Type: application/json" -i -XGET "localhost:9200/mv/adv/_search?from=1&size=5"

当不指定from和size时,会使用默认值,其中from的默认值是0,size的默认值是10.

### 检索某些字段 ###
有时候,我们只需检索文档的个别字段,这时可以使用**_source**参数,多个子弹可以使用逗号分隔,如下所示:

	curl -H "Content-Type: application/json" -i -XGET "localhost:9200/mv/adv/1?_source=name"

	curl -H "Content-Type: application/json" -i -XGET "localhost:9200/mv/adv/1?_source=name,actors"

### query string 搜索 ###
query string 搜索以**q=field:value**的形式进行查询,比如查询name字段含有life的电影:

	curl -H "Content-Type: application/json" -i -XGET "localhost:9200/mv/adv/_search?q=name:JACK"

### DSL搜索 ###
上面的query string搜索比较轻量级,只适用于简单的场合.Elasticsearch提供了更为强大的DSL(Domain Specific Language)查询语言,适用于复杂的搜索场景,比如全文搜索.我们可以将上面的query string 搜索转换为DSL搜索,如下:

	curl -H "Content-Type: application/json" -i -XGET "localhost:9200/mv/adv/_search" -d '{"query" : {"match" : {"name" : "JACK"}}}'

## 文档是否存在 ##
使用head方法查看文档是否存在:

	curl -H "Content-Type: application/json" -i -XHEAD "localhost:9200/mv/adv/2"	

如果文档存在则返回200,否则返回404.

## 删除文档 ##
使用delete方法删除文档:

	curl -H "Content-Type: application/json" -i -XDELETE "localhost:9200/mv/adv/2"

## 小结 ##
- Elasticsearch通过简单的Restful api 来隐藏lucene的复杂性,从而让全文搜索变得简单
- 在创建文档时,我们可以用post方法指定将文档添加到某个**_index/_type**下,来让Elasticsearch自动生成唯一的**_id**,而用put方法指定将文档的**_index/_type/_id**

