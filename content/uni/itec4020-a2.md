---
title: "Secured Web Information System Prototype"
slug: "Secured Web Information System Prototype"
date: 2023-05-07T02:26:29-04:00
tags: ["Java", "Servlet", "JSP", "Tomcat", "HTML","CSS","JavaScript"]
categories: ["Web Development"]

draft: false
---

View the full project here: [Github Repo](https://github.com/sky-haihai/4020A2-Secured-Web-IS.git)

## Introduction

In a recent academic project, I implemented a secured web information system using Java Servlet and created several content pages including AboutMe, Skills, and Contact pages for testing my system.

The goal was to implement a web application that could allow users to login, logout and view their profile. The application also had to be secured against unauthorized access such as directly typing the content page URL.

The URL of the deployed website on the school SIT server is:
http://sit.itec.yorku.ca:8080/itec4020grp9/

> :warning: **Note**  
The website is deployed on the school System Integration Testing(SIT) server. Access it requires a York University VPN. Please contact me if you are interested and I will send you a video demo of the website.

## Environment

### Server

- Tomcat 7.0.76
- JavaSE 8

### Local

- JavaSE 8 :warning: Must match the version of the server side
- VS Code (with the following extensions)
  - Language Support for Java(TM) by Red Hat `v1.18.0`
  - Project Manager for Java `v0.22.0`

## Structure Overview

```
website
├── css
│       ├── preventPrint.css
│       ├── useDefaultCursor.css
│       └── ...
├── images // images that do not need to be protected
├── js
│       ├── disableDragging.js
│       ├── disableSavingImg.js
│       └── disableSelection.js
├── src
│       ├── util
│       │   └── ImageHelper.java
│       ├── LoginServlet.java
│       ├── LogoutServlet.java
│       ├── AboutMeServlet.java
│       └── ...
├── WEB-INF
│       ├── classes
│       │       ├── util
│       │       │       ├── ImageHelper.class
│       │       ├── LoginServlet.class
│       │       ├── LogoutServlet.class
│       │       ├── AboutMeServlet.class
│       │       └── ...
│       ├── images // images that need to be protected
│       │       ├── user_icon.png
│       │       └── ...
│       └── lib
│               └── javax.servlet-api-3.0.1.jar
├── index.jsp
├── readme.txt //instructions for setting up the environment
```

> :warning: **Note**    
Servlet api version 3.0.x is supported by Tomcat 7.0.76. If you are using a different version of Tomcat, you may need to change the version of the servlet api jar file.

## Implementation

### User Authentication

#### Login

For simplicity, the login page is implemented inside `index.jsp`. A login form is submitted to the `LoginServlet.class`. when the user clicks the login button.

Since this was a school project, the username and password are hard-coded in the **Controller** layer. In a real-world application, the username and password should be stored in a database and accessed by **Data Access Object(DAO)** layer.

```java
private final String fakeUsername = "admin";
private final String fakePassword = "admin";

String username = request.getParameter("username");
String password = request.getParameter("password");

boolean inputMatches = fakeUserName.equals(username) && fakePassword.equals(password);

if (inputMatches) {
	session.setAttribute("userId",username);
	response.sendRedirect("aboutme");
	return;
}
```

Source [LoginServlet.java](https://github.com/sky-haihai/4020A2-Secured-Web-IS/blob/fc29ffe396baaaaa47062d61970c054b70014d36/website/src/LoginServlet.java)

> :warning: **Note**    
As the user authentication details, such as the username and password, are transmitted through an HTML form, it is recommended to implement the authentication logic in the doPost() method rather than the doGet() method. Otherwise, the user authentication information will be exposed in the URL, causing security issues.

#### Logout

The logout page is much simpler. It only needs to invalidate the session and redirect the user to the login page. This logic can be implemented inside either the doGet() method or the doPost() method.

```java
HttpSession session = request.getSession(false);
if (session != null) {
	session.removeAttribute("userId");
}

response.sendRedirect("index.jsp");
```

Source [Logout.java](https://github.com/sky-haihai/4020A2-Secured-Web-IS/blob/fc29ffe396baaaaa47062d61970c054b70014d36/website/src/LogoutServlet.java)

### Secured Content Pages

For each protected content page, the servlet class needs to check if the user has logged in. If the user has not logged in, the servlet class will redirect the user to the login page.

Check if the user has logged in:

```java
// check if user is logged in
HttpSession session = request.getSession(false);
if (session == null || session.getAttribute("userId") == null) {
    response.sendRedirect("index.jsp");
    return;
}

```

Disable Caching:

```java
response.setHeader("Cache-Control", "no-cache, no-store, must-revalidate");
response.setHeader("Pragma", "no-cache");
response.setDateHeader("Expires", 0);
```

Actually writing the content of the page:

```java
response.setContentType("text/html");
PrintWriter out = response.getWriter();
out.println("<html>");

out.println("<head>");
out.println("<title>AboutMe</title>");
out.println("<link rel=\"stylesheet\" href=\"css/reset.css\">");
out.println("<link rel=\"stylesheet\" href=\"css/useDefaultCursor.css\">");
out.println("<link rel=\"stylesheet\" href=\"css/preventPrint.css\">");
out.println("<link rel=\"stylesheet\" href=\"css/credit.css\">");
out.println("<link rel=\"stylesheet\" href=\"css/profile.css\">");
out.println("<link rel=\"stylesheet\" href=\"css/aboutme.css\">");

out.println("</head>");

out.println("<body>");

...

out.println("</body>");

out.println("<script src='js/disableSelection.js'></script>");
out.println("<script src='js/disableSavingImg.js'></script>");
out.println("<script src='js/disableDragging.js'></script>");

out.println("</html>");
```

Source [AboutMeServlet.java](https://github.com/sky-haihai/4020A2-Secured-Web-IS/blob/fc29ffe396baaaaa47062d61970c054b70014d36/website/src/AboutMeServlet.java)

### Image Protection

The images that need to be protected are put into the `WEB-INF` folder. `WEB-INF` folder is a special folder which is protected by web application containers like Tomcat. Anything inside will not be accessable directly through a URL such as `https://www.example.com/WEB-INF/images/user_icon.png`

To read the image file, pass the relative path of the image file to the Servlet API:

```java
// path: "/WEB-INF/images/user_icon.png"
InputStream inputStream = context.getResourceAsStream(path);
```

Finally read the Input Stream and convert the image into data URI schemes to completely hide the image URL:

```java
imageStr = Base64.getEncoder().encodeToString(imageData);
```

Source [ImageHelper.java](https://github.com/sky-haihai/4020A2-Secured-Web-IS/blob/fc29ffe396baaaaa47062d61970c054b70014d36/website/src/util/ImageHelper.java)

Then include the URI schemes in the HTML:

```java
out.println("<img src='data:image/png;base64," + imageStr + "' />");
```

Source [AboutMeServlet.java](https://github.com/sky-haihai/4020A2-Secured-Web-IS/blob/fc29ffe396baaaaa47062d61970c054b70014d36/website/src/AboutMeServlet.java)

### Preventing Unwanted User Actions

To prevent users from saving images on the web pages:

```javascript
document.addEventListener(
  "contextmenu",
  function (e) {
    e.preventDefault();
  },
  false
);
```

Source [disableSavingImg.js](https://github.com/sky-haihai/4020A2-Secured-Web-IS/blob/fc29ffe396baaaaa47062d61970c054b70014d36/website/js/disableSavingImg.js)

To prevent users from selecting text on the web pages:

```javascript
if (typeof document.onselectstart != "undefined") {
  document.onselectstart = new Function("return false");
} else {
  document.onmousedown = function () {
    return false;
  };
  document.onmouseup = function () {
    return true;
  };
}
```

Source [disableSelection.js](https://github.com/sky-haihai/4020A2-Secured-Web-IS/blob/fc29ffe396baaaaa47062d61970c054b70014d36/website/js/disableSelection.js)

To prevent users from dragging images on the web pages:

```javascript
document.addEventListener("dragstart", function (event) {
  event.preventDefault();
});
```

Source [disableDragging.js](https://github.com/sky-haihai/4020A2-Secured-Web-IS/blob/fc29ffe396baaaaa47062d61970c054b70014d36/website/js/disableDragging.js)

To prevent users from printing the web pages:

```css
@media print {
  body * {
    visibility: hidden;
  }
}
```

Source [preventPrint.css](https://github.com/sky-haihai/4020A2-Secured-Web-IS/blob/fc29ffe396baaaaa47062d61970c054b70014d36/website/css/preventPrint.css)

## Deployment

### Step 0: Prerequisites (Done by the server administrator)

1. Install [JavaSE 8](https://www.oracle.com/java/technologies/javase/javase-jdk8-downloads.html) on the server machine.
2. Install [Apache Tomcat 7.0.76](https://archive.apache.org/dist/tomcat/tomcat-7/v7.0.76/bin/) on the server machine.

### Step 1: Download SSH/FTP client

For windows users:

- SSH client: [Putty](https://www.putty.org/)
- FTP client: [WinSCP](https://winscp.net/eng/download.php)

### Step 2: Upload the website folder to the server

1. tag every servlet class with `@WebServlet("/path")` annotation.

```
@WebServlet("/aboutme")
public class AboutMeServlet extends HttpServlet {
    ...
}
```

path is the URL path to access the servlet. For example, the above servlet can be accessed by the URL: `http://localhost:8080/itec4020grp9/aboutme`

2. alternitively, add the following code to the `web.xml` file to map the servlet to a URL path.

The `web.xml` file should be created in the `WEB-INF` folder like this:

```xml
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
    version="3.0">

    <servlet>
        <servlet-name>AboutMeServlet</servlet-name>
        <servlet-class>AboutMeServlet</servlet-class>
    </servlet>

    ...
</web-app>
```

To include Servlets to the `web.xml` file:

```xml
<servlet>
    <servlet-name>MyServlet</servlet-name>
    <servlet-class>/path/to/myServlet</servlet-class>
</servlet>
```

To include JSP files to the `web.xml` file:

```xml
<servlet>
  <servlet-name>MyJsp</servlet-name>
  <jsp-file>/path/to/myJsp</jsp-file>
</servlet>
```

**Note**

1. The path does not need to include the file extension.
2. The path is relative to the `WEB-INF/classes` folder.
3. make sure all Java Servlet classes are compiled into `WEB-INF/classes` folder.

### Step 3: Deploy the website on the Tomcat server

1. To deploy the website on the Tomcat server, simply copy the website folder to the `webapps` folder of the Tomcat server.

### Step 4: Making changes to the website

1. Upload the changes to the server.
2. Restart the Tomcat server. **Not Practical** if you are not the server administrator.
3. Alternatively, slightly changing the `web.xml` file back and forth will trigger a kind of reload event inside Tomcat. **Not Elegant** but Practical:kissing_face:

## Test Cases

### Case 1

A user cannot get into the secured Web information system after logout by clicking on the browser's back button or typing in the URL address directly.

### Case 2

The user has to login using userid and password in order to get into the secured website after logout.

### Case 3

All the URL addresses of the secured Web information system need to be protected.

### Case 4

The image files of secured Web pages shouldn’t be allowed to save on the client’s local machine.

### Case 5

After closing the browser, the user cannot get into the secured Web information system without login again.

### Case 6

Users cannot copy and paste the text information on the secured Web pages.

### Case 7

Some secured pages should not be able to be printed, for example, by using a browser’s print function.

### Case 8

Information on the server side(e.g. Image URL) should be protected.

## Future Improvements

1. Use a database to store user information instead of hard-coding the user information in the Java Servlet code.
2. Use `<canvas>` tags on html to display the secured content pages instead of disabling user actions on the web pages. Because disabling user actions may have a negative impact on the overall user experience and the logic is handled on the client-side which is not secure.

## Conclusion

In conclusion, I have successfully implemented a secured web information system. The system is secured by using HTTPS, Tomcat and Java Servlet. The system is protected from unauthorized user accesses and also protected from unwanted user actions such as saving images, selecting text, dragging images, and printing the web pages.
