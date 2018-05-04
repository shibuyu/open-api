

# 车智汇车联网开放数据车辆事件


 
## 1.	概述
合作方可以接收车辆事件，在自有的APP做推送或提示个性化的需求开发。
## 2.	接收车辆事件
### 2.1.	前提
- 合作方申请车智汇开放平台的openid
- 双方协商一个用于验证数据的密文
- 合作方提供一个接收事件的http(建议使用https)接口，采用post方式，内容为json，url并带一个签名参数versign
### 2.2.	传输格式
采用JSON格式传输事件内容，内容说明


| **字段名称** | **类型** | **必须** | **说明**                   |
| --------     | ------ | ------ | ------------------------ |
|records|Array|否|提醒事件数组，按唯一编号从小到大排序|
|records[].id|Int64|是|提醒事件的唯一编号|
|records[].esn|String|是|发生提醒事件的终端编号|
|records[].uid|String|是|关联的用户（或用户车辆）唯一编号|
|records[].type|String|是|事件类型|
|records[].time|Int32|是|发生时间，Unix时间戳，精确到秒|
|records[].location|Array|否|发生地点坐标，采用GCJ-02坐标系：[经度,纬度]|
|records[].text|String|否|提醒描述|
|records[].attrs|Object|否|其它属性，内置JSON格式|


### 2.3.	数据范例
    {
    "records":[{
    	"id":123,
    	"esn":"351234567812345",
    	"uid":"1424587",
    	"type":"震动",
    	"time":1487589471,
    	"text":"[粤A12345] 于 [2017-06-16 13:16] 在 [广东省广州市天河区龙口东路]附近发生轻微震动(可能有人轻微触碰,例如拉门把),请留意.",
    	"location":[113.1452,
    			23.41254
    	],
    	"attrs":{
    	"level":1
    	}
    },
    {
    	"id":124,
    	"esn":"351234567812345",
    	"uid":"1424587",
    	"type":"点火",
    	"time":1487589471,
    	"text":"[粤A12345] 于 [2017-06-16 13:16:07] 启动点火",
    	"location":[113.1452,
    			23.41254
    	]
    	}]
    }   
 
### 2.4.	详细流程
![](https://github.com/iauto360/open-api/blob/master/vehicle_events_sequence_chart.png)
 
1.  从各方面[终端上报/违章查询]监听车辆事件
2.  监听到车辆相关事件
3.  把事件组装为传输的JSON数据格式字符串events
4.  通过sha1算法，把该字符串与前提中第2点的验证数据的密文作签名得到verSign
  
        //Java范例
        //16进制字符映射数组
        char[] HEX_DIGITS = {'0', '1', '2', '3', '4', '5', '6', '7', '8', '9','a'};
        //事件数组JSON字符串
        String events="{\records\":[{\"id\":123,\"esn\":\"35124...";
        //双方协商用于一个验证数据的密文
        String secureCode = "Ax7scb224xf8z";

        //拼接字符串
        String signRawStr=events+secureCode;
        //采用SHA1算法hash字符串
        MessageDigest messageDigest = MessageDigest.getInstance("SHA1");
        messageDigest.update(signRawStr.getBytes());
        byte[] bytes = messageDigest.digest();

        //把byte转为可视的16进制字符串
        int len = bytes.length;
        StringBuilder builder = new StringBuilder(len * 2);
        for(int j=0;j<len;j++){
            builder.append(HEX_DIGITS[(bytes[j] >4) & 0x0f]);
            builder.append(HEX_DIGITS[(bytes[j]) & 0x0f]);
        }
        String verSign = builder.toString();   

	

5.  POST到前提中第3点合作方提供的http接口，并在参数中增加一个参数versign，值为		上一步得到hash签名
6. 【合作方】获取POST过来的JSON字符串数据
7. 【合作方】与第4步一样，得到verSign
8. 【合作方】把上一步得到的verSign与url参数传过来的versign进行对比，如果一致，表		示该消息是由车智汇平台推送，如果不一致，则可能是恶意数据
9. 【合作方】实现自有逻辑，比如做APP推送等

### 2.5.	事件类型


| **类型** | **说明** | **text内容** | **attrs内容**|
| --------     | ------ | ------ | ------------------------ |
|震动|震动发生在非行驶阶段，按敏感度分为[高/中/低]三级，对应[较大/异常/轻微]强度|【#车牌号码#】于【#时间#】 在 【#地点#】 附近发生#强度#震动，请留意。|level:敏感度，1-高，2-中，3-低|
|拔出|终端被拔出，8小时内多次拔出，只提醒第一次|智能盒已从【#车牌号码#】拔出，爱车将失去保护！||
|点火|点火时触发，如果因网络问题等原因，超5分钟不作提醒|请注意！【#车牌号码#】于【#点火时间#】启动点火|status:车辆状态，1-无异常，2-有异常|
|熄火|熄火时触发，如果因网络问题等原因，超5分钟不作提醒|【#车牌号码#】于【#熄火时间#】停车熄火||
|越界|驾驶出电子围栏外时触发，10分只提醒一次|请注意！【#车牌号码#】驾驶出电子围栏#距离#米，当前地址【#当前地址#"】||
|碰撞|碰撞发生在行驶阶段，按敏感度分为[高/低]两级，对应[轻微/强烈]强度|【#车牌号码#】于【#时间#】 在 【#地点#】 附近疑似发生#强度#碰撞|level:敏感度，1-高，2-低|
|违章|收集到新的违章信息时推送|您有#违章记录数#条新的违章记录，点击查看>||
|故障|出现故障码后推送|【#车牌号码#】产生了#故障#项故障，请注意！||
|电瓶电压|电瓶电压分两档警告阀值提醒|【#车牌号码#】电瓶电压低于【#警告阀值#】V，可能会导致汽车无法打火，请及时检查。【#车牌号码#】电瓶电压低于【#警告阀值#】V，电量偏低，打开车智汇可以随时监控您的汽车电瓶状况。|level:1-第一档，偏低，2-第二档，过低|
|胎压|胎压版胎压出现异常时推送，异常情况：可能正在漏气/胎压异常/胎压偏低/胎温异常/胎压传感器电量低|#异常#(#轮胎位置#)，请及时检查。|type:1-可能正在漏气，2-胎压异常，3-胎压偏低，4-胎温异常,5-胎压传感器电量低|
|保养|将达到保养里程时熄火后推送|【#车牌号码#】还有【#剩余公里#】公里就要保养了。||
|开锁|远程控制车门开锁操作响应事件，状态：成功/失败|【#车牌号码#】于【#开锁时间#】在 【#地点#】 附近开锁【#状态#】||
|关锁|远程控制车门关锁操作响应事件，状态：成功/失败|【#车牌号码#】于【#关锁时间#】在 【#地点#】 附近关锁【#状态#】||
|流量超出|流量使用超出时推送|您的流量已使用【#使用量#】MB，将关闭WIFI功能。|use：使用量|
|禁入地界|驾驶进禁入地界时触发，10分只提醒一次|请注意！【#车牌号码#】驾驶进禁入地界【#禁止地界名#】|lng: 当前车辆经度 lat: 当前车辆纬度|



