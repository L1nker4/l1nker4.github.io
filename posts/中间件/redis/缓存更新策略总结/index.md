# 缓存更新策略总结




## Cache Aside

具体逻辑如下：

- 写策略：更DB数据，再删除cache数据
- 读策略：
  - cache hit：直接返回命中数据。
  - cache miss：从数据库中读取，然后将数据写入cache，并返回。



适用场景：

- 适合读多写少的场景，不适合写多的场景。



## Read/Write Through

具体逻辑如下：

- Read Through：写入时先查询cache是否hit，hit直接返回，miss则有cache组件负责去DB查，并写入cache，再返回。
- Write Through：更新先查询cache是否hit，hit则更新缓存中数据，然后有cache组件更新到DB，完成后通知更新完成。若miss则直接更新数据库



## Write Back

具体逻辑如下：

- 只更新缓存，立即返回，持久层更新采用异步更新方式。



广泛用于OS，比如CPU Cache、文件系统Cache，数据非强一致性，存在丢数据的问题。

适用场景：写多场景

---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/redis/%E7%BC%93%E5%AD%98%E6%9B%B4%E6%96%B0%E7%AD%96%E7%95%A5%E6%80%BB%E7%BB%93/  

