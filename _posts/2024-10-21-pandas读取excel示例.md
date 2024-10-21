---
layout: post
title:  "pandas读取excel示例"
date:   2024-10-21 17:29:20 +0800
categories:
      - python
      - pandas
tags:
      - excel
---

有时候用excel处理数据。

虽然一直在用python写代码，不过好记性确实不如烂笔头，哈哈，备注下。


pandas读写excel

```python


import pandas as pd
import pyperclip


def main():
    excel = pd.ExcelFile('C:\\Users\\ray\\Desktop\\sharefiles\\移远咻电10月到期.xlsx')
    # 读取第0个sheet, 当然可以直接传sheet表名为参数
    df = excel.parse(0, header=1)
    # 取第1到2列的所有列为imei
    # iloc函数参数, iloc[行起始位置:行结束位置,列起始位置:列结束位置]
    imeis =  df.iloc[:,1:2]
    values = imeis.values
    res = "select `sim_code` ,`sn_id` ,`model`  from `cp_equipment_metadata` WHERE `sim_code`  in ("
    for tmp in values :
        res =res + "'" + tmp[0].strip() + "',"
        pass
    
    res = res[:-1]
    res += ")"
    pyperclip.copy(res)
    print("已经拷贝到剪切板")


    pass
if __name__=="__main__":
    main()
```
