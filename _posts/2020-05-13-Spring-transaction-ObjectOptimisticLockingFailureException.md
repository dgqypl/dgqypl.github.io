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

---
参考资料：
[记一次事务的坑 Transaction rolled back because it has been marked as rollback-only](https://yunlongn.github.io/2019/05/06/%E8%AE%B0%E4%B8%80%E6%AC%A1%E4%BA%8B%E5%8A%A1%E7%9A%84%E5%9D%91Transaction-rolled-back-because-it-has-been-marked-as-rollback-only/)