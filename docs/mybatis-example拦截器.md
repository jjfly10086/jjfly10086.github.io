### 针对Mybatis,generator自动生成的example类型DAO接口拦截器
对查询字段包装，查询结果处理；常用使用场景如字段加解密

### SecurityFacadeInterceptor拦截器
```
package com.creativearts.common.msg.insurance.interceptor;

import com.creativearts.common.msg.insurance.dao.model.InsOrder;
import com.creativearts.common.msg.insurance.dao.model.InsOrderExample;
import com.creativearts.common.msg.insurance.util.SecurityUtil;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.collections.CollectionUtils;
import org.apache.commons.lang.StringUtils;
import org.apache.ibatis.binding.MapperMethod;
import org.apache.ibatis.cache.CacheKey;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;

import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.List;
import java.util.Properties;

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
                // 参数为InsOrder，returnValue为List<Integer>的ID集合，此类型忽略处理
                if(parameter instanceof InsOrder){

                }
            }
            // 更新，参数加密为密文更新
            if("update".equalsIgnoreCase(method)){
                // insert(), insertSelective(), updateByPrimaryKey(), updateByPrimaryKeySelective()
                if(parameter instanceof InsOrder){
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
                    if(record instanceof InsOrder) {
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
        if(parameter instanceof  InsOrderExample){
            InsOrderExample example = (InsOrderExample) parameter;
            List<InsOrderExample.Criteria> oredCriteria = example.getOredCriteria();
            for (InsOrderExample.Criteria criteria : oredCriteria) {
                List<InsOrderExample.Criterion> criterionList = criteria.getCriteria();
                for (InsOrderExample.Criterion criterion : criterionList) {
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
package com.creativearts.common.msg.insurance.util;

import com.creativearts.common.msg.insurance.channel.InsuranceChannelFactory;
import com.creativearts.common.msg.insurance.service.IDubboService;
import com.creativearts.common.security.common.SecType;
import com.creativearts.common.security.dto.SecurityRequestBatchDto;
import com.creativearts.common.security.dto.SecurityRequestDto;
import com.creativearts.common.security.facade.SecurityFacade;
import com.creativearts.fx.agent.utils.IPUtils;
import org.springframework.util.Assert;

import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class SecurityUtil {

    /** 需要加解密的Model字段 */
    public static List<String> encryptFields = new ArrayList<>();
    /** 需要加解密的Table Column字段 */
    public static List<String> encryptTableCol = new ArrayList<>();

    private static SecurityFacade securityFacade;

    static {
        encryptFields.add("applicantName");
        encryptFields.add("applicantCardNo");
        encryptFields.add("insuredName");
        encryptFields.add("insuredCardNo");
        encryptFields.add("insuredMobile");
        encryptFields.add("insuredBankCardNo");

        encryptTableCol.add("applicant_name");
        encryptTableCol.add("applicant_card_no");
        encryptTableCol.add("insured_name");
        encryptTableCol.add("insured_card_no");
        encryptTableCol.add("insured_mobile");
        encryptTableCol.add("insured_bank_card_no");

        IDubboService dubboService = InsuranceChannelFactory.applicationContext.getBean("dubboService", IDubboService.class);
        securityFacade = dubboService.getSecurityFacade();
        Assert.notNull(securityFacade, "dubbo服务securityFacade为空");

    }

    /**
     * 单个字段加密
     * @param oldValue
     * @return
     */
    public static String encryptField(String oldValue){
        String rs = oldValue;
        try {
            // 调用加密服务加密
            SecurityRequestDto dto = new SecurityRequestDto();
            dto.setFixKeyIndex("zy_business_key");
            dto.setSecType(SecType.AES);
            dto.setSecData(oldValue);
            dto.setSysIp(IPUtils.getLocalIP());
            dto.setSysName("nyd-sys");
            rs = securityFacade.encrypt(dto);
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
            SecurityRequestBatchDto dto = new SecurityRequestBatchDto();
            dto.setSecType(SecType.AES);
            dto.setFixKeyIndex("zy_business_key");
            dto.setSecData(plainList);
            dto.setSysIp(IPUtils.getLocalIP());
            dto.setSysName("nyd-sys");
            List<String> rs;
            if(isEncrypt){
                // 调用解密服务批量解密
                rs = securityFacade.encrypt(dto);
            }else{
                // 调用解密服务批量解密
                rs = securityFacade.decrypt(dto);
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
