---
title: 非spring-cloud项目增加hystrix监控
date: 2019-05-07 17:11:30
categories:
tags:
  - hystrix
---


## 增加依赖
pom文件中
```xml
 <dependency>
  <groupId>com.netflix.hystrix</groupId>
  <artifactId>hystrix-metrics-event-stream</artifactId>
  <version>${hystrix.version}</version>
 </dependency>
```
## 添加servlet
web.xml 中
```xml
  <servlet>
    <display-name>HystrixMetricsStreamServlet</display-name>
    <servlet-name>HystrixMetricsStreamServlet</servlet-name>
    <servlet-class>com.netflix.hystrix.contrib.metrics.eventstream.HystrixMetricsStreamServlet
    </servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>HystrixMetricsStreamServlet</servlet-name>
    <url-pattern>/hystrix.stream</url-pattern>
  </servlet-mapping>
```
## 增加Basic安全认证
web.xml 中
```xml
<filter>
    <filter-name>basicAuthenticationFilter</filter-name>
    <filter-class>com.dapeng.cloud.support.web.BasicAuthenticationFilter</filter-class>
    <init-param>
      <param-name>username</param-name>
      <param-value>xxxx</param-value>
    </init-param>
    <init-param>
      <param-name>password</param-name>
      <param-value>xxxx</param-value>
    </init-param>
  </filter>
  <filter-mapping>
    <filter-name>basicAuthenticationFilter</filter-name>
    <servlet-name>HystrixMetricsStreamServlet</servlet-name>
  </filter-mapping>
```
`BasicAuthenticationFilter` 源码:
```java
package com.dapeng.cloud.support.web;

import java.io.IOException;
import java.io.UnsupportedEncodingException;
import java.util.StringTokenizer;
import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.apache.commons.codec.binary.Base64;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.util.StringUtils;

public class BasicAuthenticationFilter implements Filter {
  private static final Logger LOGGER = LoggerFactory.getLogger(BasicAuthenticationFilter.class);
  private String username = "";
  private String password = "";
  private String realm = "Protected";

  public BasicAuthenticationFilter() {
  }

  public void init(FilterConfig filterConfig) throws ServletException {
    this.username = filterConfig.getInitParameter("username");
    this.password = filterConfig.getInitParameter("password");
    String paramRealm = filterConfig.getInitParameter("realm");
    if (StringUtils.hasText(paramRealm)) {
      this.realm = paramRealm;
    }

  }

  public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
    HttpServletRequest request = (HttpServletRequest)servletRequest;
    HttpServletResponse response = (HttpServletResponse)servletResponse;
    String authHeader = request.getHeader("Authorization");
    if (authHeader != null) {
      StringTokenizer st = new StringTokenizer(authHeader);
      if (st.hasMoreTokens()) {
        String basic = st.nextToken();
        if (basic.equalsIgnoreCase("Basic")) {
          try {
            String credentials = new String(Base64.decodeBase64(st.nextToken()), "UTF-8");
            LOGGER.debug("Credentials: " + credentials);
            int p = credentials.indexOf(":");
            if (p != -1) {
              String _username = credentials.substring(0, p).trim();
              String _password = credentials.substring(p + 1).trim();
              if (this.username.equals(_username) && this.password.equals(_password)) {
                filterChain.doFilter(servletRequest, servletResponse);
              } else {
                this.unauthorized(response, "Bad credentials");
              }
            } else {
              this.unauthorized(response, "Invalid authentication token");
            }
          } catch (UnsupportedEncodingException var13) {
            throw new Error("Couldn't retrieve authentication", var13);
          }
        }
      }
    } else {
      this.unauthorized(response);
    }

  }

  public void destroy() {
  }

  private void unauthorized(HttpServletResponse response, String message) throws IOException {
    response.setHeader("WWW-Authenticate", "Basic realm=\"" + this.realm + "\"");
    response.sendError(401, message);
  }

  private void unauthorized(HttpServletResponse response) throws IOException {
    this.unauthorized(response, "Unauthorized");
  }
}

```