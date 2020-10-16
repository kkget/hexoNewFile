---
title: Thymleaf项目常用操作
date: 2020-06-02 11:45:38
tags: Thymleaf
---

Thymleaf搭配Springboot完成页面渲染，整理下日常开发中常见常用操作
1.下拉框动态被选中
```java
   <label for="inputLevel"
   class="col-6 col-form-label form-label">用户等级:</label>
   <div class="col-6">
   <select id="inputLevel" class="form-control" name="level">
   <option value=""
   th:selected="${null==req?'selected':req.level==null?'select':'false'}">
   全部
   </option>
   <option value="1"
   th:selected="${null==req?'false':req.level=='1'?'select':'false'}">1级
   </option>
   <option value="2"
   th:selected="${null==req?'false':req.level=='2'?'select':'false'}">2级
   </option>
   </select>
   </div>
   </div>
```
2.动态复选框选中(判断List是否包含)
```java
<input type="checkbox"
  th:id="'checkboxResource' + ${resourceEn.key}"
  th:checked="${null==req.resourceIdList?'false':#arrays.contains(req.resourceIdList, #strings.toString(resourceEn.key))?'checked':'false'}"
  name="resourceIdList" th:value="${resourceEn.key}">
  <label th:text="${resourceEn.value.name}" th:for="'checkboxResource' + ${resourceEn.key}"></label>
```
3.onclick动态传值
```java
<button type="button" th:text="*{status}==0?'开启':'关闭'"
th:attr="disabled=*{status==10?true:false}"
th:data-id="${supplier.id}"
th:data-status="*{status==0?1:0}"
th:class="*{status ==0||status!=1}?'btn btn-block btn-success':'btn btn-block btn-danger'"
onclick="enable(this.getAttribute('data-id'),this.getAttribute('data-status'))"></button>
```
4.日期格式化
```java
<td th:text="*{#dates.format(updateTime, 'yyyy-MM-dd HH:mm:ss')}">
```
5.保留小数点后两位
```java
<label class="ml-3" th:if="*{price ne 1.0}" th:text="*{#numbers.formatDecimal(price * 10,0,2)}"></label>
```
6.点击详情/编辑回显下拉被选中
```java
<select class="form-control select2bs4" style="width: 100%;" name="id">
<option value="" selected="selected">==请选择==</option>
<option th:each="user : ${users}" th:selected="${user.id eq dept.id}"  th:text="${user.Name}"></option></select>
```