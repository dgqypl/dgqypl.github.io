---
layout: post
title:  "记一次抛乐观锁的异常"
author: mew151
image: assets/images/2.jpg
---
问题如下：

```Text
org.springframework.orm.ObjectOptimisticLockingFailureException: Object of class [com. ... .UserProfileItemDataDO] with identifier [1821]: optimistic locking failed; nested exception is org.hibernate.StaleObjectStateException: Row was updated or deleted by another transaction (or unsaved-value mapping was incorrect) : [com. ... .UserProfileItemDataDO#1821]
```

代码如下：
```Java
import org.springframework.transaction.support.TransactionTemplate;

public class UserDealWithServiceTest extends BaseTest {

    @Autowired
    private TransactionTemplate transactionTemplate;

    @Test
    public void testSimulateRollbackException() {
        transactionTemplate.execute(transactionStatus -> {
            UserProfileItemDataDO userProfileItemDataDO = userProfileItemDataDORepository.getOne(1821l);
            userProfileItemDataDO.setLastLockTime(new Date());
            userProfileItemDataDORepository.upsert(userProfileItemDataDO);
            return null;
        });
    }
```

#### 复现步骤
1、在下图中打断点：
![](/assets/images/202005/20200513blog1.png)
2、
debug程序，当程序定位在断点时，手工从数据库里将id=1821这条记录的version+1，再释放断点，就会报这个错了。