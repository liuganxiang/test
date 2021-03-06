## Installing solr on macOS
```shell
$ brew install solr
$ solr -version
7.7.1
```

## 基本概念 
### コレクション
インデックスを作って検索するドキュメント集合の単位。
### シャード数
- コレクションの論理的なパーティションで、ドキュメントのインデックス作成時にどのシャードに属するのかが決められる。
- クエリは各シャードにルーティングしてそれぞれで検索される。
### レプリカ数
- レプリカはシャードごとに設定し、各シャードは1つ以上のレプリカを持つ。
- レプリカの1つはleaderに指名され、leaderはインデックスの更新を他のreplicaに伝搬する。

### Starting solr running in SolrCloud mode
```shell
$ which solr
/usr/local/bin/solr
$ solr start -e cloud
```
- Running two nodes using 8983 and 7574 ports
- two shards
- two replicas per shard
- When SolrCloud example running, visit http://localhost:8983/solr

### Other commands
- Removing existing collection
```shell
$ solr delete -c [collection]
```
- Create a new collection
```shell
$ solr create -c <yourCollection> -s 2 -rf 2
```
- Stoping all services
```shell
$ solr stop -all
```
- (if you had stopped solr) Restarting node1
```shell
$ solr start -c -p 8983 -s example/cloud/node1/solr
```
- When it’s done start the second node, and tell it how to connect to to ZooKeeper:
```shell
$ solr start -c -p 7574 -s example/cloud/node2/solr -z localhost:9983
```

## Indexing and searching using exampledocs
### Indexing
At first, we can make indexes using exampledocs
```shell
$ cd /usr/local/Cellar/solr/7.7.1
$ bin/post -c techproducts example/exampledocs/*
...omitted...
SimplePostTool version 5.0.0
Posting files to [base] url http://localhost:8983/solr/techproducts/update...
Entering auto mode. File endings considered are xml,json,jsonl,csv,pdf,doc,docx,ppt,pptx,xls,xlsx,odt,odp,ods,ott,otp,ots,rtf,htm,html,txt,log
POSTing file books.csv (text/csv) to [base]
POSTing file books.json (application/json) to [base]/json/docs
POSTing file gb18030-example.xml (application/xml) to [base]
POSTing file hd.xml (application/xml) to [base]
POSTing file ipod_other.xml (application/xml) to [base]
POSTing file ipod_video.xml (application/xml) to [base]
POSTing file manufacturers.xml (application/xml) to [base]
POSTing file mem.xml (application/xml) to [base]
POSTing file money.xml (application/xml) to [base]
POSTing file monitor.xml (application/xml) to [base]
POSTing file monitor2.xml (application/xml) to [base]
POSTing file more_books.jsonl (application/json) to [base]/json/docs
POSTing file mp500.xml (application/xml) to [base]
POSTing file post.jar (application/octet-stream) to [base]/extract
POSTing file sample.html (text/html) to [base]/extract
POSTing file sd500.xml (application/xml) to [base]
POSTing file solr-word.pdf (application/pdf) to [base]/extract
POSTing file solr.xml (application/xml) to [base]
POSTing file test_utf8.sh (application/octet-stream) to [base]/extract
POSTing file utf8-example.xml (application/xml) to [base]
POSTing file vidcard.xml (application/xml) to [base]
21 files indexed.
COMMITting Solr index changes to http://localhost:8983/solr/techproducts/update...
Time spent: 0:00:43.292
```
The exampledocs have been indexed and we can start to search.

### Searching
Now we can try the following 5 types of search:
- Basic searching
	- Via Solr Admin UI
	` http://localhost:8983/solr/#/techproducts/query`
	- Via curl
	`curl "http://localhost:8983/solr/techproducts/select?indent=on&q=*:*"`
- Search for a Single Term
Just like this:
```shell
$ curl "http://localhost:8983/solr/techproducts/select?q=foundation"
{
  "responseHeader":{
    "zkConnected":true,
    "status":0,
    "QTime":651,
    "params":{
      "q":"foundation"}},
  "response":{"numFound":4,"start":0,"maxScore":2.6861532,"docs":[
      {
        "id":"0553293354",
        "cat":["book"],
        "name":"Foundation",
        "price":7.99,
        "price_c":"7.99,USD",
        "inStock":true,
        "author":"Isaac Asimov",
        "author_s":"Isaac Asimov",
        "series_t":"Foundation Novels",
        "sequence_i":1,
        "genre_s":"scifi",
        "_version_":1628664318309433344,
        "price_c____l_ns":799},
      {
        "id":"UTF8TEST",
        "name":"Test with some UTF-8 encoded characters",
	...omitted..}]
}
```

As another usage, we can put "id" (without quotes) to the "fl" field so as to 
matching return records only contain id fields.
```shell
$ curl "http://localhost:8983/solr/techproducts/select?q=foundation&fl=id"
{
  "responseHeader":{
    "zkConnected":true,
    "status":0,
    "QTime":446,
    "params":{
      "q":"foundation",
      "fl":"id"}},
  "response":{"numFound":4,"start":0,"maxScore":2.6861532,"docs":[
      {
        "id":"0553293354"},
      {
        "id":"UTF8TEST"},
      {
        "id":"SOLR1000"},
      {
        "id":"/usr/local/Cellar/solr/7.7.1/example/exampledocs/test_utf8.sh"}]
  }}
```

- Field Searches
For example, we can limit the our search for only documents with the category "electronics",
the results will be more precise for us.
`curl "http://localhost:8983/solr/techproducts/select?q=cat:electronics"`

- Phrase Search
To search for a multi-term phrase, enclose it in double quotes: q="multiple terms here". 
If we’re using curl, note that the space between terms must be converted to "+" in a URL, as so:
`curl "http://localhost:8983/solr/techproducts/select?q=\"CAS+latency\""`

- Combining Searches
A term or phrase is present by prefixing it with a +; conversely, to disallow the presence of a term or phrase, prefix it with a -.
	- To find documents that contain both terms "electronics" and "music"
	`curl "http://localhost:8983/solr/techproducts/select?q=%2Belectronics%20%2Bmusic"`

	- To find documents that contain term "electronics" but don’t contain the term "music"
	`curl "http://localhost:8983/solr/techproducts/select?q=%2Belectronics+-music"`

## Indexing and searching using contents of my blog
- At first, let's delete collection techproducts and then create a new one
```shell
$ solr delete -c techproducts
$ solr create -c gxliublog -s 2 -rf 2 -d $SOLRHOME/server/solr/configsets/sample_techprodfucts_configs/conf/
```
Note: If I don't use configsets of sample_techprodfucts_configs, I can't create indexes successfully, why?!
- Writing a python srcipt named getContents.py as below
```shell
import urllib.request
import os.path
import time

urls = [
    'http://blog.livedoor.jp/gxliu/archives/278227.html',
    'http://blog.livedoor.jp/gxliu/archives/312679.html',
    'http://blog.livedoor.jp/gxliu/archives/613538.html',
    'http://blog.livedoor.jp/gxliu/archives/767863.html',
    'http://blog.livedoor.jp/gxliu/archives/776386.html',
    'http://blog.livedoor.jp/gxliu/archives/1003197.html',
    'http://blog.livedoor.jp/gxliu/archives/1012986.html',
    'http://blog.livedoor.jp/gxliu/archives/1152128.html',
    'http://blog.livedoor.jp/gxliu/archives/1244051.html',
    'http://blog.livedoor.jp/gxliu/archives/1436671.html',
    'http://blog.livedoor.jp/gxliu/archives/1454963.html',
    'http://blog.livedoor.jp/gxliu/archives/1475006.html',
    'http://blog.livedoor.jp/gxliu/archives/1577126.html',
    'http://blog.livedoor.jp/gxliu/archives/1583854.html',
    'http://blog.livedoor.jp/gxliu/archives/1756964.html',
    'http://blog.livedoor.jp/gxliu/archives/1757080.html',
    'http://blog.livedoor.jp/gxliu/archives/1877291.html',
    'http://blog.livedoor.jp/gxliu/archives/2049060.html'
]

dir = '/tmp/solr/documents'
for url in urls:
	filename = os.path.basename(url)
	urllib.request.urlretrieve(url, dir + '/' + filename)
	time.sleep(1) 

exit(0)
```
- The files downloaded are below
```shell
$ ls /tmp/solr/downloads
1003197.html  1436671.html  1583854.html  2049060.html  767863.html
1012986.html  1454963.html  1756964.html  278227.html   776386.html
1152128.html  1475006.html  1757080.html  312679.html
1244051.html  1577126.html  1877291.html  613538.html
```
### Indexing
- Cleansing the content datas
```shell
$ for x in `ls`; do sed -e 's/^M//g' -e 's/EUC-JP/utf-8/g' -e 's/euc-jp/utf-8/g' $x > aaa; iconv -f euc-jp -t utf-8 aaa > $x; done
```
- Indexing
```shell
$ which post
/usr/local/bin/post
$ /usr/local/bin/post -c gxliublog /tmp/solr/documents/*
/Library/Java/JavaVirtualMachines/jdk1.8.0_05.jdk/Contents/Home/bin/java -classpath /usr/local/Cellar/solr/7.7.1/libexec/dist/solr-core-7.7.1.jar -Dauto=yes -Dc=gxliublog -Ddata=files org.apache.solr.util.SimplePostTool ./1003197.html ./1012986.html ./1152128.html ./1244051.html ./1436671.html ./1454963.html ./1475006.html ./1577126.html ./1583854.html ./1756964.html ./1757080.html ./1877291.html ./2049060.html ./278227.html ./312679.html ./613538.html ./767863.html ./776386.html
SimplePostTool version 5.0.0
Posting files to [base] url http://localhost:8983/solr/gxliublog/update...
Entering auto mode. File endings considered are xml,json,jsonl,csv,pdf,doc,docx,ppt,pptx,xls,xlsx,odt,odp,ods,ott,otp,ots,rtf,htm,html,txt,log
POSTing file 1003197.html (text/html) to [base]/extract
POSTing file 1012986.html (text/html) to [base]/extract
POSTing file 1152128.html (text/html) to [base]/extract
POSTing file 1244051.html (text/html) to [base]/extract
POSTing file 1436671.html (text/html) to [base]/extract
POSTing file 1454963.html (text/html) to [base]/extract
POSTing file 1475006.html (text/html) to [base]/extract
POSTing file 1577126.html (text/html) to [base]/extract
POSTing file 1583854.html (text/html) to [base]/extract
POSTing file 1756964.html (text/html) to [base]/extract
POSTing file 1757080.html (text/html) to [base]/extract
POSTing file 1877291.html (text/html) to [base]/extract
POSTing file 2049060.html (text/html) to [base]/extract
POSTing file 278227.html (text/html) to [base]/extract
POSTing file 312679.html (text/html) to [base]/extract
POSTing file 613538.html (text/html) to [base]/extract
POSTing file 767863.html (text/html) to [base]/extract
POSTing file 776386.html (text/html) to [base]/extract
18 files indexed.
COMMITting Solr index changes to http://localhost:8983/solr/gxliublog/update...
Time spent: 0:00:30.977
```
- So far, the preparation for searching is completed

### Searching
- Search for "Windows" and get id field
```shell
$ curl "http://localhost:8983/solr/gxliublog/select?q=Windows&fl=id"
{
  "responseHeader":{
    "zkConnected":true,
    "status":0,
    "QTime":261,
    "params":{
      "q":"Windows",
      "fl":"id"}},
  "response":{"numFound":18,"start":0,"maxScore":0.17436156,"docs":[
      {
        "id":"/private/tmp/solr/documents/./776386.html"},
      {
        "id":"/private/tmp/solr/documents/./1877291.html"},
      {
        "id":"/private/tmp/solr/documents/./1152128.html"},
      {
        "id":"/private/tmp/solr/documents/./1454963.html"},
      {
        "id":"/private/tmp/solr/documents/./1003197.html"},
      {
        "id":"/private/tmp/solr/documents/./613538.html"},
      {
        "id":"/private/tmp/solr/documents/./1583854.html"},
      {
        "id":"/private/tmp/solr/documents/./767863.html"},
      {
        "id":"/private/tmp/solr/documents/./312679.html"},
      {
        "id":"/private/tmp/solr/documents/./1012986.html"}]
  }}
```

