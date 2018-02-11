## 注意问题
spring cache 由于基于 spring AOP 技术，尤其是动态的 proxy 技术，导致其不能很好的支持方法的内部调用或者非 public 方法的缓存设置