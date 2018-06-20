---
title: 全局统一异常处理和@ControllerAdvice注解实现源码解析
date: 2018-06-20 17:41:20
tags: Java
---

更新中。。。
本文将介绍目前两种常用的全局统一异常处理方式，并从源码上分析@ControllerAdvice注解的实现原理。
<!--more-->

### 前言

异常处理是开发过程中一个不可避免的问题，你是否写过这样的代码？你是否还在写这样的代码？你是否已经厌倦写这样的代码？

```java
@RestController
@RequestMapping(value = "/api/v1/tasks")
@Slf4j
public class TaskController implements ITaskController {

	@Autowired
	private ITaskService taskService;
  
	@PostMapping
	@Override
	public Result<String> addTask(@RequestBody AddTaskDTO task) {
		Result<String> result = null;
		try {
			/*
				...
			*/
			taskService.addTask(task);
			result = Result.success("新增成功");  
  		} catch (IllegalArgumentException e) {
      		result = Result.fail("参数错误");
   		} catch (PermissionException e) {
     		result = Result.fail("权限校验失败");
  		} catch (BusinessException e) {
    		result = Result.fail("业务异常");
  		} catch (Exception e) {
    		result = Result.fail("未知错误");
 		}

		return result;
    }
}
```

看看这无止境的try...catch...，全局统一异常处理了解一下？

全局统一异常处理是一种非常方便的异常处理方式，它能够极大的简化控制层和服务层的代码，这使得开发者不再需要花大量精力去关注异常的抛出和捕获，从而把更多的精力放在处理业务逻辑上。如果使用了全局统一异常处理，我们的代码将会变成什么样？让我们看看下面的代码来对比一下。

```java
@RestController
@RequestMapping(value = "/api/v1/tasks")
@Slf4j
public class TaskController implements ITaskController {

	@Autowired
	private ITaskService taskService;

	@PostMapping
	@Override
	public Result<String> addTask(@RequestBody AddTaskDTO task) {
		taskService.addTask(task);
		return Result.sucess("新增成功");
  	}
	
}
```
是不是非常优雅？

目前有两种比较常用的全局统一异常处理方式：

* HandlerExceptionResolver接口
* @ControllerAdvice注解

我们先介绍如何使用这两种方式进行全局统一异常处理，然后从源码上分析@ControllerAdvice注解的实现原理。

### 准备工作
首先，自定义一个异常基类BusinessException，用来表示服务可能会抛出的业务异常。

```java
@Slf4j
public class BusinessException extends RuntimeException {

    private static final long serialVersionUID = 5917336312549960151L;

    public BusinessException() {
        super();
    }

    public BusinessException(String message) {
        super(message);
    }

    public BusinessException(String message, Throwable cause) {
        super(message, cause);
    }

}
```

可以在该异常的基础上通过继承进行扩展，以满足自己的需求。

#### HandlerExceptionResolver接口的使用方式
我们先来看一下HandlerExceptionResolver接口的源码。

```java
public interface HandlerExceptionResolver {

	ModelAndView resolveException(
			HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex);

}
```

该接口只声明了一个resolveException方法，我们需要做的就是自定义一个全局统一异常处理类并实现该接口，如下所示：

```java
@Order(-1000)
@Slj4j
public class ExceptionResolver implements HandlerExceptionResolver {
    
    @Override
    public ModelAndView resolveException(HttpServletRequest request,
            HttpServletResponse response, Object handler, Exception ex) {
        
        if (ex instanceof BussinessException) {
        		/*
					处理BussinessException的逻辑
				*/
        } else {
        		/*
					处理其他异常的逻辑
				*/
        }
        
        Result<String> result = Result.fail(e.getMessage());
                
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding("UTF-8");
        response.setHeader("Cache-Control", "no-cache, must-revalidate");  
        try {
            response.getWriter().write(JSON.toJSONString(result));
        } catch (IOException e) {
            log.error("与客户端通讯异常：" + e.getMessage(), e);
        }
        
        log.error(ex.getMessage(), ex);
        
        return new ModelAndView();
	}
  
}
```

resovleException方法负责解析并处理异常，可以在该方法里面实现自定义的异常处理逻辑。该方法返回ModelAndView，所以对于前后端分离的开发场景，需要手动打开流并写回JSON格式。另外需要注意的是，需要在这个类上加上@Order注解，因为Spring默认有三个异常拦截器，里面的order属性分别为0，1，2，当异常被捕获时，会首先去这三个拦截器中寻找匹配的异常，若有匹配的，则不会执行我们自定义的异常处理器。@Order(-1000)的作用就是将我们自定义的异常处理器的顺序提升到第一位，先加载我们自定义的异常处理器，有匹配的异常时，则不会继续走其他三个默认的异常处理器。若请求没有抛异常，则此类的resovleException方法是不会执行的。

#### @ControllerAdvice注解的使用方式
这种方式相对于上一种来说，代码更简单，逻辑更清晰易懂。同样，先定义一个全局统一异常处理类，如下所示：

```java
@ControllerAdvice
@Order(-1000)
@Slf4j
public class UnitedExceptionHandleAdvice {

    @ExceptionHandler(BusinessException.class)
    @ResponseBody
    public Result<String> handleBusinessException(BusinessException e) {
        return processBusinessException(e);
    }

    @ExceptionHandler(Exception.class)
    @ResponseBody
    public Result<String> handleOtherException(Exception e) {
        return processDefault(e);
    }
    
    private static Result<String> processBusinessException(BussinessException e) {
        log.error(e.getMessage(), e);
        return Result.fail(e.getMessage());
    }

    private static Result<String> processDefault(Exception e) {
        log.error(e.getMessage(), e);
        return Result.fail("未知错误");
    }

}
```

然后在这个类上加上@ControllerAdvice注解，同时加上@Order(-1000)，原理同上。在类中声明自定义的异常处理方法，并配合@ExceptionHandler注解指明这个方法可以处理哪一种类型的异常。同时配合@ResponseBody注解，可以轻松的返回JSON格式的数据。

推荐使用@ControllerAdvice注解的方式实现全局统一异常处理。

#### @ControllerAdvice注解的实现源码分析

首先来看看这个注解的源码。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface ControllerAdvice {

	@AliasFor("basePackages")
	String[] value() default {};

	@AliasFor("value")
	String[] basePackages() default {};

	Class<?>[] basePackageClasses() default {};

	Class<?>[] assignableTypes() default {};

	Class<? extends Annotation>[] annotations() default {};

}
```

源码非常简单，没有什么特别之处，表面上看不出任何功能的实现，但是注意它的Retention是Runtime，说明这个注解会保留到代码运行期间。

接下来，我们模拟一个请求，服务在处理这个请求的过程中会抛出一个异常，然后这个异常会被统一异常切面捕获，让我们通过Debug，来看清楚整个请求和异常的处理过程。


更新中。。。


通过上面的Debug，我们发现了统一异常处理切面的核心所在：ExceptionHandlerExceptionResolver类中的exceptionHandlerAdviceCache。
接下来看看ExceptionHandlerExceptionResolver的源码，来弄清楚exceptionHandlerAdviceCache是如何初始化的。

```java
public class ExceptionHandlerExceptionResolver extends AbstractHandlerMethodExceptionResolver
		implements ApplicationContextAware, InitializingBean {
	
	private final Map<ControllerAdviceBean, ExceptionHandlerMethodResolver> exceptionHandlerAdviceCache =
			new LinkedHashMap<ControllerAdviceBean, ExceptionHandlerMethodResolver>();
			
	@Override
	public void afterPropertiesSet() {
		// Do this first, it may add ResponseBodyAdvice beans
		initExceptionHandlerAdviceCache();

		if (this.argumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.returnValueHandlers == null) {
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
	}
	
	private void initExceptionHandlerAdviceCache() {
		if (getApplicationContext() == null) {
			return;
		}
		if (logger.isDebugEnabled()) {
			logger.debug("Looking for exception mappings: " + getApplicationContext());
		}

		List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());
		AnnotationAwareOrderComparator.sort(adviceBeans);

		for (ControllerAdviceBean adviceBean : adviceBeans) {
			ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(adviceBean.getBeanType());
			if (resolver.hasExceptionMappings()) {
				this.exceptionHandlerAdviceCache.put(adviceBean, resolver);
				if (logger.isInfoEnabled()) {
					logger.info("Detected @ExceptionHandler methods in " + adviceBean);
				}
			}
			if (ResponseBodyAdvice.class.isAssignableFrom(adviceBean.getBeanType())) {
				this.responseBodyAdvice.add(adviceBean);
				if (logger.isInfoEnabled()) {
					logger.info("Detected ResponseBodyAdvice implementation in " + adviceBean);
				}
			}
		}
	}
		
}
```

可以看到，ExceptionHandlerExceptionResolver实现了InitializingBean接口，这个接口只定义一个afterPropertiesSet方法，如下所示：

```java
public interface InitializingBean {
	void afterPropertiesSet() throws Exception;
}
```

这个接口的作用就是在Spring容器初始化bean的时候用来完成一些初始化操作。ExceptionHandlerExceptionResolver实现了该接口，并在afterPropertiesSet方法中调用了exceptionHandlerAdviceCache的初始化方法，我们来看一下其关键的部分：通过ControllerAdviceBean.findAnnotatedBeans方法从上下文中找出所有注解了@ControllerAdvice的类，findAnnotatedBeans方法的实现如下所示：

```java
public class ControllerAdviceBean implements Ordered {

	public static List<ControllerAdviceBean> findAnnotatedBeans(ApplicationContext applicationContext) {
		List<ControllerAdviceBean> beans = new ArrayList<ControllerAdviceBean>();
		for (String name : BeanFactoryUtils.beanNamesForTypeIncludingAncestors(applicationContext, Object.class)) {
			if (applicationContext.findAnnotationOnBean(name, ControllerAdvice.class) != null) {
				beans.add(new ControllerAdviceBean(name, applicationContext));
			}
		}
		return beans;
	}
	
}
```


找到之后封装成ExceptionHandlerMethodResolver，装入exceptionHandlerAdviceCache中，从而完成初始化动作。应用启动后，对于请求抛出的异常，将在exceptionHandlerAdviceCache中寻找合适的处理器的对应方法进行处理。

### 总结

