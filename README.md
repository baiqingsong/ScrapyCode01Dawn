# scrapy应用1--爬取西刺网站

* [网站首页](#网站首页)
* [创建工程](#创建工程)
* [编写item](#编写item)
* [编写spider](#编写spider)
* [编写pipeline](#编写pipeline)

## 网站首页
[http://www.xicidaili.com](http://www.xicidaili.com)

## 创建工程
命令行，在对应的文件夹下
```
scrapy startproject ScrapyCode01Dawn
```
完成工程的创建

## 编写item
需要保存的内容有ip,port,position,type,speed,last_check_time这几个字段
```
class CollectipsItem(scrapy.Item):
    IP = scrapy.Field()
    PORT = scrapy.Field()
    POSITION = scrapy.Field()
    TYPE = scrapy.Field()
    SPEED = scrapy.Field()
    LAST_CHECK_TIME = scrapy.Field()
```

## 编写spider
创建XiciSpider类，编写代码
```
# _*_ coding: utf-8 _*_
import scrapy
from ScrapyCode01Dawn.items import CollectipsItem

class XiciSpider(scrapy.Spider):
    name = "xici"
    allowed_domains = ["xicidaili.com"]
    start_urls = ["http://www.xicidaili.com"]

    def start_requests(self):
        reqs = []

        for i in range(1, 3):
            req = scrapy.Request("http://www.xicidaili.com/nn/%s" % i)
            reqs.append(req)
        return reqs

    def parse(self, response):
        ip_list = response.xpath('//table[@id="ip_list"]')
        trs = ip_list[0].xpath('tr')
        items = []

        for ip in trs[1:]:
            pre_item = CollectipsItem()
            pre_item['IP'] = ip.xpath('td[2]/text()')[0].extract()
            pre_item['PORT'] = ip.xpath('td[3]/text()')[0].extract()
            # 获取到td下面所有的内容，可能嵌套一层a标签
            pre_item['POSITION'] = ip.xpath('string(td[4])')[0].extract().strip()
            pre_item['TYPE'] = ip.xpath('td[6]/text()')[0].extract()
            # 获取到速度的数字，不要单位
            pre_item['SPEED'] = ip.xpath('td[7]/div[@class="bar"]/@title').re('\d{0,2}\.\d{0,}')[0]
            pre_item['LAST_CHECK_TIME'] = ip.xpath('td[9]/text()')[0].extract()
            items.append(pre_item)
        return items
```

## 编写pipeline
创建pipeline类
```
import MySQLdb

class CollectipsPipeline(object):

    def process_item(self, item, spider):
        DBKWARGS = spider.settings.get('DBKWARGS')
        con = MySQLdb.Connect(**DBKWARGS)
        cur = con.cursor()
        sql = ("insert into proxy(IP,PORT,TYPE,POSITION,SPEED,LAST_CHECK_TIME) "
               "values(%s,%s,%s,%s,%s,%s)")
        lis = (item['IP'],item['PORT'],item['TYPE'],item['POSITION'],item['SPEED'],item['LAST_CHECK_TIME'])
        try:
            cur.execute(sql, lis)
        except Exception,e:
            print "insert error:",e
            con.rollback()
        else:
            con.commit()
        cur.close()
        con.close()
        return item
```
其中需要注意的是在配置settings.py文件中
```
# database connection parameters
DBKWARGS={'db':'ips', 'user':'root', 'passwd':'bai910214', 'host':'localhost', 'use_unicode':True, 'charset':'utf8'}

# Configure item pipelines
# See http://scrapy.readthedocs.org/en/lastest/topics/item-pipeline.html
ITEM_PIPELINES = {
    'ScrapyCode01Dawn.pipelines.CollectipsPipeline':300,
}

# Configure log file name
LOG_FILE = "scrapy.log"

# Crawl responsibly by identifying yourself (and your website) on the user-agent
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/62.0.3202.94 Safari/537.36'

# Obey robots.txt rules
ROBOTSTXT_OBEY = False
```
添加数据库配置和pipeline的引用,并且添加请求方和robot不检查