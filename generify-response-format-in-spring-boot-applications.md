---
title: "在Spring Boot应用中配置统一的请求响应"
source: https://www.boris1993.com/generify-response-format-in-spring-boot-applications.html
date: 2019-12-21
updated: 2026-10-10
tags: [Java, Spring Boot, 统一响应]
categories: [小技巧]
---

# 在Spring Boot应用中配置统一的请求响应

在前后端分离的架构下，后端通常是一个RESTFul的接口，而因为HTTP的响应码数量有限，无法灵活的反映出接口执行的各种结果，在这种情况下，就需要通过自定义的结构来表达接口最终的状态和返回的信息。而我正好最近在一个项目中实现了一个基于`ControllerAdvice`的统一请求响应的功能，在这里记录一下实现的方式。

## 创建common模块

因为这是一个公共的功能，所以需要创建一个新的Maven模块，并被所有项目引用为依赖。具体操作这里不再赘述。以下的所有代码，如无特殊说明，都将存在于这个common模块中。

## 定义全局的错误码

首先我们需要定义一个全局的错误码，使得项目中的所有模块都可以使用统一的一套返回码来表达自己接口的状态。

```java
/**
 * 接口返回码和描述
 *
 * @author Boris Zhao
 */
@Getter
public enum ReturnCode {
    /**
     * 成功
     */
    OK("0000", "成功"),

    /**
     * 服务端异常，当发生未知异常时使用该错误码
     */
    FAIL("9999", "失败"),

    /**
     * 请求参数中包含无效参数或请求体为空
     */
    INVALID_REQUEST_PARAM("0001", "请求参数中包含无效参数或请求体为空"),

    /**
     * 新数据的主键与已有数据重复
     */
    DUPLICATED_RECORD("0002", "新数据的主键与已有数据重复"),

    /**
     * 未找到对应记录
     */
    NON_EXISTENT_RECORD("0003", "未找到对应记录，请检查主键或操作流水号"),

    /**
     * 签名校验失败
     */
    SIGNATURE_VERIFICATION_FAIL("0004", "签名校验失败"),

    // 以下为各模块自定义的错误码
    ;

    private String code;
    private String message;

    ReturnCode(final String code, final String message) {
        this.code = code;
        this.message = message;
    }

    /**
     * 根据状态码获取其错误信息
     *
     * @param code 状态码
     * @return 错误码对应的错误信息。如果没有找到则返回{@code null}
     */
    public static String getMessageByCode(String code) {
        for (ReturnCode item : values()) {
            if (item.code.equals(code)) {
                return item.message;
            }
        }

        return null;
    }
}
```

## 定义统一响应结构

在这个项目中，我选择在这个结构中定义三个字段：错误码`errCode`，错误信息`errMessage`，和返回的数据`data`。

同时，用于构造响应体的类应该同时兼顾数据合法性和灵活性，所以我决定不允许通过构造方法或者setter来填充信息，而是使用定义好了的静态方法来完成构造。

```java
/**
 * 公共响应参数<br>
 * 成功的返回通过{@link CommonResponseParams#ofSuccessful()}或{@link CommonResponseParams#ofSuccessful(Object)}生成<br>
 * 失败的返回通过{@link CommonResponseParams#ofFailure()}或{@link CommonResponseParams#ofFailure(ReturnCode)}生成
 *
 * @author Boris Zhao
 */
@Data
public class CommonResponseParams {
    /**
     * 返回码 - 必填
     */
    private String errCode;

    /**
     * 返回描述 - 必填
     */
    private String errMessage;

    /**
     * 业务数据 - 必填
     */
    private Object data;

    /**
     * 构造一个{@link CommonResponseParams}对象
     *
     * @param errCode    返回码
     * @param errMessage 返回描述
     * @param data    业务数据
     */
    private CommonResponseParams(final String errCode, final String errMessage, final Object data) {
        this.errCode = errCode;
        this.errMessage = errMessage;
        this.data = data;
    }

    /**
     * 返回成功结果，没有响应数据
     *
     * @return 公共响应参数实体
     */
    public static CommonResponseParams ofSuccessful() {
        return ofSuccessful(null);
    }

    /**
     * 返回成功结果
     *
     * @param content 返回的数据
     * @param <T>     返回的数据的类型
     * @return 公共响应参数实体
     */
    public static <T> CommonResponseParams ofSuccessful(final T content) {
        return new CommonResponseParams(
                ReturnCode.OK.getCode(),
                ReturnCode.OK.getMessage(),
                JSONArray.toJSON(content));
    }

    /**
     * 返回失败结果
     *
     * @return 公共响应参数实体
     */
    public static CommonResponseParams ofFailure() {
        return new CommonResponseParams(
                ReturnCode.FAIL.getCode(),
                ReturnCode.FAIL.getMessage(),
                null);
    }

    public static CommonResponseParams ofFailure(String errMessage) {
        return new CommonResponseParams(
                ReturnCode.FAIL.getCode(),
                errMessage,
                null);
    }

    /**
     * 返回失败结果
     *
     * @param returnCode 错误的返回码
     * @return 公共响应参数实体
     */
    public static CommonResponseParams ofFailure(ReturnCode returnCode) {
        return new CommonResponseParams(
                returnCode.getCode(),
                returnCode.getMessage(),
                null);
    }

    /**
     * 返回带有自定义错误信息的失败结果
     *
     * @param returnCode 错误相关的返回码
     * @param errMessage 自定义的错误信息
     * @return 公共响应参数实体
     */
    public static CommonResponseParams ofFailure(ReturnCode returnCode, String errMessage) {
        return new CommonResponseParams(
                returnCode.getCode(),
                errMessage,
                null);
    }
}
```

## 定义统一的业务异常基类

为了减少不必要的`try-catch`模版代码，业务异常必须不能为受检异常；而为了与其它的运行时异常区分开来，业务异常类就不能直接继承`RuntimeException`，而是需要继承于一个自定义的基类。同时，这个业务异常基类不能被直接使用，所以必须是一个抽象类。

```java
/**
 * 业务异常基类
 *
 * @author Boris Zhao
 * @date 2019-12-13
 */
@Getter
public abstract class BaseBizException extends RuntimeException {
    protected ReturnCode returnCode = null;

    public BaseBizException(String message) {
        super(message);
    }

    public BaseBizException(ReturnCode returnCode) {
        super(returnCode.getMessage());
        this.returnCode = returnCode;
    }

    /**
     * 业务异常不记录stack trace
     */
    @Override
    public synchronized Throwable fillInStackTrace() {
        return this;
    }
}
```

## 定义统一的异常处理方法

在上面的准备工作全部完成后，就可以开始着手配置统一的异常处理方法。之所以选择不使用`AOP`实现，是因为在这个情况下，业务接口必须返回`Object`类型，而这样一来，会降低代码层面的可读性。使用`ControllerAdvice`注解实现则没有这个限制，业务接口可以自由选择自己合适的数据类型。

需要注意的是，因为我们所有的controller类都会带有`RestController`注解，所以在`ControllerAdvice`注解中，我们使用`annotations`参数指定了这个配置类仅针对带有`RestController`的类启用。

```java
/**
 * 统一异常处理配置类<br>
 * 包装格式见{@link CommonResponseParams}
 *
 * @author Boris Zhao
 * @date 2019-12-13
 */
@Slf4j
@ResponseBody
@ControllerAdvice(annotations = RestController.class)
public class UnifiedExceptionHandler {

    /**
     * 处理数据库连接失败抛出的异常
     *
     * @return 带有数据库连接失败信息的失败返回
     */
    @ExceptionHandler(CannotCreateTransactionException.class)
    public CommonResponseParams handleCannotCreateTransactionException(CannotCreateTransactionException e) {
        log.error(e.getMessage(), e);
        return CommonResponseParams.ofFailure("数据库连接失败");
    }

    /**
     * 处理未知的运行时错误
     *
     * @return 默认的失败返回
     */
    @ExceptionHandler(RuntimeException.class)
    public CommonResponseParams handleUnknownRuntimeExceptions(RuntimeException e) {
        log.error(e.getMessage(), e);
        return CommonResponseParams.ofFailure(e.getMessage());
    }

    /**
     * 处理公共请求参数校验失败异常
     *
     * @param e 参数校验失败抛出的异常
     * @return 带有校验失败原因提示信息的失败返回
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public CommonResponseParams handleRequestParamValidationExceptions(MethodArgumentNotValidException e) {
        String errMessage = Optional.ofNullable(e.getBindingResult().getFieldError())
                .map(FieldError::getDefaultMessage)
                .orElse(ReturnCode.INVALID_REQUEST_PARAM.getMessage());

        log.error(e.getMessage());
        return CommonResponseParams.ofFailure(ReturnCode.INVALID_REQUEST_PARAM, errMessage);
    }

    /**
     * 处理请求body为空的异常
     *
     * @return 带有请求体无效错误的失败返回
     */
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public CommonResponseParams handleHttpMessageNotReadableException() {
        return CommonResponseParams.ofFailure(ReturnCode.INVALID_REQUEST_PARAM);
    }

    /**
     * 处理新增数据主键重复异常
     *
     * @return 带有主键重复错误的失败返回
     */
    @ExceptionHandler(DuplicateKeyException.class)
    public CommonResponseParams handleDuplicateKeyException() {
        return CommonResponseParams.ofFailure(ReturnCode.DUPLICATED_RECORD);
    }

    /**
     * 处理业务异常
     *
     * @return 业务异常对应的失败返回
     */
    @ExceptionHandler(BaseBizException.class)
    public CommonResponseParams handleBizExceptions(BaseBizException e) {
        if (e.getReturnCode() != null) {
            ReturnCode returnCode = e.getReturnCode();
            log.error(returnCode.getMessage());
            return CommonResponseParams.ofFailure(returnCode);
        } else if (StringUtils.isNotBlank(e.getMessage())) {
            log.error(e.getMessage());
            return CommonResponseParams.ofFailure(e.getMessage());
        } else {
            log.error(e.getMessage());
            return CommonResponseParams.ofFailure();
        }
    }
}
```

这里详细说一下各个方法的作用。

第一个方法用于处理`CannotCreateTransactionException`异常类，这个异常会在应用无法成功连接数据库时被抛出。处理方式就是返回一个错误信息为“数据库连接失败”的失败结果。

第二个方法用于处理`RuntimeException`异常，这个方法的意义在于，我们无法预见所有可能出现的异常，所以使用这个方法作为一个兜底的处理方法。

第三个方法用于处理`MethodArgumentNotValidException`异常。因为这个项目中我们选择使用`javax.validation.constraints`包中的注解实现输入参数合法性的校验，而当校验失败时会抛出`MethodArgumentNotValidException`异常，并且在异常中会包含具体的校验失败的原因。同时为了保证方法的健壮性，在代码中也保证了如果无法获取到校验失败信息，就会选择`INVALID_REQUEST_PARAM`这个错误码作为兜底的错误信息。

第四个方法用于处理`HttpMessageNotReadableException`异常。如果一个接口方法的参数中存在被`@RequestBody`标记的参数，但是在请求该接口时body为空时，就会抛出这个异常。在出现了这个异常后，就会返回带有`INVALID_REQUEST_PARAM`错误信息的失败结果。

第五个方法用于处理`DuplicateKeyException`异常。因为这个项目中一部分数据的主键是由请求发起方生成的，同时数据库中也会将这一列定为主键来实现插入接口的幂等性。一旦出现网络状况不佳的情况时，发起方会尝试再次调用接口。而在重发请求时，可能数据已经在上一个请求中就已经成功插入了，只是因为网络不佳导致发起方没能接收到返回，在第二次请求中重复插入相同主键的数据，就会抛出这个异常。为了最终接口返回信息的可读性，我们选择在这里返回一个用户友好的信息。

最后一个方法就是这里的主角了，它用于处理所有继承了`BaseBizException`的业务异常。这个方法中，我们对应着`CommonResponseParams`中不同的静态方法，实现了对应的错误处理逻辑。

## 定义统一的成功响应处理方法

上面洋洋洒洒写了一堆针对异常的处理逻辑，但是接口成功执行的处理逻辑也不能落下。这里我们使用`RestControllerAdvice`表示这是一个接口增强类，同时实现了`ResponseBodyAdvice`接口，用于实现实际的处理逻辑。

在这个配置类上，我们也指定了该配置类仅针对被`RestController`标记的类生效。

```java
/**
 * 统一响应格式配置类<br>
 * 包装格式见{@link CommonResponseParams}
 *
 * @author Boris Zhao
 * @date 2019-12-13
 */
@EnableWebMvc
@Configuration
@RestControllerAdvice(annotations = RestController.class)
public class UnifiedReturnConfig implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    @Override
    public Object beforeBodyWrite(Object body,
                                  MethodParameter returnType,
                                  MediaType selectedContentType,
                                  Class<? extends HttpMessageConverter<?>> selectedConverterType,
                                  ServerHttpRequest request,
                                  ServerHttpResponse response) {
        if (body instanceof CommonResponseParams) {
            return body;
        }

        return CommonResponseParams.ofSuccessful(body);
    }
}
```

上面代码的重点是在`beforeBodyWrite`方法中。这个方法会在`HttpMessageConverter#write()`方法执行前，也就是返回被发出去之前被调用。借助这个功能，我们就可以实现在业务接口返回之后，将返回信息重新包装。

实现逻辑很简单，如果返回信息是一个`CommonResponseParams`对象，那么就认为这个返回信息已经被包装好了，所以不再进行二次包装，直接返回；否则就通过`CommonResponseParams#ofSuccessful()`方法，将返回信息包装为一个成功响应的格式，再返回到客户端。

## 最后的一点配置

在上文中，统一返回格式的配置已经完成了。但是有的人可能会发现，虽然在自己的项目中引用了这个模块，但是实际上却没有生效，这是因为上面的配置类都存在于另一个jar包中，导致在应用启动时这些请求并没有被自动发现。解决方法也很简单，在项目的启动类(即`xxxApplication`)中加上`@ComponentScan`注解，并在注解参数中加上`UnifiedReturnConfig`和`UnifiedExceptionHandler`所在的包名即可。
