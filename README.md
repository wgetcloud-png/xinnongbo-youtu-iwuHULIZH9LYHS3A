
如果多个实体类都有 `isDelete` 字段，并且你希望在插入时为它们统一设置默认值，可以采取以下几种方法来减少代码重复：


### 1\. 使用基类（抽象类）


创建一个基类，其中包含 `isDelete` 字段和 `@PrePersist` 方法。然后让所有需要这个字段的实体类继承这个基类。


#### 示例代码：



```
import javax.persistence.MappedSuperclass;
import javax.persistence.PrePersist;

@MappedSuperclass
public abstract class BaseEntity {

    protected Integer isDelete;

    @PrePersist
    public void prePersist() {
        if (isDelete == null) {
            isDelete = 0; // 设置默认值为0
        }
    }

    // Getter 和 Setter
    public Integer getIsDelete() {
        return isDelete;
    }

    public void setIsDelete(Integer isDelete) {
        this.isDelete = isDelete;
    }
}

```

然后在其他实体类中继承 `BaseEntity`：



```
import javax.persistence.Entity;
import javax.persistence.Id;

@Entity
public class MyEntity extends BaseEntity {

    @Id
    private Long id;

    // 其他字段、getter 和 setter
}

```

### 2\. 使用 AOP（面向切面编程）


通过 Spring AOP 创建一个切面，在插入操作时检查并设置 `isDelete` 的默认值。这种方式不需要修改每个实体类，适合大规模应用。


#### 示例代码：



```
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.springframework.stereotype.Component;

import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import java.lang.reflect.Field;

@Aspect
@Component
public class DefaultValueAspect {

    @PersistenceContext
    private EntityManager entityManager;

    @Before("execution(* com.example.repository.*.save(..))") // 根据你的仓库路径调整
    public void setDefaultValues(Object entity) throws IllegalAccessException {
        Field[] fields = entity.getClass().getDeclaredFields();
        for (Field field : fields) {
            if ("isDelete".equals(field.getName())) { // 检查字段名
                field.setAccessible(true);
                if (field.get(entity) == null) {
                    field.set(entity, 0); // 设置默认值为0
                }
            }
        }
    }
}

```

### 3\. 使用 JPA 审计功能


使用 Spring Data JPA 的审计功能，通过实现 `AuditorAware` 接口来统一处理审计字段，包括 createdBy,createdTime,updatedBy,updatedTime等，而`isDelete`这种状态字段在审计注释中并没有实现。


### 4\. 使用事件监听@EntityListeners


JPA 提供了事件监听器的功能，你可以定义一个事件监听器来处理所有需要设置默认值的实体类。


#### 示例代码：



```
import javax.persistence.PostLoad;
import javax.persistence.PrePersist;
import javax.persistence.EntityListeners;

public interface DeletedField {

  	Integer getDeletedFlag();

	void setDeletedFlag(Integer deletedFlag);
}

public class DeleteDefaultValueListener {

	@PrePersist
	public void setDefaultValues(DeletedFlagField deletedFlagField) {
		if (deletedFlagField.getDeletedFlag() == null) {
			deletedFlagField.setDeletedFlag(0); // 设置默认值为0
		}
	}

}

@EntityListeners(DefaultValueListener.class)
@Entity
public class TableUserAccount extends EntityBase implements DeletedFlagField {

  	/**
	 * 删除标识（逻辑删除），1删除 0未删除
	 */
	@Column(name = "deleted_flag")
	private Integer deletedFlag;
}

```

### 5\. 扩展JPA，对审计字段建立者和更新者的扩展


* CreatedByField 建立者字段接口
* UpdatedByField 更新者字段接口
* CreatedByDefaultValueListener 建立者字段监听器
* UpdatedByDefaultValueListener 更新者字段监听器
* AuditorAwareImpl 审计接口，返回当前用户


#### CreatedByField



```
public interface CreatedByField {

	String getCreatedBy();

	void setCreatedBy(String createdBy);

}

```

#### 扩展EntityBase实体，不使用默认的`CreatedBy`和`LastModifiedBy`



```
@Getter
@Setter
@MappedSuperclass
@EntityListeners({ AuditingEntityListener.class, UpdatedByDefaultValueListener.class,
		CreatedByDefaultValueListener.class })
public abstract class EntityBase implements Serializable, CreatedByField, UpdatedByField {

	/**
	 * 创建人
	 */
	@Column(name = "created_by")
	private String createdBy;

	/**
	 * 修改人
	 */
	@Column(name = "updated_by")
	private String updatedBy;
	}

```

### CreatedByDefaultValueListener



```
public class CreatedByDefaultValueListener implements ApplicationContextAware {

	private ApplicationContext applicationContext;

	@PrePersist
	public void setDefaultValues(CreatedByField createdByField) {
		if (createdByField.getCreatedBy() == null) {
			if (this.applicationContext.getBean(AuditorAwareImpl.class) != null) {
				createdByField.setCreatedBy(
						this.applicationContext.getBean(AuditorAwareImpl.class).getCurrentAuditor().orElse(""));

			}
		}
	}

	/**
	 * @param applicationContext
	 * @throws BeansException
	 */
	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		this.applicationContext = applicationContext;
	}

}

```

 本博客参考[veee加速器](https://youhaochi.com)。转载请注明出处！
