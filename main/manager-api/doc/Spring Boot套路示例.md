# Spring Boot 标准开发套路

## 1. Controller层套路 (相当于Android View)

```java
@RestController                    // 等同于Android的 @Activity
@RequestMapping("/api/user")       // 等同于Android的路由
@Tag(name = "用户管理")             // 等同于Android的页面描述
@AllArgsConstructor               // 等同于Android的 @Inject
public class UserController {
    
    private final UserService userService;  // 注入Service，类似Android注入ViewModel
    
    // 标准CRUD套路：
    
    // 1. 查询列表 (GET)
    @GetMapping("/list")
    @Operation(summary = "获取用户列表")
    public Result<PageData<UserDTO>> list(@RequestParam Map<String, Object> params) {
        // 1. 接收参数 (类似Android接收Intent参数)
        // 2. 调用Service (类似Android调用ViewModel方法)
        // 3. 返回结果 (类似Android更新UI)
        PageData<UserDTO> page = userService.page(params);
        return new Result<PageData<UserDTO>>().ok(page);
    }
    
    // 2. 查询详情 (GET)
    @GetMapping("/{id}")
    @Operation(summary = "获取用户详情")
    public Result<UserDTO> get(@PathVariable("id") Long id) {
        UserDTO user = userService.get(id);
        return new Result<UserDTO>().ok(user);
    }
    
    // 3. 新增 (POST)
    @PostMapping
    @Operation(summary = "新增用户")
    @LogOperation("新增用户")          // 记录操作日志
    public Result<Void> save(@RequestBody UserDTO dto) {
        // 参数校验 (类似Android的数据验证)
        ValidatorUtils.validateEntity(dto, AddGroup.class);
        userService.save(dto);
        return new Result<Void>();
    }
    
    // 4. 修改 (PUT)
    @PutMapping
    @Operation(summary = "修改用户")
    public Result<Void> update(@RequestBody UserDTO dto) {
        ValidatorUtils.validateEntity(dto, UpdateGroup.class);
        userService.update(dto);
        return new Result<Void>();
    }
    
    // 5. 删除 (DELETE)
    @DeleteMapping("/{id}")
    @Operation(summary = "删除用户")
    public Result<Void> delete(@PathVariable("id") Long id) {
        userService.delete(id);
        return new Result<Void>();
    }
}
```

## 2. Service层套路 (相当于Android ViewModel + UseCase)

```java
@Service                          // 等同于Android的 @ViewModel 
@Transactional(rollbackFor = Exception.class)  // 事务管理
public class UserServiceImpl extends BaseServiceImpl<UserDao, UserEntity> implements UserService {
    
    @Autowired
    private UserDao userDao;       // 注入DAO，类似Android注入Repository
    
    @Override
    public PageData<UserDTO> page(Map<String, Object> params) {
        // 1. 处理查询条件 (类似Android的数据转换)
        IPage<UserEntity> page = baseDao.selectPage(
            getPage(params, Constant.CREATE_DATE, false),
            getWrapper(params)
        );
        
        // 2. 实体转DTO (类似Android的Entity转UI Model)
        return getPageData(page, UserDTO.class);
    }
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void save(UserDTO dto) {
        // 1. 业务逻辑校验 (类似Android业务规则)
        checkBusiness(dto);
        
        // 2. DTO转Entity (类似Android数据转换)
        UserEntity entity = ConvertUtils.sourceToTarget(dto, UserEntity.class);
        
        // 3. 保存到数据库 (类似Android存储到Room)
        insert(entity);
        
        // 4. 缓存更新 (类似Android更新SharedPreferences)
        updateCache(entity);
    }
    
    private void checkBusiness(UserDTO dto) {
        // 业务逻辑校验 (类似Android的UseCase逻辑)
        if (userDao.selectCount(new QueryWrapper<UserEntity>()
            .eq("username", dto.getUsername())) > 0) {
            throw new RenException("用户名已存在");
        }
    }
}
```

## 3. DAO层套路 (相当于Android Repository)

```java
@Repository                       // 等同于Android的 @Repository
public interface UserDao extends BaseDao<UserEntity> {
    
    // 基础CRUD自动生成，类似Android Room的@Dao
    
    // 自定义查询方法
    List<UserDTO> getUserList(@Param("params") Map<String, Object> params);
    
    // 复杂查询
    @Select("SELECT * FROM sys_user WHERE status = #{status}")
    List<UserEntity> selectByStatus(@Param("status") Integer status);
}
```

```xml
<!-- UserDao.xml - 对应Android Room的SQL -->
<mapper namespace="com.example.dao.UserDao">
    
    <select id="getUserList" resultType="com.example.dto.UserDTO">
        SELECT 
            id,
            username,
            email,
            create_date
        FROM sys_user
        <where>
            <if test="params.username != null and params.username != ''">
                AND username LIKE CONCAT('%', #{params.username}, '%')
            </if>
            <if test="params.status != null">
                AND status = #{params.status}
            </if>
        </where>
        ORDER BY create_date DESC
    </select>
    
</mapper>
```

## 4. Entity/DTO/VO套路 (相当于Android数据类)

```java
// Entity - 数据库实体 (类似Android Room Entity)
@Data
@TableName("sys_user")            // 对应数据库表
public class UserEntity extends BaseEntity {
    @TableId(type = IdType.ASSIGN_ID)
    private Long id;
    
    @TableField("username")
    private String username;
    
    @TableField("email")
    private String email;
    
    @TableField("status")
    private Integer status;
}

// DTO - 数据传输对象 (类似Android Request/Response)
@Data
@Schema(description = "用户信息")
public class UserDTO {
    @Schema(description = "用户ID")
    private Long id;
    
    @NotBlank(message = "用户名不能为空", groups = {AddGroup.class, UpdateGroup.class})
    @Schema(description = "用户名")
    private String username;
    
    @Email(message = "邮箱格式不正确")
    @Schema(description = "邮箱")
    private String email;
    
    @Schema(description = "状态")
    private Integer status;
}

// VO - 视图对象 (类似Android UI Model)
@Data
@Schema(description = "用户显示信息")
public class UserVO {
    private Long id;
    private String username;
    private String email;
    private String statusText;      // 状态文本，如：启用/禁用
    private String createTime;      // 格式化后的创建时间
}
```

## 5. 固定套路总结

### Controller固定套路:
```java
1. @RestController + @RequestMapping    // 声明控制器和基础路径
2. @AllArgsConstructor                 // 构造器注入
3. 标准HTTP方法映射:
   - @GetMapping     (查询)
   - @PostMapping    (新增)
   - @PutMapping     (修改)
   - @DeleteMapping  (删除)
4. 参数校验: ValidatorUtils.validateEntity()
5. 统一返回: Result<T>
6. 异常处理: throw new RenException()
```

### Service固定套路:
```java
1. @Service + @Transactional           // 服务声明和事务管理
2. extends BaseServiceImpl             // 继承基础服务
3. 业务逻辑处理:
   - 参数校验
   - 数据转换 (DTO ↔ Entity)
   - 业务规则检查
   - 数据库操作
   - 缓存更新
```

### DAO固定套路:
```java
1. @Repository                         // 数据访问层声明
2. extends BaseDao<Entity>             // 继承基础DAO
3. 自定义查询方法
4. 配套的XML SQL文件
```

### 数据流转固定套路:
```
HTTP请求 → Controller (参数接收)
    ↓
Controller → Service (业务调用)
    ↓  
Service → DAO (数据查询)
    ↓
DAO → Database (SQL执行)
    ↓
Database → Entity (数据返回)
    ↓
Entity → DTO (数据转换)
    ↓
DTO → Result<DTO> (统一封装)
    ↓
Result → HTTP响应 (JSON返回)
```
