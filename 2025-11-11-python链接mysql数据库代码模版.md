---
layout: post
title:  "python操作mysql数据库代码模版"
date:   2025-11-11 17:29:20 +0800
categories:
      - python
      - 代码模版
tags:
      - python代码模版
---

这里只是一个代码模版, 记录了一些常用代码片段，方便copy

#### pymysql 操作mysql

```
import os

def dataparse(db):
    cur = db.cursor()
    # 执行的都是原生SQL语句
    cur.execute("SELECT * FROM t_rent_info")
    orders = cur.fetchall()
    for row in orders:
        # print(row['id'])
        orderno = row['order_no']

        am = None
        adate = None
        cur.execute("SELECT * FROM t_pay_order where type = 2 and status=2 and order_id = '{}' order by id desc limit 1".format(orderno))
        payor = cur.fetchone()
        if(payor != None):
            period1 = payor['period']
            if(period1 % 3 == 1):
                am = row['rent_amount'] * 2
                adate = payor['pay_time']
            if(period1 % 3 == 0):
                am = 0
                adate = payor['pay_time']
            if(period1 % 3 == 2):
                am = row['rent_amount'] 
                adate = payor['pay_time']
            
        if(adate != None)        :
            cur.execute("update t_rent_info set platform_pre_pay_amount = {}, platform_pre_pay_time='{}' where order_no = '{}'".format( am, adate,  orderno))
            print("更新订单", orderno, am, adate)

        

    db.commit()
    db.close()
    pass

def start():
    arg_kwargs={
    'host':"192.168.0.185",
    'port':3306,
    'user':'root',
    'password':"Sws23@w312d",
    'database':"house_db",
    'charset':'utf8',
    'cursorclass':pymysql.cursors.DictCursor
    }
    db=pymysql.connections.Connection(**arg_kwargs)#pymysql.connections.Connection对象
    dataparse(db)
    pass

if __name__ == '__main__':
    start()


```
