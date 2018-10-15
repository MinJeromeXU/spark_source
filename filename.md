spark 源码剖析

``` javascript
 map.put("batchNo", batchNo);
        map.put("accountingDate", accountingDate);
        map.put("creator",username);
        map.put("creatorName",nickName);
        String tempPfc = null;
        String tempPfc2 = null;//当取的预定义编码根据ORACLE分类来时，使用这个做备用
        PredefineCodeVO  pc = null;
        PredefineCodeVO  pc2 = null;
```