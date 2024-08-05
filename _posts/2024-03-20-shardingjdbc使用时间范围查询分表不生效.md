---
layout: post
title:  "shardingjdbc使用时间范围查询分表不生效"
date:   2024-03-20 10:29:20 +0800
categories:
      - java
tags:
      - shardingjdbc
---

最近写一个小功能，公司有封装一套sharding包，引入即可直接使用。

于是按照文档引入，配置完成。使用了id和时间分表， 分别涉及到根据id查询和根据时间查询；

也就是sql长得像下面:
```sql
select * from test where id = 1;
select * from test where create_date < '2024-03-20 11:00:00' and create_date > '2024-03-19 11:00:00'

```
小功能，写不了多久就可以跑起来。但是跑起来后发现一个问题，TMD id能够正常分表， but： 时间不行。我擦，脑子嗡嗡的。

这里直接先说结果， 低版本的sharding只支持`in ,between, =` 这三种条件。我擦，升级到`4.1.1`即可支持'>,>=,<,<='。



下文为源码跟踪分析，断点找为啥，哈哈

首先断在分表策略那里，因为id可以正常分表，所以断住后直接看堆栈。

首先看到最可疑的一个类`StandardRoutingEngine`，里面有个这个方法:

时间sql执行时走到这里可以断住，但是走不到我们的分表策略，所以断下继续分析

下面代码中，idsql和时间sql `this.isRoutingByShardingConditions(tableRule)` 都返回true

所以跟进`this.routeByShardingConditions(tableRule) ` 看看
```java

    private Collection<DataNode> getDataNodes(TableRule tableRule) {
        if (this.shardingRule.isRoutingByHint(tableRule)) {
            return this.routeByHint(tableRule);
        } else {
            return this.isRoutingByShardingConditions(tableRule) ? this.routeByShardingConditions(tableRule) : this.routeByMixedConditions(tableRule);
        }
    }

```

跟进去代码如下, 这里两个sql都执行到这里，但是 ` this.optimizedStatement.getShardingConditions().getConditions().isEmpty()` 这个代码

时间查询返回false,  id查询返回true

大约可以看到, 这个条件是 `optimizedStatement` 初始化的时候完成的，所以继续往上翻，找到 optimizedStatement 的初始化代码。

```java
    private Collection<DataNode> routeByShardingConditions(TableRule tableRule) {
        return this.optimizedStatement.getShardingConditions().getConditions().isEmpty() ?
          this.route(tableRule, Collections.emptyList(), Collections.emptyList()) :
          this.routeByShardingConditionsWithCondition(tableRule);
    }
```

往上查看堆栈信息可以找到`ShardingSelectOptimizeEngine`， 这个类是负责初始化 `optimizedStatement`的。

初始化代码:`shardingConditionEngine.createShardingConditions(sqlStatement, parameters);`



```java
public final class ShardingSelectOptimizeEngine implements ShardingOptimizeEngine<SelectStatement> {
    public ShardingSelectOptimizeEngine() {
    }

    public ShardingSelectOptimizedStatement optimize(ShardingRule shardingRule, ShardingTableMetaData shardingTableMetaData, String sql, List<Object> parameters, SelectStatement sqlStatement) {
        WhereClauseShardingConditionEngine shardingConditionEngine = new WhereClauseShardingConditionEngine(shardingRule, shardingTableMetaData);
        WhereClauseEncryptConditionEngine encryptConditionEngine = new WhereClauseEncryptConditionEngine(shardingRule.getEncryptRule(), shardingTableMetaData);
        GroupByEngine groupByEngine = new GroupByEngine();
        OrderByEngine orderByEngine = new OrderByEngine();
        SelectItemsEngine selectItemsEngine = new SelectItemsEngine(shardingTableMetaData);
        PaginationEngine paginationEngine = new PaginationEngine();
        List<ShardingCondition> shardingConditions = shardingConditionEngine.createShardingConditions(sqlStatement, parameters);
        List<EncryptCondition> encryptConditions = encryptConditionEngine.createEncryptConditions(sqlStatement);
        GroupBy groupBy = groupByEngine.createGroupBy(sqlStatement);
        OrderBy orderBy = orderByEngine.createOrderBy(sqlStatement, groupBy);
        SelectItems selectItems = selectItemsEngine.createSelectItems(sql, sqlStatement, groupBy, orderBy);
        Pagination pagination = paginationEngine.createPagination(sqlStatement, selectItems, parameters);
        ShardingSelectOptimizedStatement result = new ShardingSelectOptimizedStatement(sqlStatement, shardingConditions, encryptConditions, groupBy, orderBy, selectItems, pagination);
        this.setContainsSubquery(sqlStatement, result);
        return result;
    }
```

找到这个方法，这里 `this.createShardingConditions` 负责创建分表条件，跟进去。
```java
public final class WhereClauseShardingConditionEngine {
    private final ShardingRule shardingRule;
    private final ShardingTableMetaData shardingTableMetaData;

    public List<ShardingCondition> createShardingConditions(SQLStatement sqlStatement, List<Object> parameters) {
        if (!(sqlStatement instanceof WhereSegmentAvailable)) {
            return Collections.emptyList();
        } else {
            List<ShardingCondition> result = new ArrayList();
            Optional<WhereSegment> whereSegment = ((WhereSegmentAvailable)sqlStatement).getWhere();
            Tables tables = new Tables(sqlStatement);
            if (whereSegment.isPresent()) {
                result.addAll(this.createShardingConditions(tables, ((WhereSegment)whereSegment.get()).getAndPredicates(), parameters));
            }
```

跟进来看这里有一个方法 `this.createRouteValueMap(tables, each, parameters);` 负责创建where条件, 继续跟。
```java


    private Collection<ShardingCondition> createShardingConditions(Tables tables, Collection<AndPredicate> andPredicates, List<Object> parameters) {
        Collection<ShardingCondition> result = new LinkedList();
        Iterator var5 = andPredicates.iterator();

        while(var5.hasNext()) {
            AndPredicate each = (AndPredicate)var5.next();
            Map<Column, Collection<RouteValue>> routeValueMap = this.createRouteValueMap(tables, each, parameters);
            if (routeValueMap.isEmpty()) {
                return Collections.emptyList();
            }

            result.add(this.createShardingCondition(routeValueMap));
        }

        return result;
    }
```

这里有个静态方法`ConditionValueGeneratorFactory.generate(each.getRightValue(), column, parameters);`
负责创建分表条件对象，如过创建失败了，则不参与分表。ok 继续跟。
```java
    private Map<Column, Collection<RouteValue>> createRouteValueMap(Tables tables, AndPredicate andPredicate, List<Object> parameters) {
        Map<Column, Collection<RouteValue>> result = new HashMap();
        Iterator var5 = andPredicate.getPredicates().iterator();

        while(var5.hasNext()) {
            PredicateSegment each = (PredicateSegment)var5.next();
            Optional<String> tableName = tables.findTableName(each.getColumn(), this.shardingTableMetaData);
            if (tableName.isPresent() && this.shardingRule.isShardingColumn(each.getColumn().getName(), (String)tableName.get())) {
                Column column = new Column(each.getColumn().getName(), (String)tableName.get());
                Optional<RouteValue> routeValue = ConditionValueGeneratorFactory.generate(each.getRightValue(), column, parameters);
                if (routeValue.isPresent()) {
                    if (!result.containsKey(column)) {
                        result.put(column, new LinkedList());
                    }

                    ((Collection)result.get(column)).add(routeValue.get());
                }
            }
        }

        return result;
    }
```

上面的工厂静态方法很简单，只有3行代码，断点继续跟进去，会跟到如下类：
```java


public final class ConditionValueCompareOperatorGenerator implements ConditionValueGenerator<PredicateCompareRightValue> {
    public ConditionValueCompareOperatorGenerator() {
    }

    public Optional<RouteValue> generate(PredicateCompareRightValue predicateRightValue, Column column, List<Object> parameters) {
        if (!this.isSupportedOperator(predicateRightValue.getOperator())) {
            return Optional.absent();
        } else {
            Optional<Comparable> routeValue = (new ConditionValue(predicateRightValue.getExpression(), parameters)).getValue();
            return routeValue.isPresent() ? Optional.of(new ListRouteValue(column.getName(), column.getTableName(), Lists.newArrayList(new Comparable[]{(Comparable)routeValue.get()}))) : Optional.absent();
        }
    }

    private boolean isSupportedOperator(String operator) {
        return "=".equals(operator);
    }
}

```

这里可以看到核心业务方法`this.isSupportedOperator(predicateRightValue.getOperator())`, 如果不支持就返回空。 

看到这里就明白了，特么居然不支持。 我记得之前项目都行的啊，赶紧看下版本，原来他们偷偷升级了。我TMD。

接着升级到 `4.1.1`，重启项目，一切正常。 我是真的服了，浪费个把小时。
