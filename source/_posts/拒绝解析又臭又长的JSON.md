---
title: 拒绝解析又臭又长的JSON
date: 2020-06-01 16:48:28
tags:
---


author:赵KK

 在日常工作中，不管是因为接收前端返回约定格式的JSON字符串，还是因为需要约定格式请求第三方服务，或者需要将前端画像xml解析成JSON，再或者需要接入第三方短信，供应商，数据提供商的JSON数据，或是需要提供对外暴露接口的API，可见解析JSON是一个常见操作。

JSON是一个轻量级的数据交换格式。

一：表单数据由数据库实体对象接收

常见的前后端约定字段，指定字段名称后，由数据库实体接收序列化后的表单数据，无序解析。

二：JSONObject解析

前后端约定格式，实体接收String类型，通过JSONObject解析JSON，JSONArray等操作

例如：
```
[
    {
        "children": [
            
        ],
        "createBy": "admin",
        "createTime": 1521171180000,
        "icon": "fa fa-gear",
        "menuId": 1,
        "menuName": "系统管理",
        "menuType": "M",
        "orderNum": "1",
        "params": {
            
        },
        "parentId": 0,
        "perms": "",
        "target": "",
        "url": "#",
        "visible": "0"
    },
    {
        "children": [
            
        ],
        "createBy": "admin",
        "createTime": 1521171180000,
        "icon": "fa fa-video-camera",
        "menuId": 2,
        "menuName": "系统监控",
        "menuType": "M",
        "orderNum": "2",
        "params": {
            
        },
        "parentId": 0,
        "perms": "",
        "target": "",
        "url": "#",
        "visible": "0"
    },
    {
        "children": [
            
        ],
        "createBy": "admin",
        "createTime": 1521171180000,
        "icon": "fa fa-bars",
        "menuId": 3,
        "menuName": "系统工具",
        "menuType": "M",
        "orderNum": "3",
        "params": {
            
        },
        "parentId": 0,
        "perms": "",
        "target": "",
        "url": "#",
        "visible": "0"
    }
]
```
通过JSONObject以及解析JSONArray获取
三：接入第三方API

接入第三方API，或者按约定调用第三方服务时，你会发现约定了又臭有长的JSON格式，包含特定字段，包含token，包含秘钥，一个详细数据解析接口，上百个字段是常见的，而且多种格式嵌套解析，如果单纯将收到的字符串手动转化成JSONObject，还要判空，还要层层遍历，还要验证数据的有效性，这是在是不小的工作量。

改造方法：提取最长，覆盖字段最全的作为实体列接收，含有List数据就由List接收，最外层K值由字段接收，涉及类型判断需按约定传不同数值的，定义为枚举，秘钥等特殊Key值MD5加解密传递。
```java
// 如果url是空，则认为是解析历史数据 不需要拼装请求
        if (url != null && !"".equals(url)) {
            Client client = new Client();
            Map<String, String> params = new HashMap<String, String>();
            if ("mobileReli".equals(interfaceCode)) { //if类型判断定义为枚举      
                String infoJson = String.format("{\"phone\":\"%s\",\"name\":\"%s\",\"curDate\":\"%s\"}",
                        applyRecord.getPhone(), applyRecord.getName(), applyRecord.getFlashblackDate());
                StringBuffer sb = new StringBuffer();
                long time = System.currentTimeMillis();//重复度高的字段由优特实体类接收
                sb.append(secret + "!" + appKey + "!" + time + "!" + applyRecord.getName() + "!"
                        + applyRecord.getPhone() + "!" + secret + "!");
                sign = hdsClient.md5(sb.toString());
                String param = String.format("appKey=%s&infoJson=%s&sign=%s&time=%s", appKey, infoJson, sign, time);
                try {
                    jsonData = hdsClient.getResult(url, param);
                } catch (IOException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            } else if ("DSModel".equals(interfaceCode)) {
                String infoJson = String.format("{\"mobile\":\"%s\",\"name\":\"%s\",\"starttime\":\"%s\"}",
                        applyRecord.getPhone(), applyRecord.getName(), applyRecord.getFlashblackDate());
                StringBuffer sb = new StringBuffer();
                long time = System.currentTimeMillis();
                sb.append(secret + "!" + appKey + "!" + time + "!" + applyRecord.getName() + "!"
                        + applyRecord.getPhone() + "!" + applyRecord.getFlashblackDate() + "!" + secret + "!");
                sign = hdsClient.md5(sb.toString());
                String param = String.format("appKey=%s&infoJson=%s&sign=%s&time=%s", appKey, infoJson, sign, time);
                try {
                    jsonData = hdsClient.getResult(url, param);
                } catch (IOException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            } else if ("addressDetection".equals(interfaceCode)) {   
                String infoJson = String.format("{\"phone\":\"%s\",\"address\":\"%s\",\"curDate\":\"%s\"}",
                        applyRecord.getPhone(), applyRecord.getAddress(), applyRecord.getFlashblackDate());
                StringBuffer sb = new StringBuffer();
                long time = System.currentTimeMillis();
                sb.append(secret + "!" + appKey + "!" + time + "!" + applyRecord.getPhone() + "!"
                        + applyRecord.getAddress() + "!" + applyRecord.getFlashblackDate() + "!" + secret + "!");
                sign = hdsClient.md5(sb.toString());
                String param = String.format("appKey=%s&infoJson=%s&sign=%s&time=%s", appKey, infoJson, sign, time);
                try {
                    jsonData = hdsClient.getResult(url, param);
                } catch (IOException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
```
```java
public enum InterfaceCodeEnum {
    
    InterfaceCode1(1,"mobileReli"),
    InterfaceCode2(2,"DSModel"),
    InterfaceCode3(3,"addressDetection"),
    .
    .
    .
    ;


    private Integer code;
    private String name;


    InterfaceCodeEnum(Integer code, String name) {
        this.code = code;
        this.name = name;
    }


    public static String getNameBycode(Integer code) {
        if (code == null) {
            return "";
        }


        for (InterfaceCodeEnum a : InterfaceCodeEnum.values()) {
            if (a.code.equals(code)) {
                return a.name;
            }
        }


        return "";
    }
}


@Builder
@Data
public class InterfaceCodeResult implements Serializable {


    //基础信息
    private Base base;
    //秘钥信息
    private AuthInfo authInfo;
    //外层字段封装为对象接收
    private AddressResult  addressResult;
    //重复多层信息List接收
    private List<Flashblack> flashblack;
}
```
当接收到JSON字符串时
```java
InterfaceCodeResult codeResult=JSONObject.parseObject(InterfaceCodeResult.getRequestInfo(),InterfaceCodeResult.class);
if(PreInterfaceStatus.equals(codeResult.base.getTyep())){
  return JavaConvertUtil.conversion(codeResult, CodeParams.class);
}
```
仅需要判断多个类型即可，对应字段会自动解析，当接收又臭又长的XML解析还需要后端验证时，需要封装Util类进行验证调用

同步更新至微信公众号，请搜索:赵KK日常技术记录，不定时更新文章内容