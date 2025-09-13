
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
```java
@Insert("insert into table_name (num,grade,name,age,sex)"
            + "values(#{num},#{grade},#{name},#{age},#{sex}")
@Options(useGeneratedKeys = true, keyProperty = "studentId", keyColumn = "student_id")
void insert(Student student);
```
*此处涉及 Opions 注解，后续用到的时候在详解*