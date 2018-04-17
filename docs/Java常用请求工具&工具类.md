
### 1.UrlConnection
```
常用于单个请求，接口请求
```

### 2.HttpClient
```
可用于请求/模仿浏览器请求（如可设置301/302自动跳转等）
```

### 3.selenium浏览器内核驱动（支持多语言调用）
```
可用于模仿浏览器行为，还可提供自动弹出浏览器调试
https://github.com/SeleniumHQ/selenium
需要下载对应历览器驱动如：chromedriver.exe 放到windows/system32目录下或者在java中设置系统路径：System.setProperty("webdriver.chrome.driver","路径\.exe")
注意驱动版本和浏览器版本一定要对应
```
