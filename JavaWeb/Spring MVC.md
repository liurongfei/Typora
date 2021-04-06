# Srping MVC



## 1.流程图

![1](C:\Environment\Github\Typora\JavaWeb\1.png)

## 2.流程说明

1. 客户端发送请求
2. 服务器接收请求后交于DisPatcherServlet处理
3. DisPatcherServlet调用HandlerMapping接口分析请求映射，判断由哪些Controller处理请求
4. HandlerMapping接口分析完后返回HandlerExecutionChain，包括拦截器和可处理的RequestMapping方法
5. 从处理链中获取HandlerAdapter
6. 执行拦截器的pre方法
7. 执行HandlerAdapter的handle方法，即RequestMapping中的方法，并返回ModelAndView
8. 执行拦截器的post方法
9. 



## 3.源码分析

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        try {
            ModelAndView mv = null;
            Object dispatchException = null;

            try {
                processedRequest = this.checkMultipart(request);
                multipartRequestParsed = processedRequest != request;
                //通过MappingHandler获取HandlerExecutionChain处理链
                //HandlerExecutionChain中包括handle处理类和List<Inteceptor>拦截器集合
                mappedHandler = this.getHandler(processedRequest);
                if (mappedHandler == null) {
                    this.noHandlerFound(processedRequest, response);
                    return;
                }
			   //从MappingHandler映射处理类中获取HandlerAdapter适配器
                HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler());
                String method = request.getMethod();
                boolean isGet = "GET".equals(method);
                if (isGet || "HEAD".equals(method)) {
                    long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                    if ((new ServletWebRequest(request, response)).checkNotModified(lastModified) && isGet) {
                        return;
                    }
                }
			   //执行拦截器的pre方法，拦截了则直接返回
                if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                    return;
                }
			  //执行Controller中的方法，并返回ModelAndView
                mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
                if (asyncManager.isConcurrentHandlingStarted()) {
                    return;
                }
			   //从ModelAndView中获取视图
                this.applyDefaultViewName(processedRequest, mv);
                //执行拦截器的post方法，拦截了则直接返回
                mappedHandler.applyPostHandle(processedRequest, response, mv);
            } catch (Exception var20) {
                dispatchException = var20;
            } catch (Throwable var21) {
                dispatchException = new NestedServletException("Handler dispatch failed", var21);
            }

            this.processDispatchResult(processedRequest, response, mappedHandler, mv, (Exception)dispatchException);
        } catch (Exception var22) {
            this.triggerAfterCompletion(processedRequest, response, mappedHandler, var22);
        } catch (Throwable var23) {
            this.triggerAfterCompletion(processedRequest, response, mappedHandler, new NestedServletException("Handler processing failed", var23));
        }

    } finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            if (mappedHandler != null) {
                //执行拦截器的aterCompetition方法
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        } else if (multipartRequestParsed) {
            this.cleanupMultipart(processedRequest);
        }

    }
}
```



## 4.设计模式

#### 适配者模式

将不同的Controller类统一适配成HandlerAdapter，统一使用HandlerAdapter中的handle()执行Controller中的方法

在Spring MVC中有5种适配器，分别对应5种实现Controller接口的方式，如果是普通的@RequestMapping注解的方式，则使用`SimpleControllerHandlerAdapter `适配器，将所有的Controller接口传入hanle()方法中，统一调用方法



