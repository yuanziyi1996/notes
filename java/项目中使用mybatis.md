# 项目中使用Mybatis

## 使用背景

在开发问卷功能的时候需要搭建一个新的服务，原来所有的调用数据库使用的都是 Jdbctemplate 模板。换了一个服务，由于想多接触开发技术栈的原因，questionnaire 中使用mybatis.

## 环境的搭建和配置：

1. 创建一个以 mybatis 为基础组件的springboot 项目。

2. 然后引入mybatis 的最新 pom 包

```java
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>
```

![数据库层文件结构](http://file.gsxservice.com/umeng_tutor/ddf7d574520549ba4134c3095253965e.png)
3. 在yml 文件中配置数据库配置和mybatis设置

```java
## appllo配置
apollo:
  cacheDir: ../conf
  bootstrap:
    namespaces: tech-yingxiao.mysql-tutor, tech-yingxiao.redis-tutor, application
    enabled: true

#mybatis配置
mybatis:
  configuration:
    cache-enabled: true
    lazy-loading-enabled: true
    aggressive-lazy-loading: true
    multiple-result-sets-enabled: true
    use-column-label: true
    use-generated-keys: true
    auto-mapping-behavior: full
    map-underscore-to-camel-case: true
```

> map-underscore-to-camel-case ： true 一定要配置，不然查询结果不能识别驼峰式命名

4. 搭建好本项目的 数据库层的文件目录结构，如下

![数据库层文件结构](http://file.gsxservice.com/umeng_tutor/9df5cf38f605a85cf1b382d7872bd303.png)

5. 完成以上，下面在启动类里加上注解用于给出需要扫描的mapper文件路径 @MapperScan(basePackages = {"com.baijia.questionnaire.dal.dao.owner"})

```java
@MapperScan(basePackages = {"com.baijia.questionnaire.dal.dao.owner"})
@EnableScheduling
@EnableApolloConfig
@EnableDiscoveryClient
public class QuestionnaireApplication {
    public static void main(String[] args) {
        SpringApplication.run(QuestionnaireApplication.class, args);
    }
}
```
基本框架搭建完成

## 定制模板化的insert or update 方法，让使用者不再过多关心sql语句。

### 基类：TableHelper

* 主要是包括驼峰式字段和连字符格式字段的相互转化
* 获取一个PO类的所有字段
* 获取一个PO类对应的表名字
* 预编译Mybatis支持的 sql语句。（支持批量）

```java
@Slf4j
public class InsertSqlProvider extends TableHelper {
    public <T> String insertNoNull(final T t) {
        try {
            String tableName = getPoTableName(t);
            Field[] fields = t.getClass().getDeclaredFields();
            SQL sql = new SQL();
            sql.INSERT_INTO(tableName);
            for (Field field : fields) {
                field.setAccessible(true);
                Object value = field.get(t);
                if (Objects.nonNull(value)) {
                    sql.VALUES(getLowerCase(field.getName()), preparedRightStatement(field.getName()));
                }
            }
            return sql.toString();
        } catch (Exception e) {
            e.printStackTrace();
        }

        return "";
    }


    public <T> String insertOrUpdate(T t) {
        String insertSql = insertNoNull(t);
        String onDuplicateKeySql = buildOnDuplicateKeySql(t);
        return insertSql + onDuplicateKeySql;
    }

    public <T> String batchInsert(final Map<String, List<T>> map) {
        try {
            //取第一个key
            String key = map.keySet().iterator().next();
            List<T> lists = map.get(key);
            if (CollectionUtils.isEmpty(lists)) {
                return "";
            }
            String tableName = getPoTableName(lists.get(0));
            T t = lists.get(0);
            Field[] fields = t.getClass().getDeclaredFields();
            SQL sql = new SQL();
            sql.INSERT_INTO(tableName);
            List<String> columns = new ArrayList<>();
            for (Field field : fields) {
                field.setAccessible(true);
                columns.add(getLowerCase(field.getName()));
            }
            sql.INTO_COLUMNS(columns.toArray(new String[columns.size()]));

            StringBuilder sb = new StringBuilder();
            for (int i = 0, len = lists.size(); i < len; i++) {
                if (i > 0) {
                    sb.append(") , (");
                }
                for (Field field : fields) {
                    field.setAccessible(true);
                    sb.append(prepareBatchStatement(field.getName(), i));
                }
                sb.deleteCharAt(sb.length() - 1);
            }
            sql.INTO_VALUES(sb.toString());
            return sql.toString();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "";
    }

    public <T> String batchInsertOrUpdateNoNull(final Map<String, List<T>> map) {
        String insertSql = batchInsert(map);
        String key = map.keySet().iterator().next();
        if (CollectionUtils.isEmpty(map.get(key))) {
            return insertSql;
        }
        T t = map.get(key).get(0);
        String onDuplicateKeySql = buildBatchOnDuplicateKeySql(t);
//        log.info("batchinsertorupdate sql={}", insertSql + onDuplicateKeySql);
        return insertSql + onDuplicateKeySql;
    }

    private <T> String buildOnDuplicateKeySql(T t) {
        try {
            List<String> updateArr = new ArrayList<>();
            Field[] declaredFields = t.getClass().getDeclaredFields();
            AtomicInteger count = new AtomicInteger(0);
            for (Field field : declaredFields) {
                String fieldName = field.getName();
                if (field.isAnnotationPresent(SkipField.class)
                        || Objects.equals(fieldName, "id")) {
                    continue;
                }
                field.setAccessible(true);
                Object value = field.get(t);
                if (Objects.nonNull(value)) {
                    updateArr.add(preparedStatement(fieldName));
                    count.addAndGet(1);
                }
            }
            if (count.get() == 0) {
                throw new DaoSupportException("all field value is null.");
            }
            return " ON DUPLICATE KEY UPDATE " + String.join(",", updateArr);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "";
    }

    private <T> String buildBatchOnDuplicateKeySql(T t) {
        try {
            StringBuilder stringBuilder = new StringBuilder();
            Field[] declaredFields = t.getClass().getDeclaredFields();
            for (Field field : declaredFields) {
                String fieldName = field.getName();
                if (field.isAnnotationPresent(SkipField.class)
                        || Objects.equals(fieldName, "id")) {
                    continue;
                }
                field.setAccessible(true);
                Object value = field.get(t);
                stringBuilder.append(onDuplicateValue(field.getName()));
            }
            return " ON DUPLICATE KEY UPDATE " + stringBuilder.deleteCharAt(stringBuilder.length() - 1).toString();
        } catch (Exception e) {
            e.printStackTrace();
            return "";
        }
    }
}
```

## 模板化的insert

调用者只需要在使用时：加上 InsertProvider 注解，type 处写insert语句所在的类.class，method就是对应的insert方法。 如果需要返回插入时候的主键，加上@Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "id")
```java
@InsertProvider(type = InsertSqlProvider.class, method = "insertNoNull")
    @Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "id")
    boolean insert(QuestionnairePO questionnairePO);
```
insert 语句原理：解析将要插入的PO类中不为 null的字段，并且构建一个mybatis能够识别的语句。
```java
String tableName = getPoTableName(t);
            Field[] fields = t.getClass().getDeclaredFields();
            SQL sql = new SQL();
            sql.INSERT_INTO(tableName);
            for (Field field : fields) {
                field.setAccessible(true);
                Object value = field.get(t);
                if (Objects.nonNull(value)) {
                    sql.VALUES(getLowerCase(field.getName()), preparedRightStatement(field.getName()));
                }
            }
            return sql.toString();
```

## 模板化的update

同样调用者只需要在使用时：加上 @UpdateProvider 注解，type 处写insert语句所在的类.class，method就是对应的insert方法。
```java
 @UpdateProvider(type = UpdateSqlProvider.class, method = "updateNoNull")
    boolean updateReplierInfoPoById(ReplierInfoPO replierInfoPO);
```

update 语句原理：解析将要插入的PO类中不为 null的字段，并且构建一个mybatis能够识别的语句。

```java
String tableName = getPoTableName(t);
            SQL sql = new SQL();
            Field[] fields = t.getClass().getDeclaredFields();
            sql.UPDATE(tableName);
            String id = null;
            for (Field field : fields) {
                String name = field.getName();
                field.setAccessible(true);
                Object value = field.get(t);
                if (Objects.equals("id", name)) {
                    id = field.getName();
                    continue;
                }
                if (Objects.equals("createTime", name) || Objects.equals("updateTime", name)) {
                    continue;
                }
                if (Objects.nonNull(value)) {
                    sql.SET(preparedStatement(field.getName()));
                }
            }
            sql.WHERE(preparedStatement(id));
            return sql.toString();
```

## 项目地址
http://git.baijiahulian.com/pandora/questionnaire