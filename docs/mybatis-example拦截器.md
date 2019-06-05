### 针对Mybatis,generator自动生成的example类型DAO接口拦截器
对查询字段包装，查询结果处理；常用使用场景如字段加解密

### SecurityFacadeInterceptor拦截器
DaoModel - 数据库实体
DaoModelExample - Mybatis自动生成的Example类
```
/**
 * 拦截Mybatis generate example的Dao接口
 */
@Intercepts({
        @Signature(type=Executor.class,method="update",args={MappedStatement.class,Object.class}),
        @Signature(type= Executor.class,method="query",args={MappedStatement.class,Object.class, RowBounds.class, ResultHandler.class}),
        @Signature(type= Executor.class,method="query",args={MappedStatement.class,Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class})
})
@Slf4j
public class SecurityFacadeInterceptor implements Interceptor {

    /** 加密开关：默认关闭 */
    private String encryptSwitch = "off";


    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object parameter = invocation.getArgs()[1];
        String method = invocation.getMethod().getName();
        log.info("method is {}", method);
        // 加密开关开启
        if("on".equals(encryptSwitch)){
            // 查询，参数加密为密文查询
            if("query".equalsIgnoreCase(method)) {
                // selectByExample(),countByExample(),
                if(parameter.getClass().getSimpleName().contains("Example")){
                    encryptExample(parameter);
                }
                // insert(),insertSelective()执行完成后会调用查询方法，
                // DaoModel，returnValue为List<Integer>的ID集合，此类型忽略处理
                if(parameter instanceof DaoModel){

                }
            }
            // 更新，参数加密为密文更新
            if("update".equalsIgnoreCase(method)){
                // insert(), insertSelective(), updateByPrimaryKey(), updateByPrimaryKeySelective()
                if(parameter instanceof DaoModel){
                    encryptRecord(parameter);
                }
                // deleteByExample()
                if(parameter.getClass().getSimpleName().contains("Example")){
                    encryptExample(parameter);
                }

                // updateByExample(), updateByExampleSelective()
                if(parameter instanceof MapperMethod.ParamMap){
                    MapperMethod.ParamMap paramMap = (MapperMethod.ParamMap) parameter;
                    Object record = paramMap.get("record");
                    if(record instanceof DaoModel) {
                        encryptRecord(record);
                    }
                    Object example = paramMap.get("example");
                    if(parameter.getClass().getSimpleName().contains("Example")){
                        encryptExample(example);
                    }

                    // 重新设置值
                    paramMap.put("record", record);
                    paramMap.put("param1", record);
                    paramMap.put("example", example);
                    paramMap.put("param2", example);
                }

            }
        }
        // 返回结果解密
        Object returnValue = invocation.proceed();
        if(returnValue instanceof ArrayList<?>){
            List<?> list = (List<?>) returnValue;
            if(CollectionUtils.isEmpty(list)){
                return list;
            }
            // 判断返回结果是否有待解密字段
            Object obj = list.get(0);
            if(obj instanceof InsOrder){
                Field[] fields = obj.getClass().getDeclaredFields();
                boolean isDecrypt = false;
                for(Field field : fields){
                    if(SecurityUtil.encryptFields.contains(field.getName())){
                        isDecrypt = true;
                        break;
                    }
                }
                // 需要解密
                if(isDecrypt){
                    list.forEach(t -> SecurityUtil.encryptionDecryptionField(t, fields, false));
                }
            }
        }
        return returnValue;
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
        String encryptSwitch = properties.getProperty("encryptSwitch");
        if(StringUtils.isNotBlank(encryptSwitch)){
            this.encryptSwitch = encryptSwitch;
        }
    }

    /**
     * 加密Record记录
     * @param parameter
     */
    private void encryptRecord(Object parameter) {
        Field[] fields = parameter.getClass().getDeclaredFields();
        boolean isEncrypt = false;
        for (Field field : fields) {
            if (SecurityUtil.encryptFields.contains(field.getName())) {
                isEncrypt = true;
                break;
            }
        }
        // 需要加密
        if (isEncrypt) {
            SecurityUtil.encryptionDecryptionField(parameter, fields, true);
        }
    }

    /**
     * 加密example参数
     * @param parameter
     * @throws NoSuchFieldException
     * @throws IllegalAccessException
     */
    private void encryptExample(Object parameter) throws NoSuchFieldException, IllegalAccessException {
        if(parameter instanceof  DaoModelExample){
            DaoModelExample example = (DaoModelExample) parameter;
            List<DaoModelExample.Criteria> oredCriteria = example.getOredCriteria();
            for (DaoModelExample.Criteria criteria : oredCriteria) {
                List<DaoModelExample.Criterion> criterionList = criteria.getCriteria();
                for (DaoModelExample.Criterion criterion : criterionList) {
                    String condition = criterion.getCondition();
                    String colName = condition.substring(0, condition.lastIndexOf(" ="));
                    if (SecurityUtil.encryptTableCol.contains(colName)) {
                        String value = (String) criterion.getValue();
                        String encryptValue = SecurityUtil.encryptField(value);
                        Field field = criterion.getClass().getDeclaredField("value");
                        field.setAccessible(true);
                        field.set(criterion, encryptValue);
                    }
                }
            }
        }
    }
}

```

#### 工具类 SecurityUtil
```
public class SecurityUtil {

    /** 需要加解密的Model字段 */
    public static List<String> encryptFields = new ArrayList<>();
    /** 需要加解密的Table Column字段 */
    public static List<String> encryptTableCol = new ArrayList<>();

    private static EncryptService encryptService;

    static {
        encryptFields.add("fieldName1");
        encryptFields.add("fieldName2");
        encryptFields.add("fieldName3");

        encryptTableCol.add("field_name1");
        encryptTableCol.add("field_name2");
        encryptTableCol.add("field_name3");

        encryptService = ApplicationContext.getBean("encryptService");
    }

    /**
     * 单个字段加密
     * @param oldValue
     * @return
     */
    public static String encryptField(String oldValue){
        String rs = oldValue;
        try {
            rs = encryptService.encrypt(oldValue);
        }catch (Exception e){
            throw new RuntimeException(e);
        }
        return rs;
    }

    /**
     * 对象字段属性加解密
     * @param obj
     * @param fields
     * @param isEncrypt
     */
    public static void encryptionDecryptionField(Object obj, Field[] fields, boolean isEncrypt){
        try {
            List<String> plainList = new ArrayList<>();
            Map<String, Integer> fieldIndex = new HashMap<>();
            if (fields != null && fields.length > 0) {
                for (Field field : fields) {
                    if(encryptFields.contains(field.getName()) && field.getType().toString().endsWith("String")){
                        if(!field.isAccessible()){
                            field.setAccessible(true);
                        }
                        String oldValue = (String)field.get(obj);
                        if(oldValue != null){
                            plainList.add(oldValue);
                            fieldIndex.put(field.getName(), plainList.indexOf(oldValue));
                        }
                    }
                }
            }
            List<String> rs = new ArrayList;
            if(isEncrypt){
                // 调用解密服务批量解密
                rs = encryptService.encrypt();
            }else{
                // 调用解密服务批量解密
                rs = encryptService.decrypt();
            }
            if (fields != null && fields.length > 0) {
                for (Field field : fields) {
                    if(encryptFields.contains(field.getName()) && field.getType().toString().endsWith("String")){
                        if(!field.isAccessible()){
                            field.setAccessible(true);
                        }
                        Integer index = fieldIndex.get(field.getName());
                        if(index != null){
                            field.set(obj, rs.get(index));
                        }
                    }
                }
            }

        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

}

```
