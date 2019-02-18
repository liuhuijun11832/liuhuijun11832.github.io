---
title: Spring MVC异常增强处理
categories: 编程技术
date: 2019-02-18 13:18:56
tags: Java
---
# 简述
记录一些关于平时借助Spring MVC框架开发时，对异常处理的一些想法和总结。这里引用耗子叔的总结，将异常分为三大类：

* 资源的错误，例如没有打开文件权限，写文件出现错误，网络故障等。
* 程序的错误，比如空指针，非法参数的异常，这一类最好记录下来写入日志，并触发监控系统报警。
* 用户的错误，比如缺少参数，请求方式错误，这类错误通常由于用户错误操作导致的，对于这类错误我们需要做统计，有利于改善软件或者侦测是否有恶意请求，并反馈给用户修正。

		对于我们能够预知并且需要告诉用户修正的错误，我们最好使用返回	码的形式；
		对于我们不期望发生的事，我们可以使用异常捕捉。
<!--more-->

# 代码
Java里常用try...catch...finally的形式来建立异常堆栈，但是这种做法的缺点是重复代码较多，每个需要处理的接口都需要加上如下这几行：

```java
try {
            ...
        } catch (XmanException e) {
            ...
        } catch (Exception e) {
            ...
        } finally {
            ...
        }
```
Spring MVC提供了控制器增强来方便进行统一异常处理，首先可以自定义一个业务中的异常：

```java
/**
 * @Description: 自定义用户异常
 * @Author: 刘会俊
 * @Date: 2019-02-14 10:55
 */
public class CustomException extends RuntimeException {

    private static final long serialVersionUID = 4564124491192825748L;

    private int code;

    public CustomException(int code) {
        super();
    }

    public CustomException(String message, int code) {
        super(message);
        this.setCode(code);
    }

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }
}
```
这样我们可以在业务层检测到用户参数不合法，或者某些受检异常的时候，可以直接抛出该exception，接下来定义一个接口层VO或者叫DTO，用于异常产生时，返回特定格式的数据：

```java
/**
 * @Description: 异常信息模板
 * @Author: 刘会俊
 * @Date: 2019-02-14 10:59
 */
public class ErrorResponseEntity {

    private int code;

    private String message;

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public ErrorResponseEntity(int code, String message) {
        this.code = code;
        this.message = message;
    }
}
```
当产生错误时，统一返回该数据传输对象，最后定义Spring MVC的控制器增强。我们的业务逻辑和控制逻辑通常都是分开的，在控制逻辑里进行自定义异常的处理：

```java
/**
 * @Description: 全局异常处理，@RestControllerAdvice是@ResponseBody和ControllerAdvice的组合注解
 * @Author: 刘会俊
 * @Date: 2019-02-14 11:02
 */
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    
    private final static Logger LOGGER = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    /**
     *@description: 处理自定义异常，获取自定义异常码和信息
     *@author: 刘会俊
     *@params: [request, e, response]
     *@return: com.example.aisino.exception.ErrorResponseEntity
     */
    @ExceptionHandler(CustomException.class)
    public ErrorResponseEntity customExceptionHandler(HttpServletRequest request, Exception e, HttpServletResponse response){
        response.setStatus(HttpStatus.BAD_REQUEST.value());
        CustomException customException = (CustomException) e;
        return new ErrorResponseEntity(customException.getCode(),customException.getMessage());
    }

    /**
     *@description: 处理运行时异常，统一定义为500-服务器错误
     *@author: 刘会俊
     *@params: [request, e, response]
     *@return: com.example.aisino.exception.ErrorResponseEntity
     */
    @ExceptionHandler(RuntimeException.class)
    public ErrorResponseEntity runtimeExceptionHandler(HttpServletRequest request,Exception e,HttpServletResponse response){
        response.setStatus(500);
        RuntimeException runtimeException = (RuntimeException) e;
        return new ErrorResponseEntity(500, runtimeException.getMessage());
    }

    @Override
    protected ResponseEntity<Object> handleExceptionInternal(Exception ex, Object body, HttpHeaders headers, HttpStatus status, WebRequest request) {
        if (ex instanceof MethodArgumentNotValidException){
            MethodArgumentNotValidException methodArgumentNotValidException = (MethodArgumentNotValidException) ex;
            return new ResponseEntity<>(new ErrorResponseEntity(status.value(), methodArgumentNotValidException.getBindingResult().getAllErrors().get(0).getDefaultMessage()), status);
        }
        if(ex instanceof MethodArgumentTypeMismatchException){
            MethodArgumentTypeMismatchException methodArgumentTypeMismatchException = (MethodArgumentTypeMismatchException) ex;
            LOGGER.error("参数转换失败，方法{},参数{},信息{}",
                    methodArgumentTypeMismatchException.getParameter().getMethod().getName(),
                    methodArgumentTypeMismatchException.getName(),
                    methodArgumentTypeMismatchException.getLocalizedMessage());
            return new ResponseEntity<>(new ErrorResponseEntity(status.value(), "参数转换失败"), status);
        }
        return new ResponseEntity<>(new ErrorResponseEntity(status.value(), "请求失败！"), status);
    }
}
```
其中，@ControllerAdvice支持的限定范围有：

1. `@ControllerAdvice(annotation=RestController.clcas)`，按注解；
1. ` @ControllerAdvice("org.example.controllers")`，按包名；
1. `@ControllerAdvice(assignableTypes={ControllerInterface.class,AbstractController.class})`，按类型。

如果在Controller里某个方法直接使用`@ExceptionHandler(RuntimeException.class)`注解，则表示该异常处理只针对于该Controller。

