---
title: DispatcherServlet请求处理流程解析
date: 2020-05-09 11:00:32
tags: "Spring MVC"
---
## DispatcherServlet 继承关系

![Spring DispatcherServlet](/images/DispatcherServlet.png)

### FrameworkServlet

```java
protected final void doGet(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {

    processRequest(request, response);
}

protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {

    long startTime = System.currentTimeMillis();
    Throwable failureCause = null;

    LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
    LocaleContext localeContext = buildLocaleContext(request);

    RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
    ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
    asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());

    initContextHolders(request, localeContext, requestAttributes);

    // 上面都是一些准备工作，重点看doService方法；doService是一个抽象方法，由子类实现并处理request请求
    try {
        doService(request, response);
    }
    // 省略了catch和finally相关代码
}
```

### DispatcherServlet

DispatcherServlet 实现了 doService方法

```java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
    // Make framework objects available to handlers and view objects.
    request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
    request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
    request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
    request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

    FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
    if (inputFlashMap != null) {
        request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
    }
    request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
    request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);

    // 上面设置request的一些属性，重点看doDispatch方法
    try {
        doDispatch(request, response);
    }
    // 省略了catch和finally相关代码
}

protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // 从handlerMappings找到处理request的处理器handler
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null || mappedHandler.getHandler() == null) {
                // 没有找到对应的处理器，返回404状态
                noHandlerFound(processedRequest, response);
                return;
            }

            // 从handlerAdapters找到支持handler的HandlerAdapter
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // Process last-modified header, if supported by the handler.
            String method = request.getMethod();
            boolean isGet = "GET".equals(method);
            if (isGet || "HEAD".equals(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (logger.isDebugEnabled()) {
                    logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
                }
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }

            // 请求处理之前，调用拦截器的preHandle方法预处理，一般是权限验证，如果返回false，结束请求
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // 调用AnnotationMethodHandlerAdapter的handler方法处理request请求，返回ModelAndView
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }

            applyDefaultViewName(processedRequest, mv);
            // 请求处理之后渲染页面之前，调用拦截器的postHandle方法
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            dispatchException = ex;
        }

        // 处理handler返回的结果，即渲染页面
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    // 省略catch和finally代码块
}

private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
        HandlerExecutionChain mappedHandler, ModelAndView mv, Exception exception) throws Exception {

    boolean errorView = false;

    // 处理异常情况，自定义的异常处理类可以参考DefaultHandlerExceptionResolver
    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException) {
            logger.debug("ModelAndViewDefiningException encountered", exception);
            mv = ((ModelAndViewDefiningException) exception).getModelAndView();
        }
        else {
            Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
            mv = processHandlerException(request, response, handler, exception);
            errorView = (mv != null);
        }
    }

    // 渲染页面，通过视图名称进行视图解析
    if (mv != null && !mv.wasCleared()) {
        render(mv, request, response);
        if (errorView) {
            WebUtils.clearErrorRequestAttributes(request);
        }
    }
    else {
        if (logger.isDebugEnabled()) {
            logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + getServletName() +
                    "': assuming HandlerAdapter completed request handling");
        }
    }

    if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
        // Concurrent handling started during a forward
        return;
    }

    // 调用拦截器的afterCompletion，一般是处理资源回收操作
    if (mappedHandler != null) {
        mappedHandler.triggerAfterCompletion(request, response, null);
    }
}

protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
    // Determine locale for request and apply it to the response.
    Locale locale = this.localeResolver.resolveLocale(request);
    response.setLocale(locale);

    // 根据视图名称解析成视图对象，例如：视图解析器FreeMarkerViewResolver将视图名解析成FreeMarkerView并完成FreeMarkerView的初始化操作，设置FreeMarkerConfig
    View view;
    if (mv.isReference()) {
        // We need to resolve the view name.
        view = resolveViewName(mv.getViewName(), mv.getModelInternal(), locale, request);
        if (view == null) {
            throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
                    "' in servlet with name '" + getServletName() + "'");
        }
    }
    else {
        // No need to lookup: the ModelAndView object contains the actual View object.
        view = mv.getView();
        if (view == null) {
            throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
                    "View object in servlet with name '" + getServletName() + "'");
        }
    }

    // Delegate to the View object for rendering.
    if (logger.isDebugEnabled()) {
        logger.debug("Rendering view [" + view + "] in DispatcherServlet with name '" + getServletName() + "'");
    }
    try {
        // 视图对象执行渲染操作
        view.render(mv.getModelInternal(), request, response);
    }
    catch (Exception ex) {
        if (logger.isDebugEnabled()) {
            logger.debug("Error rendering view [" + view + "] in DispatcherServlet with name '" +
                    getServletName() + "'", ex);
        }
        throw ex;
    }
}
```

### AbstractView

下面解析视图渲染过程，实际是 renderMergedOutputModel 方法进行渲染，这是一个抽象方法，由子类实现

```java
public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
    if (logger.isTraceEnabled()) {
        logger.trace("Rendering view with name '" + this.beanName + "' with model " + model +
            " and static attributes " + this.staticAttributes);
    }

    // 合并属性
    Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
    prepareResponse(request, response);
    renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
}

protected abstract void renderMergedOutputModel(
        Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception;

```

### AbstractTemplateView

AbstractTemplateView 是 AbstractView 的一个子类，实现了 renderMergedOutputModel 方法

```java
protected final void renderMergedOutputModel(
        Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {


    // 将RequestContext添加到model中，默认添加
    if (this.exposeSpringMacroHelpers) {
        if (model.containsKey(SPRING_MACRO_REQUEST_CONTEXT_ATTRIBUTE)) {
            throw new ServletException(
                    "Cannot expose bind macro helper '" + SPRING_MACRO_REQUEST_CONTEXT_ATTRIBUTE +
                    "' because of an existing model object of the same name");
        }
        // Expose RequestContext instance for Spring macros.
        model.put(SPRING_MACRO_REQUEST_CONTEXT_ATTRIBUTE,
                new RequestContext(request, response, getServletContext(), model));
    }

    // 设置response的contentType
    applyContentType(response);

    // 抽象方法，由子类实现
    renderMergedTemplateModel(model, request, response);
}

protected abstract void renderMergedTemplateModel(
            Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception;
```

### FreeMarkerView

FreeMarkerView 是 AbstractTemplateView 的一个子类，负责视图解析

```java
protected void renderMergedTemplateModel(
        Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {

    // 可以在这里执行一些帮助操作，FreeMarkerView中此方法为空
    exposeHelpers(model, request);

    // FreeMarkerView执行渲染
    doRender(model, request, response);
}

protected void doRender(Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
    // 这里遍历model中的key-value，添加到request的属性中
    exposeModelAsRequestAttributes(model, request);

    // Expose all standard FreeMarker hash models.
    SimpleHash fmModel = buildTemplateModel(model, request, response);

    if (logger.isDebugEnabled()) {
        logger.debug("Rendering FreeMarker template [" + getUrl() + "] in FreeMarkerView '" + getBeanName() + "'");
    }
    // Grab the locale-specific version of the template.
    Locale locale = RequestContextUtils.getLocale(request);

    // 将FreeMarker模板处理结果响应到response
    processTemplate(getTemplate(locale), fmModel, response);
}

protected void processTemplate(Template template, SimpleHash model, HttpServletResponse response)
        throws IOException, TemplateException {

    template.process(model, response.getWriter());
}
```

接下来就看到 FreeMarker 核心的 API 了，通过 Configuration 创建 template

```java
Configuration cfg = new Configuration();
...
Template myTemplate = cfg.getTemplate("myTemplate.html");
```

#### freemarker.template.Template

```java
public void process(Object rootMap, Writer out)
throws TemplateException, IOException
{
    createProcessingEnvironment(rootMap, out, null).process();
}
```

#### freemarker.core.Environment

```java
public void process() throws TemplateException, IOException {
    Object savedEnv = threadEnv.get();
    threadEnv.set(this);
    try {
        // Cached values from a previous execution are possibly outdated.
        clearCachedValues();
        try {
            // 自动导入freemarker.properties中的auto_import和auto_include
            doAutoImportsAndIncludes(this);
            visit(getTemplate().getRootTreeNode());
            // It's here as we must not flush if there was an exception.
            if (getAutoFlush()) {
                out.flush();
            }
        } finally {
            // It's just to allow the GC to free memory...
            clearCachedValues();
        }
    } finally {
        threadEnv.set(savedEnv);
    }
}
```

### 附录

一个 FreeMarker 解析 ftl 模板的示例，其中，hello.ftl 是 ftl 模板，hello.html 是最终解析成 html 的文件

```java
Configuration configuration = new Configuration();
File outFile = null;
try {
    configuration.setDirectoryForTemplateLoading(new File("/WEB-INF/ftl/template"));
    Template template = configuration.getTemplate("hello.ftl");

    outFile = new File("hello.html");
    FileOutputStream fileOutputStream = new FileOutputStream(outFile);
    OutputStreamWriter writer = new OutputStreamWriter(fileOutputStream, "UTF-8");

    MapModel mapModel = new MapModel(dataMap, new BeansWrapper());
    template.process(mapModel, writer);
    writer.close();
    fileOutputStream.close();
} catch (Exception e) {
    LOGGER.info(e.getMessage(), e);
}
```
