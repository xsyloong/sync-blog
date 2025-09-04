
# Tips

## 1. 插入数据如何将数据 id 返回

- 使用 useGeneratedKeys
 ```xml
  <insert id="insertUser" parameterType="com.example.User" useGeneratedKeys="true" keyProperty="id">
    
    INSERT INTO users (name, age) 
  
    VALUES (#{name}, #{age})
  
  </insert>
  ```

- 如果使用注解则使用