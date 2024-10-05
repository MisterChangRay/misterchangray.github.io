---
layout: post
title:  "easypoi导出多sheet的excel"
date:   2024-10-05 13:29:20 +0800
categories:
      - java
tags:
      - excel导出
      - 多sheetExcel导出
---

项目有个导出需求，需要导出多个sheet，xlsx格式的excel表格。

这里简单说明下，在excel中，有xls,xlsx两种格式。

- xls是excel2005版本以下的文件格式，支持数据量小而且格式很老。
- xlsx是2005版本以上支持的格式，数据量大而且新特性多

我觉得都2024年了，大家导出excel应该都使用xlsx格式了吧。

在网上找了一圈没有什么好使的代码（其实找到了一个xls的导出代码，可以直接使用）。但是xlsx的却没找到现成的，于是基于xls的修改了下，这里直接贴出可用的，大家拿去直接食用吧。

1. 引入依赖pom
```xml
       <dependency>
            <groupId>cn.afterturn</groupId>
            <artifactId>easypoi-base</artifactId>
            <version>4.5.0</version>
        </dependency>
       <dependency>
            <groupId>cn.afterturn</groupId>
            <artifactId>easypoi-web</artifactId>
            <version>4.5.0</version>
        </dependency>
	   <dependency>
	        <groupId>cn.afterturn</groupId>
	        <artifactId>easypoi-annotation</artifactId>
	        <version>4.5.0</version>
	    </dependency>

```

2. 导入上面的依赖后，样例java代码如下:

大概说明下，以下代码中定义了两个sheet， 并导出到`test.xlsx`文件中
```java

		import cn.afterturn.easypoi.excel.entity.ExportParams;
		import cn.afterturn.easypoi.excel.entity.enmus.ExcelType;
		import cn.afterturn.easypoi.excel.export.ExcelExportService;
		import org.apache.poi.ss.usermodel.Workbook;
		import org.apache.poi.xssf.streaming.SXSSFWorkbook;
		import org.apache.poi.xssf.usermodel.XSSFWorkbook;
		import org.apache.poi.xssf.usermodel.XSSFWorkbookType;

		// java导出代码
		List<Data1> data1List = new ArrayList();
		List<Data2> data2List = new ArrayList();
		Workbook workbook = new XSSFWorkbook(XSSFWorkbookType.XLSX);
		ExportParams exportParams0 = new ExportParams();
		exportParams0.setSheetName("sheet1");
		exportParams0.setType(ExcelType.XSSF);
		ExportParams exportParams1 = new ExportParams();
		exportParams1.setSheetName("shhet2");
		exportParams1.setType(ExcelType.XSSF);

		ExcelExportService service0 = new ExcelExportService();
		// 这里将data1List集合导出到excel,sheet1中
		service0.createSheet(workbook, exportParams0, Data1.class, data1List);
		ExcelExportService service1 = new ExcelExportService();
		// 这里将data2List集合导出到excel,sheet2中
		service1.createSheet(workbook, exportParams1, Data2.class, data2List);


		FileOutputStream fos = null;
		try {
		    // 写出文件
			File f = new File("./test.xlsx");
			f.createNewFile();
			fos = new FileOutputStream(f);
			workbook.write(fos);
			fos.close();
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
```

最后一点，定义导出实体类时需要注意, 所有的导出字段需要指定`width`属性，不然当前版本会报错。

```java
@Data
@ExcelTarget(value = "Data1")
public class Data1 implements Serializable {
    /**
     * 审核状态
     * commAuditStatus
     */
    @Excel(name = "审核状态", width = 20)
    private String commAuditStatus;

    /**
     * 申请单号
     * billId
     */
    @Excel(name = "申请单号", width = 20)
    private String billId;
}


@Data
@ExcelTarget(value = "Data2")
public class Data2 implements Serializable {
    /**
     * 审核状态2
     * commAuditStatus
     */
    @Excel(name = "审核状态2", width = 20)
    private String commAuditStatus;

    /**
     * 申请单号2
     * billId
     */
    @Excel(name = "申请单号2", width = 20)
    private String billId;
}
```

上面代码粘贴即可直接食用，好用点个赞！
