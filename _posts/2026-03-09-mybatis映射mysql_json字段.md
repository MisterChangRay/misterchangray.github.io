---
layout: post
title:  "mybatis映射mysql_json字段"
date:   2026-03-09 13:29:20 +0800
categories:
      - java
tags:
      - mybatis
      - mysql8_json数据
---

阅读此文章可以解决的问题：

1. mysql json 字段查询为null
2. mybatis typehandler 查询返回为Null
3. 使用自定义 typehandler 查询返回 null


一个需求，很简单，需要储存图片，于是在 mysql 中我使用了 json 数据类型，想的是直接储存 `List<String>` ，很合理是吧。

所以先定义定义具体的`typehandler`转换器

```java

import com.alibaba.fastjson2.JSON;
import org.apache.ibatis.type.BaseTypeHandler;
import org.apache.ibatis.type.JdbcType;
import org.apache.ibatis.type.MappedJdbcTypes;
import org.apache.ibatis.type.MappedTypes;
import org.springframework.stereotype.Component;

import java.sql.CallableStatement;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.List;

@Component
@MappedTypes({List.class})
@MappedJdbcTypes(JdbcType.VARCHAR)

public class ListStringHandler extends BaseTypeHandler<List<String>> {


    public ListStringHandler() {

    }
    public ListStringHandler(Class<java.lang.String> type) {

    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, List<java.lang.String> parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, JSON.toJSONString(parameter));
    }

    @Override
    public List<String> getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String value = rs.getString(columnName);
        if (value != null)
            return JSON.parseArray(value, String.class);
        return null;
    }

    @Override
    public List<String> getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        String value = rs.getString(columnIndex);
        if (value != null)
            return JSON.parseArray(value, String.class);
        return null;
    }

    @Override
    public List<String> getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        String value = cs.getString(columnIndex);
        if (value != null)
            return JSON.parseArray(value, String.class);
        return null;
    }
}
```


然后实体类这样定义

```java
// @TableName(value = "t_shop_goods_item", autoResultMap = true)
@Data
public class TShopGoodsItem implements Serializable {
    private static final long serialVersionUID = 1L;

    /**
     * id
     */
    @TableId(type = IdType.AUTO)

    private Integer id;

    /**
     * name 商品名称
     */
    private String name;

    /**
     * price
     */
    private BigDecimal price;

    /**
     * 图片, json数组格式 如: "[img1,img2]"
     */
    @TableField(typeHandler = com.baomidou.mybatisplus.extension.handlers.Fastjson2TypeHandler.class, value = "pics")
    private List<String> pics;
}
```


这里的pics字段就是储存的图片数组，这样设计没问题吧。但是调试的时候就遇到问题了，具体体现在新增没有问题，数据也入库了。但是查询的时候，pics 字段为 null ，查不出来数据？

查看`mybatis plus `官方文档， 还需要`实体类`添加 `autoResultMap=True`。 添加后重试，依然查询不出来。

晕了，这是什么问题？

这里首先怀疑就是`typehandler`没有注册上，于是进入代码断点。先新增，能成功断点。再查询，断点却没有命中。

那应该就是没有注册上了，搜索了下项目， 发现问题在：

```java
   @Bean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
       
        VFS.addImplClass(SpringBootVFS.class);
        // mybatis-plus sqlSession配置
        final MybatisSqlSessionFactoryBean sessionFactory = new MybatisSqlSessionFactoryBean();
        sessionFactory.setDataSource(dataSource);
        // 问题在这里
        // sessionFactory.setTypeHandlers(new ListStringHandler(), new BTypeHandler());
        return sessionFactory.getObject();
    }
```

这里注意的是：如果是自己手动生成的的 `SqlSessionFactory`， 那么需要自己手动注册handler。 如果没有自己生成，那么直接就可以使用。

所以破案了。 悲催的一下午。




好了，这里总结下如果mysql使用json类型，项目配置流程应该如下：

1. 先定义定义具体的`typehandler`转换器，这里根据类型定义（这是必须要有的）。
2. 如果`typehandler`定义了`@Component`那么能被自动扫描发现
3. 如果没有定义`@Component`,那么主动使用配置文件`  type-handlers-package:  com.xxxx.biz.utils.typehandler`
4. 如果项目中的`SqlSessionFactory`是手动创建的（那么自动扫描策略将会失效），需要像项目中一样手动注册

散会!~~~~~~~~~~~~




