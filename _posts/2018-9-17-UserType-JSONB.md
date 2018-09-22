---
layout: post
title: 自定义数据类型UserType映射到表中的JSONB
tags: [Hibernate, UserType, PostgreSQL, JSONB]
---

该篇主要讲解如何将自定义的数据类型映射到PostgreSQL表中的JSONB，结合上一篇的字段类型介绍，该篇算是实际应用。

因为我采用是hibernate框架，所以我要自定义数据类型就要实现`UserType`这个接口。

## UserType

```java
public interface UserType {

	/**
	 * 返回自定义数据类型所映射字段的SQL类型代码
	 */
	int[] sqlTypes();

	/**
	 * 返回 nullSafeGet() 自定义数据类型.
	 */
	Class returnedClass();

	/**
	 * 自定义数据类型比对
	 */
	boolean equals(Object x, Object y) throws HibernateException;

	/**
	 * 获取hashcode
	 */
	int hashCode(Object x) throws HibernateException;

	/**
	 * 从resultset从获取数据，转换成自定义数据类型后返回。
	 * 这里需要对空值进行处理。
	 */
	Object nullSafeGet(ResultSet rs, String[] names, SharedSessionContractImplementor session, Object owner) throws HibernateException, SQLException;

	/**
	 * 将自定义数据类型写入到prepared statement
	 * 这里也需要对空值进行处理。
	 */
	void nullSafeSet(PreparedStatement st, Object value, int index, SharedSessionContractImplementor session) throws HibernateException, SQLException;

	/**
	 * 自定义数据类型的深拷贝
	 * 当向用户返回数据之前，deepCopy被调用，拷贝的对象返回给用户，原始对象由hibernate维护。
	 * 当数据发生变化时（equals() 返回false），就会执行对应的持久化操作。
	 * 所以如果自定义数据类型时不可变的（isMutable() 返回false， 该方法就没必要实现了。
	 */
	Object deepCopy(Object value) throws HibernateException;

	/**
	 * 自定义数据类型是否可变
	 */
	boolean isMutable();

	/**
	 * Transform the object into its cacheable representation.
	 */
	Serializable disassemble(Object value) throws HibernateException;

	/**
	 * Reconstruct an object from the cacheable representation. 
	 */
	Object assemble(Serializable cached, Object owner) throws HibernateException;

	/**
	 *  During merge, replace the existing (target) value in the entity we are merging to with a new (original) value from the detached entity we are merging
	 */
	Object replace(Object original, Object target, Object owner) throws HibernateException;
}

```

## 具体实现

``` java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.hibernate.HibernateException;
import org.hibernate.engine.spi.SharedSessionContractImplementor;
import org.hibernate.type.SerializationException;
import org.hibernate.usertype.UserType;
import org.postgresql.util.PGobject;
import org.springframework.util.ObjectUtils;

import java.io.IOException;
import java.io.Serializable;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Types;
import java.util.HashMap;
import java.util.Map;

public class JsonType implements UserType {
	
    //这里用到的ObjectMapper是为了解析JSON
    private final ObjectMapper mapper = new ObjectMapper();

    @Override
    public int[] sqlTypes() {
        return new int[]{Types.JAVA_OBJECT};
    }

    @Override
    public Class<Map> returnedClass() {
        return Map.class;
    }

    @Override
    public boolean equals(Object x, Object y) throws HibernateException {
        return ObjectUtils.nullSafeEquals(x,y);
    }

    @Override
    public int hashCode(Object x) throws HibernateException {
        if (x == null) {
            return 0;
        }

        return x.hashCode();
    }

    @Override
    public Object nullSafeGet(ResultSet rs, String[] names, SharedSessionContractImplementor session, Object owner) throws HibernateException, SQLException {
        PGobject o = (PGobject) rs.getObject(names[0]);
        if (o.getValue() != null) {
            try {
                return mapper.readValue(o.getValue(), Map.class);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return new HashMap<>(16);
    }

    @Override
    public void nullSafeSet(PreparedStatement st, Object value, int index, SharedSessionContractImplementor session) throws HibernateException, SQLException {
        if (value == null) {
            st.setNull(index, Types.OTHER);
        } else {
            try {
                st.setObject(index, mapper.writeValueAsString(value), Types.OTHER);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    @Override
    public Object deepCopy(Object value) throws HibernateException {

        if (value != null) {
            try {
                return mapper.readValue(mapper.writeValueAsString(value),
                        returnedClass());
            } catch (IOException e) {
                throw new HibernateException("Failed to deep copy object", e);
            }
        }
        return null;
    }

    @Override
    public boolean isMutable() {
        return true;
    }

    @Override
    public Serializable disassemble(Object value) throws HibernateException {
        Object copy = deepCopy(value);

        if (copy instanceof Serializable) {
            return (Serializable) copy;
        }

        throw new SerializationException(String.format("Cannot serialize '%s', %s is not Serializable.", value, value.getClass()), null);
    }

    @Override
    public Object assemble(Serializable cached, Object owner) throws HibernateException {
        return deepCopy(cached);
    }

    @Override
    public Object replace(Object original, Object target, Object owner) throws HibernateException {
        return deepCopy(original);
    }
}


```

## 实体类

``` java
import org.hibernate.annotations.Type;
import org.hibernate.annotations.TypeDef;

import javax.persistence.*;
import java.util.Map;

@Entity
//自定义数据类型引入
@TypeDef(name = "JsonType", typeClass = JsonType.class)
public class JsonUser {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    //对应表中的JSONB数据类型
    @Column(columnDefinition = "jsonb")
  	//自定义类型（@TypeDef引入，@Type注明）
    @Type(type = "JsonType")
    private Map<String,Object> attributes;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Map<String, Object> getAttributes() {
        return attributes;
    }

    public void setAttributes(Map<String, Object> attributes) {
        this.attributes = attributes;
    }

    @Override
    public String toString() {
        return "JsonUser{" +
                "id=" + id +
                ", attributes=" + attributes +
                '}';
    }
}

```

## 测试

这里为了方便直接实现了JPA的一个接口, 做了一个保存操作和一个根据JSONB中的内容查询操作。

### Repository

``` java
public interface JsonUserDao extends CrudRepository<JsonUser, Long> {
    @Query(value = "select * from json_user where attributes #>> '{address,city}' = ?1",nativeQuery = true)
    JsonUser getUserByCity(String city);
}
```

### Test Method

``` java
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import java.util.HashMap;
import java.util.Map;

@RunWith(SpringRunner.class)
@SpringBootTest
public class JsonDemoApplicationTests {

    @Autowired
    private JsonUserDao userDao;

    @Test
    public void contextLoads() {
    }

    @Test
    public void testAddUser() {
        JsonUser user = new JsonUser();

        Map address = new HashMap<>(16);
        address.put("country", "china");
        address.put("city","xi'an");
        
        Map attributes = new HashMap<>(16);
        attributes.put("sex","man");
        attributes.put("address",address);

        user.setAttributes(attributes);
        userDao.save(user);

        System.out.println(userDao.getUserByCity("xi'an"));
    }

}

```

### Test Result

```
JsonUser{id=1, attributes={address={country=china, city=xi'an}, sex=man}}
```



## 结论

为了映射到表中JSONB数据类型，只需要俩步。

1. 自定义数据类型，我实现的是Map，当然你想实现哪种都可以；
2. 实体类中表明你想要映射的数据类型，以及你的自定义数据类型。
