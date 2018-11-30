---
title: Spring_Validation
date: 2018-10-16 14:24:09
tags: Spring
---

> 在后台校验参数是接收参数后的第一步，如果使用@RequestParam接收参数，逐个校验，Controller方法会过长 校验代码页非常冗余而且多个方法都需要校验。
>
> 而使用@Validated 进行校验，可以更为快捷的完成参数校验的工作，而且可以将参数校验放置到Bean钟，Ctrl代码简洁优雅，参数校验实现复用

<!--more-->

### JSR303/JSR349 HibernateValidation SpringValidation三者关系

* JSR303/JSR349是一项标准提供了一些**校验规范**，如@Null @NotNull @Pattern等，他们位于**javax.validation.constraints.\***;
* HibernateValidation 则是对于这个规范的一个**实践**(不同于ORM框架Hibernate),并提供了一些其他的校验注解，如@Email @Length @Range等，它们位于**org.hibernate.validator.constraints.\***
* SpringValidation则是**封装**了HibernateValidation,而且添加了自动校验，并将校验信息封装到特定的类中，它们位于**import org.springframework.validation.\***

## @Validated 使用

### Controller方法上使用

```java
@Controller
@Validated
public class TestController {
     @RequestMapping(value = "/test", method = RequestMethod.GET)
    public String testValidate(@RequestParam(value = "recordId", defaultValue = "")
                               @NotEmpty(message = "记录ID不可以为空")
                               @Pattern(regexp = "\\d{1,11}", message = "记录ID不符合规范") String recordId) {
        return "test";
    }

```



### Bean上使用

```java
public class UserSchoolGradeDTO {

    /**
     * 主键id
     */
    private Integer id;
    /**
     * 用户Id
     */
    @NotEmpty(message = "用户ID不可以为空")
    @Pattern(regexp = "\\d{1,11}", message = "用户ID不符合规范")
    private String userId;
    /**
     * 学科成绩
     */
    @Digits(integer = 3, fraction = 1)
    @Range(min = 0, max = 100)
    @NotNull
    private Float subjectScore;
}

// springMVC提供了自动封装表单参数的功能
// @Validated标识Spring对其进行校验 且校验信息会被存放到BingingResult中
// (@Validated A a ,BingingResult aBindingResult, @Validated B b, BingingResult bBingResult) 同时校验多个参数
public ResultBO insertUserSchoolGrade(@Validated UserSchoolGradeDTO userSchoolGradeDTO, BindingResult bindingResult) {

        // 若参数校验错误 则返回其中第一个错误信息
        if (bindingResult.hasErrors()) {
            return Results.fail(bindingResult.getFieldErrors().get(0).getDefaultMessage());
        }
    
    	// 遍历所有错误
        if(bindingResult.hasErrors()){
            for (FieldError fieldError : bindingResult.getFieldErrors()) {
                //...
            }
        }
      
}

```



### 常用校验注解

```java
JSR提供的校验注解：         
@Null   被注释的元素必须为 null    
@NotNull    被注释的元素必须不为 null    
@AssertTrue     被注释的元素必须为 true    
@AssertFalse    被注释的元素必须为 false    
@Min(value)     被注释的元素必须是一个数字，其值必须大于等于指定的最小值    
@Max(value)     被注释的元素必须是一个数字，其值必须小于等于指定的最大值    
@DecimalMin(value)  被注释的元素必须是一个数字，其值必须大于等于指定的最小值    
@DecimalMax(value)  被注释的元素必须是一个数字，其值必须小于等于指定的最大值    
@Size(max=, min=)   被注释的元素的大小必须在指定的范围内    
@Digits (integer, fraction)     被注释的元素必须是一个数字，其值必须在可接受的范围内    
@Past   被注释的元素必须是一个过去的日期    
@Future     被注释的元素必须是一个将来的日期    
@Pattern(regex=,flag=)  被注释的元素必须符合指定的正则表达式    


Hibernate Validator提供的校验注解：  
@NotBlank(message =)   验证字符串非null，且长度必须大于0    
@Email  被注释的元素必须是电子邮箱地址    
@Length(min=,max=)  被注释的字符串的大小必须在指定的范围内    
@NotEmpty   被注释的字符串的必须非空    
@Range(min=,max=,message=)  被注释的元素必须在合适的范围内
```



### 分组校验

当使用Bean校验时，同样一个Bean可能在不同的fuction接收校验的函数不同，通过使用分组校验可以实现这个功能，而不用编写多个Bean.

```java
Class Foo{

    @Min(value = 18,groups = {Adult.class})
    @Max(value = 18,groups = {Minor.class})
    private Integer age;

    public interface Adult{}

    public interface Minor{}
}

// 使用分组校验
@RequestMapping("/drink")
public String drink(@Validated({Foo.Adult.class}) Foo foo, BindingResult bindingResult) {
    if(bindingResult.hasErrors()){
        for (FieldError fieldError : bindingResult.getFieldErrors()) {
            //...
        }
        return "fail";
    }
    return "success";
}

// 不使用分组校验
@RequestMapping("/live")
public String live(@Validated Foo foo, BindingResult bindingResult) {
    if(bindingResult.hasErrors()){
        for (FieldError fieldError : bindingResult.getFieldErrors()) {
            //...
        }
        return "fail";
    }
    return "success";
}
```



### 自定义校验

**自定义校验注解**

```java
@Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER})
@Retention(RUNTIME)
@Documented
// 定义注解真正的实现类 
@Constraint(validatedBy = {CannotHaveBlankValidator.class})
public @interface CannotHaveBlank {

    //默认错误消息
    String message() default "不能包含空格";

    //分组
    Class<?>[] groups() default {};

    //负载
    Class<? extends Payload>[] payload() default {};

    //指定多个时使用
    @Target({FIELD, METHOD, PARAMETER, ANNOTATION_TYPE})
    @Retention(RUNTIME)
    @Documented
    @interface List {
        CannotHaveBlank[] value();
    }

}
```

**真正实现类**

```java
// 继承ConstraintValidator 初始化initialize 校验isValid方法
// 实现CannotHavaBlank方法 接收String
public class CannotHaveBlankValidator implements ConstraintValidator<CannotHaveBlank, String> {

    @Override
    public void initialize(CannotHaveBlank constraintAnnotation) {
    }

	// String value接收验证参数
    // ConstraintValidatorContext包含上下文中认证的的所有信息 可以获取错误提示信息 禁用错误提示信息 修改错误信息等操作
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        //null时不进行校验
        if (value != null && value.contains(" ")) {
            //获取默认提示信息
            String defaultConstraintMessageTemplate = context.getDefaultConstraintMessageTemplate();
            System.out.println("default message :" + defaultConstraintMessageTemplate);
            //禁用默认提示信息
            context.disableDefaultConstraintViolation();
            //设置提示语
            context.buildConstraintViolationWithTemplate("can not contains blank").addConstraintViolation();
            return false;
        }
        return true;
    }
}
```



### 手动校验

**Hibernate Validation校验**

```java
Foo foo = new Foo();
foo.setAge(22);
foo.setEmail("000");

// 获取ValidatorFactory 创建Validator实例
ValidatorFactory vf = Validation.buildDefaultValidatorFactory();
Validator validator = vf.getValidator();
// 手动校验类 获取校验结果实例Set
Set<ConstraintViolation<Foo>> set = validator.validate(foo);
for (ConstraintViolation<Foo> constraintViolation : set) {
    System.out.println(constraintViolation.getMessage());
}

```

**Spring Validation**

```java
// spring封装了Hibernate Validate实现 而且提供了自动化注入的方法
// 封装了LocalValidatorFactoryBean作为validator的实现，兼容了Spring和Hibernate的Validate体系
@Autowired
Validator globalValidator; <1>

@RequestMapping("/validate")
public String validate() {
    Foo foo = new Foo();
    foo.setAge(22);
    foo.setEmail("000");

    Set<ConstraintViolation<Foo>> set = globalValidator.validate(foo);<2>
    for (ConstraintViolation<Foo> constraintViolation : set) {
        System.out.println(constraintViolation.getMessage());
    }

    return "success";
}
```

### 

### 全局校验异常Catch

```java
 	@ExceptionHandler
    @ResponseBody
    public Map handle(MethodArgumentNotValidException exception) {
        return exception.getBindingResult().getFieldErrors()
                .stream()
                .map(FieldError::getDefaultMessage)
                .map(message->new HashMap<String,String>(){{
                    put("code", "2");
                    put("message", message);
                }})
                .collect(Collectors.toList()).get(0);
    }


    @ExceptionHandler
    @ResponseBody
    public Map handle(ConstraintViolationException exception) {
        return exception.getConstraintViolations()
                .stream()
                .map(ConstraintViolation::getMessage)
                .map(message->new HashMap<String,String>(){{
                    put("code", "2");
                    put("message", message);
                }})
                .collect(Collectors.toList()).get(0);
    }
```

