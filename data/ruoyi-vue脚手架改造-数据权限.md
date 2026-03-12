# 若依集成 MyBatis-Plus 数据权限改造记录

## 一、 依赖管理

在 `pom.xml` 中添加 MyBatis-Plus 及相关依赖。**特别注意**：引入 `pagehelper` 时需排除 `mybatis` 相关依赖，防止与 MP 冲突；引入 `jsqlparser` 需排除旧版本，使用 MP 依赖的版本。

```
 <!-- MyBatis-Plus 核心依赖 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>${mybatis-plus.version}</version>
</dependency>

<!-- SQL 解析器（MP 数据权限依赖） -->
<dependency>
    <groupId>com.github.jsqlparser</groupId>
    <artifactId>jsqlparser</artifactId>
    <version>4.6</version>
</dependency>

<!-- PageHelper 分页插件（需排除冲突依赖） -->
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>${pagehelper.boot.version}</version>
    <exclusions>
        <exclusion>
            <artifactId>mybatis-spring</artifactId>
            <groupId>org.mybatis</groupId>
        </exclusion>
        <exclusion>
            <artifactId>mybatis</artifactId>
            <groupId>org.mybatis</groupId>
        </exclusion>
        <exclusion>
            <artifactId>jsqlparser</artifactId>
            <groupId>com.github.jsqlparser</groupId>
        </exclusion>
    </exclusions>
</dependency>
```

## 二、 核心配置

配置 MP 拦截器链，主要包含数据权限、分页、乐观锁等功能。

```
@EnableTransactionManagement(proxyTargetClass = true)
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 数据权限插件
        interceptor.addInnerInterceptor(dataPermissionInterceptor());
        // 分页插件
        interceptor.addInnerInterceptor(paginationInnerInterceptor());
        // 乐观锁插件
        interceptor.addInnerInterceptor(optimisticLockerInnerInterceptor());
        // 阻断插件
        interceptor.addInnerInterceptor(blockAttackInnerInterceptor());
        return interceptor;
    }

    /**
     * 数据权限插件
     */
    public DataPermissionInterceptor dataPermissionInterceptor()
    {
        return new DataPermissionInterceptor(new MpDataPermissionHandler());
    }

    /**
     * 分页插件，自动识别数据库类型 https://baomidou.com/guide/interceptor-pagination.html
     */
    public PaginationInnerInterceptor paginationInnerInterceptor() {
        PaginationInnerInterceptor paginationInnerInterceptor = new PaginationInnerInterceptor();
        // 设置数据库类型为mysql
        paginationInnerInterceptor.setDbType(DbType.MYSQL);
        // 设置最大单页限制数量，默认 500 条，-1 不受限制
        paginationInnerInterceptor.setMaxLimit(-1L);
        return paginationInnerInterceptor;
    }

    /**
     * 乐观锁插件 https://baomidou.com/guide/interceptor-optimistic-locker.html
     */
    public OptimisticLockerInnerInterceptor optimisticLockerInnerInterceptor() {
        return new OptimisticLockerInnerInterceptor();
    }

    /**
     * 如果是对全表的删除或更新操作，就会终止该操作 https://baomidou.com/guide/interceptor-block-attack.html
     */
    public BlockAttackInnerInterceptor blockAttackInnerInterceptor() {
        return new BlockAttackInnerInterceptor();
    }
}
```

## 三、 数据权限逻辑实现

### 1. 上下文上下文持有者

用于在 ThreadLocal 中存储当前请求的数据权限配置。

```
public class DataScopeContextHolder
{
    private static final ThreadLocal<DataScope> DATA_SCOPE_THREAD_LOCAL = new ThreadLocal<>();

    public static void setDataScope(DataScope dataScope)
    {
        DATA_SCOPE_THREAD_LOCAL.set(dataScope);
    }

    public static DataScope getDataScope()
    {
        return DATA_SCOPE_THREAD_LOCAL.get();
    }

    public static void clear()
    {
        DATA_SCOPE_THREAD_LOCAL.remove();
    }
}
```

### 2. 切面拦截

在 Controller 层拦截 `@DataScope` 注解，将配置存入 ThreadLocal，并在方法结束后清理。

```
@Aspect
@Component
public class DataScopeAspect
{
    @Before("@annotation(controllerDataScope)")
    public void doBefore(JoinPoint point, DataScope controllerDataScope) throws Throwable
    {
        clearDataScope();
        handleDataScope(controllerDataScope);
    }
    /**
     * 后置通知：清理 ThreadLocal
     * 必须加这个，确保方法结束后数据被清理，防止内存泄漏和后续请求污染
     */
    @After("@annotation(controllerDataScope)")
    public void doAfter(JoinPoint point, DataScope controllerDataScope) {
        clearDataScope();
    }


    protected void handleDataScope(DataScope controllerDataScope)
    {
        LoginUser loginUser = SecurityUtils.getLoginUser();
        if (StringUtils.isNotNull(loginUser))
        {
            SysUser currentUser = loginUser.getUser();
            if (StringUtils.isNotNull(currentUser) && !currentUser.isAdmin())
            {
                DataScopeContextHolder.setDataScope(controllerDataScope);
            }
        }
    }

    private void clearDataScope()
    {
        DataScopeContextHolder.clear();
    }
}
```

### 3. 权限 SQL 生成器

核心逻辑：获取用户角色，根据不同的数据范围生成对应的 SQL 过滤条件。添加MybatisPlus拦截插件。

```
@Slf4j
public class MpDataPermissionHandler implements DataPermissionHandler {
    public static final String DATA_SCOPE_ALL = "1";
    public static final String DATA_SCOPE_CUSTOM = "2";
    public static final String DATA_SCOPE_DEPT = "3";
    public static final String DATA_SCOPE_DEPT_AND_CHILD = "4";
    public static final String DATA_SCOPE_SELF = "5";

    @Override
    public Expression getSqlSegment(Expression where, String mappedStatementId) {
        DataScope dataScope = DataScopeContextHolder.getDataScope();
        if (dataScope == null)
        {
            // 这里应该返回原 where，而不是 null。返回 null 会覆盖原 where 条件
            return where;
        }

        LoginUser loginUser = SecurityUtils.getLoginUser();
        if (loginUser == null)
        {
            return null;
        }

        SysUser currentUser = loginUser.getUser();
        if (currentUser == null || currentUser.isAdmin())
        {
            return where;
        }

        String deptAlias = dataScope.deptAlias();
        String userAlias = dataScope.userAlias();
        String permission = StringUtils.defaultIfEmpty(dataScope.permission(), "");
        try
        {
            List<String> conditions = new ArrayList<>();
            // 用于收集自定义权限的角色ID
            List<Long> customRoleIds = new ArrayList<>();
            // 用于去重，防止同一种权限类型拼接多次
            Set<String> handledScopeTypes = new HashSet<>();

            List<SysRole> roles = currentUser.getRoles();
            if (roles == null || roles.isEmpty())
            {
                // 没有角色，无权查看任何数据
                return buildFalseCondition();
            }

            // 3. 遍历角色，生成条件
            for (SysRole role : roles)
            {
                // 跳过禁用的角色
                if (StringUtils.equals(role.getStatus(), UserConstants.ROLE_DISABLE))
                {
                    continue;
                }

                // 权限字符校验（如果注解配置了permission，则角色必须拥有该权限）
                if (StringUtils.isNotEmpty(permission)
                        && !StringUtils.containsAny(role.getPermissions(), Convert.toStrArray(permission)))
                {
                    continue;
                }

                String dataScopeValue = role.getDataScope();

                // 遇到 ALL 权限，直接放行，无需拼接其他条件
                if (DATA_SCOPE_ALL.equals(dataScopeValue))
                {
                    return where;
                }

                // 处理自定义权限：先收集ID，最后统一处理
                if (DATA_SCOPE_CUSTOM.equals(dataScopeValue))
                {
                    customRoleIds.add(role.getRoleId());
                }
                // 其他权限类型，去重处理
                else if (!handledScopeTypes.contains(dataScopeValue))
                {
                    String segment = buildScopeSegment(dataScopeValue, deptAlias, userAlias, currentUser);
                    if (StringUtils.isNotEmpty(segment))
                    {
                        conditions.add(segment);
                        handledScopeTypes.add(dataScopeValue);
                    }
                    else {
                        log.warn("数据权限生成失败，可能缺少必要的别名配置: scope={}, deptAlias={}, userAlias={}", dataScopeValue, deptAlias, userAlias);
                    }
                }
            }

            // 4. 处理自定义权限（合并为一个 IN 条件）
            if (!customRoleIds.isEmpty())
            {
                String ids = customRoleIds.stream()
                        .map(String::valueOf)
                        .collect(Collectors.joining(","));
                String deptPrefix = StringUtils.isNotEmpty(deptAlias) ? deptAlias + "." : "";
                // 优化：无论几个角色，都使用 IN，SQL更规整
                String customSql = String.format(
                        " %sdept_id IN ( SELECT dept_id FROM sys_role_dept WHERE role_id IN (%s) )",
                        deptPrefix, ids
                );
                conditions.add(customSql);
            }

            // 5. 根据结果生成最终 SQL
            if (conditions.isEmpty())
            {
                // 没有任何权限条件，默认无权查看
                return buildFalseCondition();
            }

            // 使用括号包裹所有条件，并用 OR 连接
            // 结果类似: ( (cond1) OR (cond2) OR (cond3) )
            String whereSegment = "(" + String.join(" OR ", conditions) + ")";
            Expression dataPermissionExpression = CCJSqlParserUtil.parseCondExpression(whereSegment);
            if (where == null)
            {
                return dataPermissionExpression;
            } else {
                return new AndExpression(where, new Parenthesis(dataPermissionExpression));
            }
        }
        catch (Exception e)
        {
            log.error("数据权限过滤异常: {}", e.getMessage(), e);
            // 【重要安全修复】发生异常时，抛出异常或返回永假条件，防止权限绕过
            // 这里选择返回永假条件，保证安全且不中断流程（具体策略视业务需求而定）
            try {
                return buildFalseCondition();
            } catch (JSQLParserException ex) {
                throw new RuntimeException(ex);
            }
        }
    }

    /**
     * 构建具体权限范围的 SQL 片段
     */
    private String buildScopeSegment(String dataScopeValue, String deptAlias, String userAlias, SysUser currentUser)
    {
        // 计算前缀：有别名则拼接点号，无别名则为空字符串
        String deptPrefix = StringUtils.isNotEmpty(deptAlias) ? deptAlias + "." : "";
        String userPrefix = StringUtils.isNotEmpty(userAlias) ? userAlias + "." : "";
        System.out.println(deptPrefix + " " + userPrefix);

        if (DATA_SCOPE_DEPT.equals(dataScopeValue)) {
            // SQL: dept_id = 100 或 d.dept_id = 100
            return String.format(" %sdept_id = %d ", deptPrefix, currentUser.getDeptId());
        }
        else if (DATA_SCOPE_DEPT_AND_CHILD.equals(dataScopeValue)) {
            // SQL: dept_id IN (...) 或 d.dept_id IN (...)
            return String.format(
                    " %sdept_id IN ( SELECT dept_id FROM sys_dept WHERE dept_id = %d OR find_in_set(%d, ancestors) )",
                    deptPrefix, currentUser.getDeptId(), currentUser.getDeptId()
            );
        }
        else if (DATA_SCOPE_SELF.equals(dataScopeValue)) {
            // 仅本人权限
            // 优先使用 userAlias，如果没有则尝试使用 deptAlias (兼容单表可能只有dept字段的情况)
            // 如果都为空，为了安全返回 null，外层会触发 buildFalseCondition 或警告
            if (StringUtils.isNotEmpty(userAlias)) {
                return String.format(" %suser_id = %d ", userPrefix, currentUser.getUserId());
            } else {
                // 如果 userAlias 为空，通常配置有误，但也可能存在单表通过 dept_id 等于0来拦截的逻辑
                // 这里建议返回 null，由外层判断。或者你可以根据业务需求降级处理。
                return null;
            }
        }
        return null;
    }

    /**
     * 构建一个永假条件 (1=0)
     */
    private Expression buildFalseCondition() throws JSQLParserException {
        return CCJSqlParserUtil.parseCondExpression(" 1 = 0 ");
    }
}
```

## 四、 清理遗留代码

**重要步骤**：全局搜索并删除 Mapper XML 文件中的 `${params.dataScope}` 占位符，防止新旧权限逻辑冲突。

- 主要涉及文件：`SysDeptMapper.xml`, `SysRoleMapper.xml`, `SysUserMapper.xml` 等。

## 五、 后续待办规划

由于时间紧迫，以下内容暂未实施，后续需统一规划处理。

### 1. 分页功能统一化

目前项目中 PageHelper 与 MyBatis-Plus 分页插件共存，存在拦截器顺序冲突的风险。

- **现状**：保留了 PageHelper，配置了 MP 拦截器顺序以兼容当前用法。
  
  ```
  @EnableTransactionManagement(proxyTargetClass = true)
  @Configuration
  public class MybatisPlusConfig {
  
      @Autowired
      private DataSource dataSource;
  
      @Value("${mybatis-plus.config-location:classpath:mybatis/mybatis-config.xml}")
      private String configLocation;
  
      /**
       * 核心配置：手动创建 SqlSessionFactory，精确控制插件顺序
       */
      @Bean
      public SqlSessionFactory sqlSessionFactory() throws Exception {
          // 1. 创建 MP 的工厂 Bean
          MybatisSqlSessionFactoryBean factoryBean = new MybatisSqlSessionFactoryBean();
          factoryBean.setDataSource(dataSource);
  
          // 2. 设置 XML 扫描路径 (若依默认路径)
          factoryBean.setMapperLocations(new PathMatchingResourcePatternResolver()
                  .getResources("classpath*:mapper/**/*Mapper.xml"));
  
          // 3. 设置实体类别名包路径 (若依默认路径)
          factoryBean.setTypeAliasesPackage("com.flyhub.**.domain");
          factoryBean.setConfigLocation(new DefaultResourceLoader().getResource(configLocation));
  
          // 4. 设置全局配置 (可选，根据若依习惯)
          GlobalConfig globalConfig = new GlobalConfig();
          globalConfig.setBanner(false); // 关闭控制台 Logo
          factoryBean.setGlobalConfig(globalConfig);
  
          // 5. 关键点：按顺序定义拦截器数组
          Interceptor[] plugins = new Interceptor[]{
                  // 【第一位：PageHelper 拦截器】
                  // 让它排在数组前面，实际代理时会在内层，执行时后执行
                  pageInterceptor(),
  
                  // 【第二位：MP 拦截器】
                  // 让它排在数组后面，实际代理时会在外层，执行时先执行 (先处理权限)
                  mybatisPlusInterceptor()
          };
  
          factoryBean.setPlugins(plugins);
  
          return factoryBean.getObject();
      }
      @Bean
      public MybatisPlusInterceptor mybatisPlusInterceptor() {
          MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
          // 数据权限插件
          interceptor.addInnerInterceptor(dataPermissionInterceptor());
          // 分页插件
          interceptor.addInnerInterceptor(paginationInnerInterceptor());
          // 乐观锁插件
          interceptor.addInnerInterceptor(optimisticLockerInnerInterceptor());
          // 阻断插件
          interceptor.addInnerInterceptor(blockAttackInnerInterceptor());
          return interceptor;
      }
  
      public PageInterceptor pageInterceptor() {
          PageInterceptor pageInterceptor = new PageInterceptor();
          Properties properties = new Properties();
          // 这里的配置参考若依原有的 application.yml 中的 pagehelper 配置
          properties.setProperty("helperDialect", "mysql");
          properties.setProperty("reasonable", "true");
          properties.setProperty("supportMethodsArguments", "true");
          properties.setProperty("params", "count=countSql");
          pageInterceptor.setProperties(properties);
          return pageInterceptor;
      }
      /**
       * 数据权限插件
       */
      public DataPermissionInterceptor dataPermissionInterceptor()
      {
          return new DataPermissionInterceptor(new MpDataPermissionHandler());
      }
      /**
       * 分页插件，自动识别数据库类型 https://baomidou.com/guide/interceptor-pagination.html
       */
      public PaginationInnerInterceptor paginationInnerInterceptor() {
          PaginationInnerInterceptor paginationInnerInterceptor = new PaginationInnerInterceptor();
          // 设置数据库类型为mysql
          paginationInnerInterceptor.setDbType(DbType.MYSQL);
          // 设置最大单页限制数量，默认 500 条，-1 不受限制
          paginationInnerInterceptor.setMaxLimit(-1L);
          return paginationInnerInterceptor;
      }
  
      /**
       * 乐观锁插件 https://baomidou.com/guide/interceptor-optimistic-locker.html
       */
      public OptimisticLockerInnerInterceptor optimisticLockerInnerInterceptor() {
          return new OptimisticLockerInnerInterceptor();
      }
  
      /**
       * 如果是对全表的删除或更新操作，就会终止该操作 https://baomidou.com/guide/interceptor-block-attack.html
       */
      public BlockAttackInnerInterceptor blockAttackInnerInterceptor() {
          return new BlockAttackInnerInterceptor();
      }
  }
  ```


- **待办**：建议后续统一标准。
- **方案 A**：全面切换至 MyBatis-Plus 的 `Page` 分页，移除 PageHelper 依赖。
- **方案 B**：保留 PageHelper，移除 MP 分页插件，仅使用 MP 的 ORM 功能。

### 2. 实体类映射规范化

MP 默认要求实体类字段与数据库一一对应，若依原有实体类中包含大量非数据库字段（如 `BaseEntity` 中的 `searchValue`、`remark`，`SysDept` 中的 `children`）。

- **现状**：暂时通过 XML 中的 `resultMap` 兼容，未做实体类大改动。
- **待办**：统一实体类规范。
- 在 `BaseEntity` 等父类字段上添加 `@TableField(exist = false)` 注解，明确告知 MP 忽略映射。
- 或者调整 MP 配置，使其严格匹配，但这需要清理实体类中的非表字段。

