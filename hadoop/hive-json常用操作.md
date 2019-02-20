##### 查询json key
```
select get_json_object(line,'$.userInfo.osVersion'),get_json_object(line,'$.click.isActive')
from tmp_ads_feature_click
where t>='20181025' and get_json_object(line,'$.click.channelId')='tencent_rtb'
limit 100;
```
##### 查询json key(多组)
```
select json_tuple(line,'$.userInfo.osVersion','$.click.isActive')
from tmp_ads_feature_click
limit 100;
```
##### get_json_object与json_tuple小结
```
json_tuple相对与get_json_object的区别就是一次可以解析多个json字段
```

##### 查询json 数组第几次
```
select get_json_object(line, '$.click.bucketInfo[3]')
from tmp_ads_feature_click
limit 100;
```
##### 查询json 数组(explode方式)
```
select bucketItem
from tmp_ads_feature_click  
lateral view explode(
  split(regexp_replace(substr(get_json_object(line, '$.click.bucketInfo'),2,length(get_json_object(line, '$.click.bucketInfo'))-2),'},' , '}####'), '####')
) bucketList as bucketItem
limit 10;
```
##### 查询json 数组(UDF方式)

```
package com.abecedarian.udf.json;

import org.apache.hadoop.hive.ql.exec.Description;
import org.apache.hadoop.hive.ql.exec.UDF;
import org.json.JSONArray;
import org.json.JSONException;
import java.util.ArrayList;

@Description(name = "json_array",value = "_FUNC_(array_string) - Convert a string of a JSON-encoded array to a Hive array of strings.")
public class UDFJsonAsArray extends UDF {
    public ArrayList<String> evaluate(String jsonString) {
        if (jsonString == null) {
            return null;
        }
        try {
            JSONArray extractObject = new JSONArray(jsonString);
            ArrayList<String> result = new ArrayList<String>();
            for (int ii = 0; ii < extractObject.length(); ++ii) {
                result.add(extractObject.get(ii).toString());
            }
            return result;
        } catch (JSONException e) {
            return null;
        } catch (NumberFormatException e) {
            return null;
        }
    }
}

将上面的代码进行编译打包,假设打包的名称为json-array.jar,添加到hive-shell中,并创建临时函数json_array:
add jar /home/abecedarian/json-array.jar;
create temporary function json_array as 'com.abecedarian.udf.json.UDFJsonAsArray';

可使用json_array查询json数组:
select json_tuple(json, 'website', 'name') from (select explode(json_array('[{"website":"www.litianle.com","name":"abecedarian"},{"website":"www.baidu.com","name":"baidu"}]')) as json) t;
```
