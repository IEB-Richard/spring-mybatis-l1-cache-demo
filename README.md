# spring-mybatis-l1-cache-demo

This project is developed based on project `spring-mybatis-xml`. 



Updates in test case:

```java
package tk.mybatis.simple.mapper;

import org.apache.ibatis.session.SqlSession;
import org.junit.Assert;
import org.junit.Test;

import tk.mybatis.simple.model.SysUser;

public class CacheTest extends BaseMapperTest {
	
	@Test
	public void testL1Cache() {
		// get sqlSession
		SqlSession sqlSession = getSqlSession();
		SysUser user1 = null;
		try {
			// get mapper object
			UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
			
			// get user1
			user1 = userMapper.selectById(1l);
			// assgin a new value 
			user1.setUserName("New Name");
			System.out.println(user1.toString());
			
			// search the same user and assign to another user
			SysUser user2 = userMapper.selectById(1l);
			// though it haven't updated the DB, user2 have the new userName we assigned
			// with user1.
			Assert.assertEquals("New Name", user2.getUserName());
			// user1 and user2 should be the same object
			Assert.assertEquals(user1, user2);

		} finally {
			sqlSession.close();
		}
		
		// open another new sqlSession
		System.out.println("Open another new SqlSession");
		sqlSession = getSqlSession();
		
		try {
			UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
			// L1 cache only exist in the same sqlSession,
			SysUser user2 = userMapper.selectById(1l);
			// so user2 will not the same object as in the first sqlSession
			Assert.assertNotEquals("New Name", user2.getUserName());
			Assert.assertNotEquals(user1, user2);
			// any insert, delete or update action will flush L1 cache 
			userMapper.deleteById(2L);
			// get the same user again and assigned to user3
			SysUser user3 = userMapper.selectById(1l);
			// user3 and user2 will be not the same object as cache is flushed.
			Assert.assertNotEquals(user2, user3);
		} finally {
			sqlSession.close();
		}
	}
}

```



If you dont' make any changes in xml mapper file, user1 and user2 will be the same object, and it will only access database once.

```
DEBUG [main] - ==>  Preparing: select * from sys_user where id = ? 
DEBUG [main] - ==> Parameters: 1(Long)
TRACE [main] - <==    Columns: id, user_name, user_password, user_email, user_info, head_img, create_time
TRACE [main] - <==        Row: 1, admin, 123456, admin@mybatis.tk, <<BLOB>>, <<BLOB>>, 2016-06-07 01:11:12
DEBUG [main] - <==      Total: 1
SysUser [id=1, userName=New Name, userPassword=123456, userEmail=admin@mybatis.tk, userInfo=管理员用户, headImg=[18, 49, 35, 18, 48], createTime=Tue Jun 07 14:11:12 CST 2016, role=null, roleList=null]
Open another new SqlSession
```



mybatis get user2 from level 1 cache, as it called the same method with same parameter. If you want to avoid this action, make the following changes to XML mapper method:

Before update:

```xml
	<select id="selectById" resultMap="userMap">
		select * from sys_user where id = #{id}
	</select>
```



After update:

```xml
	<select id="selectById" flushCache="true" resultMap="userMap">
		select * from sys_user where id = #{id}
	</select>
```



In the second sqlSession user2 will not be equal to user1, because Level 1 cache only exists in the same sql session.

in the second sql session, user2 and user3 are not the same, because any insert, delete or update action will flush mybatis level 1 cache.