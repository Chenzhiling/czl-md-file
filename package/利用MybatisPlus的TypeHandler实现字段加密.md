# 利用`Mybatis`的`TypeHandler`实现字段加密

## 引言

记录使用`Mybatis`的`TypeHandler`实现对实体类字段的加密.

## 版本

- `Springboot` 2.5.3

- `MybatisPlus` 3.4.2
- `Mysql` 5.4.7

## `AES`工具类

编写一个工具类,实现对数据的加解密操作.

```java
public class AesUtils {

    public static final String AES_MIDDLE_KEY = "chenZhiLing";

    //aes的key
    private static Key AES_KEY;


    public static String encryptByAes(String content) {
        if(ObjectUtils.isEmpty(content)){
            return null;
        }
        try {
            // 加密
            Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
            cipher.init(Cipher.ENCRYPT_MODE, getKey());
            byte[] result = cipher.doFinal(content.getBytes());
            return org.apache.commons.codec.binary.Base64.encodeBase64String(result);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }



    public static String decryptByAes(String encryptedContent) {
        if (ObjectUtils.isEmpty(encryptedContent)){
            return null;
        }
        try {
            // 解密
            Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
            cipher.init(Cipher.DECRYPT_MODE, getKey());
            byte[] result = cipher.doFinal(org.apache.commons.codec.binary.Base64.decodeBase64(encryptedContent));
            return ObjectUtils.isEmpty(result)?null:new String(result);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

    /**
     * 获取AES对应的key
     * @return Key
     */
    public static Key getKey() {
        if (AES_KEY == null) {
            try {
                KeyGenerator keyGenerator = KeyGenerator.getInstance("AES");
                SecureRandom secureRandom = SecureRandom.getInstance("SHA1PRNG");
                secureRandom.setSeed(AES_MIDDLE_KEY.getBytes());
                keyGenerator.init(128,secureRandom);
                SecretKey secretKey = keyGenerator.generateKey();
                byte[] byteKey = secretKey.getEncoded();
                // 转换KEY
                AES_KEY = new SecretKeySpec(byteKey, "AES");
                return AES_KEY;
            } catch (Exception e) {
                return null;
            }
        } else {
            return AES_KEY;
        }
    }
}
```

## 自定义`BaseTypeHandler`实现

继承`BaseTypeHandler`,使用`AesUtils`实现即可

```java
@Service
@SuppressWarnings("unchecked")
public class CustomTableFieldHandler<T> extends BaseTypeHandler<T> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, AesUtils.encryptByAes((String) parameter));
    }

    @Override
    public T getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String columnValue = rs.getString(columnName);
        return ObjectUtils.isEmpty(columnValue) ? (T) columnValue : (T) AesUtils.decryptByAes(columnValue);

    }

    @Override
    public T getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        String columnValue = rs.getString(columnIndex);
        return ObjectUtils.isEmpty(columnValue) ? (T)columnValue : (T) AesUtils.decryptByAes(columnValue);

    }

    @Override
    public T getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        String columnValue = cs.getString(columnIndex);
        return ObjectUtils.isEmpty(columnValue) ? (T)columnValue : (T) AesUtils.decryptByAes(columnValue);
    }
}
```

## 使用

在需要加密的实体类字段上,添加注解即可.

```java
@TableField(value = "email",typeHandler = CustomTableFieldHandler.class)
@ApiModelProperty(value = "邮箱")
private String email;
```

## 注意

使用这种方式前提需要在`@TableName`注解上加上`autoResultMap = true`

```java
@TableName(value = "sys_user",autoResultMap = true)
```

