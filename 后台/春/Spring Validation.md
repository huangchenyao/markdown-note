# Spring Validation

[TOC]

## 简单示例
0. 依赖
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```
1. 在实体上加上校验注解
```Java
public class Item {

    @NotNull(message = "id不能为空")
    @Min(value = 1, message = "id必须为正整数")
    private Long id;

    @Valid // 嵌套验证必须用@Valid
    @NotNull(message = "props不能为空")
    @Size(min = 1, message = "props至少要有一个自定义属性")
    private List<Prop> props;
}
```
2. 在controller参数中加上@Valid或@Validated
```Java
@RestController
public class ItemController {

    @RequestMapping("/item/add")
    public void addItem(@Validated Item item, BindingResult bindingResult) {
        doSomething();
    }
}
```
3. 异常抛出与处理  
异常类型，`MethodArgumentNotValidException`，利用spring全局捕获异常并处理  

## 校验分类

### 自带基本校验
这里只列举了 javax.validation 包下的注解，同理在 spring-boot-starter-web 包中也存在 hibernate-validator 验证包，里面包含了一些 javax.validation 没有的注解  
| 注解                      | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| @NotNull                  | 限制必须不为null                                             |
| @NotEmpty                 | 验证注解的元素值不为 null 且不为空（字符串长度不为0、集合大小不为0） |
| @NotBlank                 | 验证注解的元素值不为空（不为null、去除首位空格后长度为0），不同于@NotEmpty，@NotBlank只应用于字符串且在比较时会去除字符串的空格 |
| @Pattern(value)           | 限制必须符合指定的正则表达式                                 |
| @Size(max,min)            | 限制字符长度必须在 min 到 max 之间（也可以用在集合上）       |
| @Email                    | 验证注解的元素值是Email，也可以通过正则表达式和flag指定自定义的email格式 |
| @Max(value)               | 限制必须为一个不大于指定值的数字                             |
| @Min(value)               | 限制必须为一个不小于指定值的数字                             |
| @DecimalMax(value)        | 限制必须为一个不大于指定值的数字                             |
| @DecimalMin(value)        | 限制必须为一个不小于指定值的数字                             |
| @Null                     | 限制只能为null（很少用）                                     |
| @AssertFalse              | 限制必须为false （很少用）                                   |
| @AssertTrue               | 限制必须为true （很少用）                                    |
| @Past                     | 限制必须是一个过去的日期                                     |
| @Future                   | 限制必须是一个将来的日期                                     |
| @Digits(integer,fraction) | 限制必须为一个小数，且整数部分的位数不能超过 integer，小数部分的位数不能超过 fraction （很少用） |

1. 对象验证demo
```Java
public class Book {

    private Integer id;
    
    @NotBlank(message = "name 不允许为空")
    @Length(min = 2, max = 10, message = "name 长度必须在 {min} - {max} 之间")
    private String name;
    
    @NotNull(message = "price 不允许为空")
    @DecimalMin(value = "0.1", message = "价格不能低于 {value}")
    private BigDecimal price;
    
}
```
2. 普通参数验证demo
```Java
@Validated
@RestController
public class ValidateController {

    @GetMapping("/test2")
    public String test2(@NotBlank(message = "name 不能为空") @Length(min = 2, max = 10, message = "name 长度必须在 {min} - {max} 之间") String name) {
        return "success";
    }

    @GetMapping("/test3")
    public String test3(@Validated Book book) {
        return "success";
    }
}
```

### 分组验证和分组顺序
新增修改时，想验证的内容不一样，需要使用分组验证器  
1. 定义分组接口，分组接口就是两个普通的接口，用于标识，类似于java.io.Serializable  
```Java
public interface First {  
}  
  
public interface Second {  
}  
```
2. 使用分组接口标识实体  
```Java
public class User implements Serializable {  
  
    @NotNull(message = "{user.id.null}", groups = {First.class})  
    private Long id;  
  
    @Length(min = 5, max = 20, message = "{user.name.length.illegal}", groups = {Second.class})  
    @Pattern(regexp = "[a-zA-Z]{5,20}", message = "{user.name.illegal}", groups = {Second.class})  
    private String name;  
  
    @NotNull(message = "{user.password.null}", groups = {First.class, Second.class})  
    private String password;  
}
```
3. 验证时使用分组接口  
```Java
@RequestMapping("/save")  
public String save(@Validated({Second.class}) User user, BindingResult result) {  
    if(result.hasErrors()) {  
        return "error";  
    }  
    return "success";  
}  
```
4. 使用多个分组  
user.name会显示两个错误消息，而且顺序不确定；如果我们先验证一个消息；如果不通过再验证另一个怎么办？可以通过@GroupSequence指定分组验证顺序，先验证First分组，如果有错误立即返回而不会验证Second分组，接着如果First分组验证通过了，那么才去验证Second分组，最后指定User.class表示那些没有分组的在最后。这样我们就可以实现按顺序验证分组了  
```Java
@GroupSequence({First.class, Second.class, User.class})  
public class User implements Serializable {  
    private Long id;  
  
    @Length(min = 5, max = 20, message = "{user.name.length.illegal}", groups = {First.class})  
    @Pattern(regexp = "[a-zA-Z]{5,20}", message = "{user.name.illegal}", groups = {Second.class})  
    private String name;  
      
    private String password;  
}  
```

### 消息中使用EL表达式
假设我们需要显示如：用户名[NAME]长度必须在[MIN]到[MAX]之间，我们不想把一些数据写死，如NAME、MIN、MAX；此时我们可以使用EL表达式  
例：  
```Java
@Length(min = 5, max = 20, message = "{user.name.length.illegal}", groups = {First.class}) 
```
错误消息：
```Yaml
user.name.length.illegal=用户名长度必须在{min}到{max}之间  
```
其中可以使用{验证注解的属性}得到这些值；如{min}得到@Length中的min值；其他的也是类似的。  
到此，我们还是无法得到出错的那个输入值，如name=zhangsan。此时就需要EL表达式的支持，首先确定引入EL jar包且版本正确。然后使用如：  
```Yaml
user.name.length.illegal=用户名[${validatedValue}]长度必须在5到20之间  
```
使用如EL表达式：${validatedValue}得到输入的值，如zhangsan。当然我们还可以使用如${min > 1 ? '大于1' : '小于等于1'}，及在EL表达式中也能拿到如@Length的min等数据。  
另外我们还可以拿到一个java.util.Formatter类型的formatter变量进行格式化：  
```Yaml
${formatter.format("%04d", min)}  
```

### 自定义验证器
1. 定义注解
```Java
@Target({ FIELD, METHOD, PARAMETER, ANNOTATION_TYPE })  
@Retention(RUNTIME)  
//指定验证器  
@Constraint(validatedBy = ForbiddenValidator.class)  
@Documented  
public @interface Forbidden {
  
    //默认错误消息  
    String message() default "{forbidden.word}";  
  
    //分组  
    Class<?>[] groups() default { };  
  
    //负载  
    Class<? extends Payload>[] payload() default { };  
  
    //指定多个时使用  
    @Target({ FIELD, METHOD, PARAMETER, ANNOTATION_TYPE })  
    @Retention(RUNTIME)  
    @Documented  
    @interface List {
        Forbidden[] value();  
    }  
}
```
2. 定义验证器，验证器中可以使用spring的依赖注入，如注入：@Autowired private ApplicationContext ctx; 
```Java
public class ForbiddenValidator implements ConstraintValidator<Forbidden, String> {  
  
    private String[] forbiddenWords = {"admin"};  
  
    @Override  
    public void initialize(Forbidden constraintAnnotation) {  
        //初始化，得到注解数据  
    }  
  
    @Override  
    public boolean isValid(String value, ConstraintValidatorContext context) {  
        if(StringUtils.isEmpty(value)) {  
            return true;  
        }  
  
        for(String word : forbiddenWords) {  
            if(value.contains(word)) {  
                return false;//验证失败  
            }  
        }  
        return true;  
    }  
}  
```
3. 使用
```Java
public class User implements Serializable {  
    @Forbidden()  
    private String name;  
}  
```

### 类级别验证器
1. 定义注解
```Java
@Target({ TYPE, ANNOTATION_TYPE})  
@Retention(RUNTIME)  
//指定验证器  
@Constraint(validatedBy = CheckPasswordValidator.class)  
@Documented  
public @interface CheckPassword {  
  
    //默认错误消息  
    String message() default "";  
  
    //分组  
    Class<?>[] groups() default { };  
  
    //负载  
    Class<? extends Payload>[] payload() default { };  
  
    //指定多个时使用  
    @Target({ FIELD, METHOD, PARAMETER, ANNOTATION_TYPE })  
    @Retention(RUNTIME)  
    @Documented  
    @interface List {  
        CheckPassword[] value();  
    }  
}  
```
2. 定义验证器
```Java
public class CheckPasswordValidator implements ConstraintValidator<CheckPassword, User> {  
  
    @Override  
    public void initialize(CheckPassword constraintAnnotation) {  
    }  
  
    @Override  
    public boolean isValid(User user, ConstraintValidatorContext context) {  
        if(user == null) {  
            return true;  
        }  
  
        //没有填密码  
        if(!StringUtils.hasText(user.getPassword())) {  
            context.disableDefaultConstraintViolation();  
            context.buildConstraintViolationWithTemplate("{password.null}")  
                    .addPropertyNode("password")  
                    .addConstraintViolation();  
            return false;  
        }  
  
        if(!StringUtils.hasText(user.getConfirmation())) {  
            context.disableDefaultConstraintViolation();  
            context.buildConstraintViolationWithTemplate("{password.confirmation.null}")  
                    .addPropertyNode("confirmation")  
                    .addConstraintViolation();  
            return false;  
        }  
  
        //两次密码不一样  
        if (!user.getPassword().trim().equals(user.getConfirmation().trim())) {  
            context.disableDefaultConstraintViolation();  
            context.buildConstraintViolationWithTemplate("{password.confirmation.error}")  
                    .addPropertyNode("confirmation")  
                    .addConstraintViolation();  
            return false;  
        }  
        return true;  
    }  
}  
```
3. 使用，放在类头上即可
```Java
@CheckPassword()  
public class User implements Serializable {  
}  
```

### 脚本验证