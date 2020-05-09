---
title: Spring MVC笔记
tags:
---
org.springframework.web.filter.OncePerRequestFilter#doFilter
    org.springframework.web.filter.CharacterEncodingFilter#doFilterInternal
        org.springframework.web.servlet.FrameworkServlet#service
            org.springframework.web.servlet.FrameworkServlet#doGet
                org.springframework.web.servlet.FrameworkServlet#processRequest
                    org.springframework.web.servlet.DispatcherServlet#doService
                        org.springframework.web.servlet.DispatcherServlet#doDispatch // 调用getHandler方法获取HandlerExecutionChain处理器执行链，调用getHandlerAdapter方法获取HandlerAdapter处理器适配器，执行HandlerExecutionChain的applyPreHandle方法（拦截器preHandle方法），执行HandlerAdapter的handle方法返回ModelAndView，执行HandlerExecutionChain的applyPostHandle方法（拦截器postHandle)，调用processDispatchResult使用视图解析器解析；

                            org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter#handle
                            org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#handleInternal
                            org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#invokeHandlerMethod
                            org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod#invokeAndHandle
