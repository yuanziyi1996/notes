# 静态工具类中使用注解注入service

一般需要在一个工具类中使用@Autowired 注解注入一个service。但是由于工具类方法一般都写成static，所以直接注入就存在问题。

使用如下方式可以解决：
```java
/**
 * @author ziyi.yuan
 * @date 2020/8/3 8:03 下午
 */
@Component
public class StudentTypeUtil {
    @Autowired
    private RedisTemplateClient redisTemplateClient;

    @Autowired
    private GaotuApiBoost gaotuApiBoost;

    private static StudentTypeUtil studentTypeUtil;

    private int ONE_HOUR_CACHE_TIME = 3600;

    @PostConstruct
    public void init() {
        studentTypeUtil = this;
        studentTypeUtil.redisTemplateClient = this.redisTemplateClient;
        studentTypeUtil.gaotuApiBoost = this.gaotuApiBoost;
    }

    public static int getStudentType(Integer tutorId, Long userId) {
        String key = RedisConstant.getGaoStudentType(tutorId, userId);
        String result = studentTypeUtil.redisTemplateClient.get(key);
        if (Objects.nonNull(result) && StringUtils.isNotBlank(result)) {
            return StudentCourseTypeEnum.getCodeByDesc(result);
        }

        //todo 获取当前学生状态,并且写入redis
        int student = 1;

        return student;

    }
}
```
