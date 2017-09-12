---
layout: post
title: java注解
keywords: java annotation 
desc: java注解
photoUrl:
---

# java注解

## 介绍
java 1.5引入了注解(annotation),注解类似注释，不同的是注解除了提供代码说明，
还能实现程序的逻辑功能，在很多java框架中都得到了广发的应用。

## 元注解
元注解的作用就是负责注解其他注解。Java5.0定义了4个标准的meta-annotation类型，它们被用来提供对其它 annotation类型作说明。
* @Target 修饰了注解使用范围 可以是类(TYPE),方法(METHOD),字段（FIELD）,构造函数（CONSTRUCTOR），参数（PARAMETER），局部变量（LOCAL_VARIABLE），包（PACKAGE）
* @Retention 修饰了注解存在的时间 SOURCE 只在源码中出现，CLASS 只在Class文件可以看见 RUNTIME 运行时也可以看见
* @Documented 用来修饰注解是否出现在API中
* @Inherited 是否会被子类继承 注解正常只出现在父类 子类不继承 加上这个标记 子类也能获得注解

## 示例代码
给类付上额外的信息
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ClassAnno {
    String value() default "类注解";
}

@ClassAnno("Test类")
public class Test {

}

public class Main {

    public static void main(String ...args){
        ClassAnno classAnno=Test.class.getAnnotation(ClassAnno.class);
        if(classAnno!=null){
            System.out.println(classAnno.value());
        }
    }
}
```
测试方法
* 去掉Test类上的注解 查看结果
* 去掉Test类上注解的值 查看结果

## 注解应用 验证类
一个验证接口
```java
/**
 * 验证接口
 * @param <T> 值类型
 * @param <A> 注解类型
 */
public interface Validator<T,A> {

    /**
     * 验证方法
     * @param value 值
     * @param anno  注解
     * @return  错误信息 如果是null表示无错误
     */
    String validate(T value,A anno);
}
```

实现一个注解和验证接口实现类的绑定
```java
/**
 * 绑定注解和验证类
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidatorBind {
    /**
     * 验证类
     * @return
     */
    Class validator();
}
```
实现一个接受验证结果的类
```java
/**
 * 验证结果
 */
public class ValidatorResult {
    /**
     * 错误信息
     */
    private String message;
    /**
     * 字段名称
     */
    private String fieldName;
    /**
     * 字段路径
     */
    private String fieldPath;

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public String getFieldName() {
        return fieldName;
    }

    public void setFieldName(String fieldName) {
        this.fieldName = fieldName;
    }

    public String getFieldPath() {
        return fieldPath;
    }

    public void setFieldPath(String fieldPath) {
        this.fieldPath = fieldPath;
    }
}
```
实现一个验证工具
```java
public class ValidatorUtil {

    /**
     * 获取所有字段
     * @param clz
     */
    public static List<Field> getAllField(Class clz){
        Field[] declaredFields = clz.getDeclaredFields();
        Field[] fields=clz.getFields();
        List<Field> fieldList=new ArrayList<>(fields.length+declaredFields.length);
        for(int i=0;i<declaredFields.length;i++){
            declaredFields[i].setAccessible(true);
            fieldList.add(declaredFields[i]);
        }
        for(int i=0;i<fields.length;i++){
            fieldList.add(fields[i]);
        }
        return fieldList;
    }

    /**
     * 验证对象的某个字段是否符合规则
     * @param field 字段
     * @param object 对象
     * @param results 错误结果存储
     * @return
     * @throws IllegalAccessException
     * @throws InstantiationException
     */
    private static boolean validatorField(Field field, Object object,List<ValidatorResult> results) throws IllegalAccessException, InstantiationException {
        Annotation[] annos = field.getAnnotations();
        for(Annotation annotation:annos){
            ValidatorBind validatorBind=annotation.annotationType().getAnnotation(ValidatorBind.class);
            if(validatorBind!=null){
               Validator validator= (Validator) validatorBind.validator().newInstance();
               String msg=validator.validate(field.get(object),annotation);
               if(msg!=null){
                   ValidatorResult validatorResult=new ValidatorResult();
                   validatorResult.setFieldName(field.getName());
                   validatorResult.setMessage(msg);
                   results.add(validatorResult);
                   return false;
               };
            }
        }
        return true;
    }

    /**
     * 验证对象
     * @param object
     * @return 错误结果合集
     */
    public static List<ValidatorResult> validator(Object object){
        //返回结果
        List<ValidatorResult> results=new ArrayList<>();
        Class clz=object.getClass();
        List<Field> fields=getAllField(clz);
        for(Field field:fields){
            try {
                validatorField(field,object,results);
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InstantiationException e) {
                e.printStackTrace();
            }
        }
        return results;
    }
}
```

上面一个验证框架就完成了
下面是增加验证功能
增加一个NotNull验证
```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@ValidatorBind(validator = NotNullValidator.class)
@interface NotNull {
    String msg() default "不能为空";
}
```
实现类
```java
/**
 * 验证Object不是NULL
 */
public class NotNullValidator implements Validator<Object,NotNull> {

    /**
     *  验证不能为NULL
     * @param value 值 Object类型
     * @param anno  注解 NotNull注解
     * @return
     */
    @Override
    public String validate(Object value, NotNull anno) {
        if(value==null){
            return anno.msg();
        }
        return null;
    }
}
```
增加正则验证
```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@ValidatorBind(validator = PatternValidator.class)
public @interface Pattern {
    String regexp() default "";
    String msg() default "不符合正则规则";
}
```
实现正则验证规则
```java
public class PatternValidator implements Validator<String,Pattern> {
    @Override
    public String validate(String value, Pattern anno) {
        String regexp=anno.regexp();
        if(!RegExp.test(regexp,value)){
            return anno.msg();
        }
        return null;
    }
}
```
测试类
```java
class A {
    @NotNull
    @Pattern(regexp = "^\\d+$",msg = "必须是纯数字")
    String name;

    @NotNull(msg = "地址不能为空")
    String address;
}

public class Main {

    public static void main(String ...args){
        A a=new A();
        a.name="asd";
        List<ValidatorResult> list = ValidatorUtil.validator(a);
        if(list.isEmpty()){
            System.out.println("通过验证");
        }else{
            for(ValidatorResult result:list) {
                System.out.println(result.getFieldName()+' '+result.getMessage());
            }
        }
    }
}
```