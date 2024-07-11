---
layout: post
title:  "python_fastAPI脚手架"
date:   2024-07-07 16:29:20 +0800
categories:
      - python
      - 脚手架
tags:
      - fastapi
      
---

开源了一个脚手架项目，适合webapi开发。
git地址:`https://github.com/MisterChangRay/fastapi-demo/`

下面适合开发一些简单得项目
---------------------

很简单得东西,留着自己用得。不做过多介绍了。

主要是整合了 fastapi, SQLAlchemy，日志，mysql, 以及一些代码示例。 留着以后快速开发一些玩具项目。

部署服务器后，使用以下命令使用：

```sh
# 安装依赖
pip install -r requirements.txt

# 启动服务
uvicorn server:app --port=8078

# 启动后自动更新服务，开发使用
uvicorn server:app --port=8078 -reload
```

脚本源码
```python
from typing import Union
import uvicorn
from fastapi import FastAPI
from fastapi import FastAPI
import os
from pydantic import BaseModel
import logging
import sys
from typing import Annotated
import requests
from fastapi import FastAPI, Form
import json
import time
from urllib.parse import unquote
import xmltodict
import datetime
from sqlalchemy import Boolean, Column, ForeignKey, Integer, String,DateTime, update
from sqlalchemy.pool import QueuePool
from fastapi import BackgroundTasks, FastAPI
import schedule
import threading
from loguru import logger
# 定义日志文件名和大小, 超过了新建一个文件
logger.add("./logs/mylog.log", rotation="50 MB")
from apscheduler.schedulers.background import BackgroundScheduler
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from sqlalchemy.orm import Session


# 数据库连接信息
SQLALCHEMY_DATABASE_URL = "mysql+pymysql://root:2vKeG&1.3@47.109.108.16:7501/datasyc"
engine = create_engine(
    url=SQLALCHEMY_DATABASE_URL, poolclass=QueuePool
    # , echo=True
)


# 定义数据库model
# 表明及字段申明, 需要提前创建数据库及表
Base = declarative_base()
class TData(Base):
    __tablename__ = "t_data"

    id = Column(Integer, primary_key=True)
    param = Column(String)
    status = Column(Integer)
    create_time = Column(DateTime)
    update_time = Column(DateTime)
    

# 基本响应对象
class BaseRes(BaseModel):
    code: str
    msg: str | None = None
    
# 普通实体类对象
class Params(BaseModel):
    req: str

# 普通实体类对象
class OrderInfo(BaseModel):
    # 商户号
    mchnt_cd: str
    # 订单号
    orderNo: str
    # 订单id
    orderId:str
    # 商户id
    merchId: str
    
    # 支付日期
    payDate: str
    # 支付金额
    payAmount: str
    # 备注
    remark: str| None = None
    # 渠道订单号
    transactionId: str
    


app = FastAPI()


@app.get("/test")
async def create_item()-> BaseRes: 
    return BaseRes(code=999, msg="okk")


# 数据库新增演示
@app.post("/pay/callback")
def paycallback(req= Form()) -> str:
    logger.info("收到支付回调: {}", req)
    with Session(engine) as session:
        data = TData(param=req, status=1, create_time=datetime.datetime.now(), update_time=datetime.datetime.now())
        session.add(data)
        session.commit()
    return 1
    
    

def xml2obj(req= Form()) -> str:
    res = unquote( req)
    tmp = res.index("<xml>")
    res = """<?xml version="1.0" encoding="utf-8"?>""" + res[tmp:]
    # xml 转为对象
    res = xmltodict.parse(res, disable_entities=True)
    return False


# 数据库查询更新演示
def start():
    with Session(engine) as session:
        reslist = session.query(TData).filter(TData.status == 1, TData.update_time <= datetime.datetime.now()).limit(10)
        for tmp in list(reslist):
            stmt = update(TData).where(TData.id ==id).values(status = 2, update_time = datetime.datetime.now())
            session.execute(stmt)
        session.commit()
            


# web容器启动事件
# 这里添加一个定时任务，每3s执行一次
@app.on_event('startup')
def init_data():
    scheduler = BackgroundScheduler()
    scheduler.add_job(start, 'cron', second='*/3')
    scheduler.start()
    
# 入口函数
if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=int(8078))
    
    
```


下面是依赖 `requirements.txt`
```txt
APScheduler==3.10.4
fastapi==0.111.0
loguru==0.7.2
pydantic==1.10.2
Requests==2.32.3
schedule==1.2.2
SQLAlchemy==1.4.41
uvicorn==0.30.1
xmltodict==0.13.0

```
