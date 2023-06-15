---
title:  log_reduce_predict_full_fl()
description: This article describes the log_reduce_predict_full_fl() user-defined function in Azure Data Explorer.
ms.reviewer: adieldar
ms.topic: reference
ms.date: 05/07/2023
zone_pivot_group_filename: data-explorer/zone-pivot-groups.json
zone_pivot_groups: kql-flavors-all
---
# log_reduce_predict_full_fl()

::: zone pivot="azuredataexplorer"

The function `log_reduce_predict_full_fl()` parses semi structured textual columns, such as log lines, and for each line it matches the respective pattern from a pretrained model or reports an anomaly if no matching pattern was found. The patterns are retrieved from a pretrained model, generated by `log_reduce_train_fl()`. The function is similar to [log_reduce_predict_fl()](log-reduce-predict-fl.md), but unlike log_reduce_predict_fl() that outputs a patterns summary table, this function outputs a full table containing the pattern and parameters per each line.

## Prerequisites

* The Python plugin must be [enabled on the cluster](../query/pythonplugin.md#enable-the-plugin). This is required for the inline Python used in the function.

## Syntax
*T* `|` `invoke` `log_reduce_predict_full_fl(`*models_tbl*`,` *model_name*`,` *reduce_col*`,` *pattern_col*`,` *parameters_col* [`,` *anomaly_str* ]`)`

## Parameters

| Name | Type | Required | Description |
|--|--|--|--|
| *models_tbl* | table | &check; | A table  containing models generated by [log_reduce_train_fl()](log-reduce-train-fl.md). The table's schema should be (name:string, timestamp: datetime, model:string).  |
| *model_name* | string | &check; | The name of the model that will be retrieved from *models_tbl*. If the table contains few models matching the model name, the latest one is used. |
| *reduce_col* | string | &check; | The name of the string column the function is applied to. |
| *pattern_col* | string | &check; | The name of the string column to populate the pattern. |
| *parameters_col* | string | &check; | The name of the string column to populate the pattern's parameters. |
| *anomaly_str* | string | | This string is output for lines that have no matched pattern in the model. Default value is "ANOMALY". |

## Function definition

You can define the function by either embedding its code as a query-defined function, or creating it as a stored function in your database, as follows:

### [Query-defined](#tab/query-defined)

Define the function using the following [let statement](../query/letstatement.md). No permissions are required.

> [!IMPORTANT]
> A [let statement](../query/letstatement.md) can't run on its own. It must be followed by a [tabular expression statement](../query/tabularexpressionstatements.md). To run a working example of `log_reduce_fl()`, see [Example](#example).

~~~kusto
let log_reduce_predict_full_fl=(tbl:(*), models_tbl: (name:string, timestamp: datetime, model:string), 
                           model_name:string, reduce_col:string, pattern_col:string, parameters_col:string, 
                           anomaly_str: string = 'ANOMALY')
{
    let model_str = toscalar(models_tbl | where name == model_name | top 1 by timestamp desc | project model);
    let kwargs = bag_pack('logs_col', reduce_col, 'output_patterns_col', pattern_col,'output_parameters_col', 
                          parameters_col, 'model', model_str, 'anomaly_str', anomaly_str, 'output_type', 'full');
    let code = ```if 1:
        from log_cluster import log_reduce_predict
        result = log_reduce_predict.log_reduce_predict(df, kargs)
    ```;
    tbl
    | evaluate hint.distribution=per_node python(typeof(*), code, kwargs)
};
// Write your query to use the function here.
~~~

### [Stored](#tab/stored)

Define the stored function once using the following [`.create function`](../management/create-function.md). [Database User permissions](../management/access-control/role-based-access-control.md) are required.

> [!IMPORTANT]
> You must run this code to create the function before you can use the function as shown in the [Example](#example).

~~~kusto
.create-or-alter function with (folder = 'Packages\\Text', docstring = 'Apply a trained model to find common patterns in textual logs, output a full table')
log_reduce_predict_full_fl(tbl:(*), models_tbl: (name:string, timestamp: datetime, model:string), 
                           model_name:string, reduce_col:string, pattern_col:string, parameters_col:string, 
                           anomaly_str: string = 'ANOMALY')
{
    let model_str = toscalar(models_tbl | where name == model_name | top 1 by timestamp desc | project model);
    let kwargs = bag_pack('logs_col', reduce_col, 'output_patterns_col', pattern_col,'output_parameters_col', 
                          parameters_col, 'model', model_str, 'anomaly_str', anomaly_str, 'output_type', 'full');
    let code = ```if 1:
        from log_cluster import log_reduce_predict
        result = log_reduce_predict.log_reduce_predict(df, kargs)
    ```;
    tbl
    | evaluate hint.distribution=per_node python(typeof(*), code, kwargs)
}
~~~

---

## Example

The following example uses the [invoke operator](../query/invokeoperator.md) to run the function.

### [Query-defined](#tab/query-defined)

To use a query-defined function, invoke it after the embedded function definition.

~~~kusto
let log_reduce_predict_full_fl=(tbl:(*), models_tbl: (name:string, timestamp: datetime, model:string), 
                           model_name:string, reduce_col:string, pattern_col:string, parameters_col:string, 
                           anomaly_str: string = 'ANOMALY')
{
    let model_str = toscalar(models_tbl | where name == model_name | top 1 by timestamp desc | project model);
    let kwargs = bag_pack('logs_col', reduce_col, 'output_patterns_col', pattern_col,'output_parameters_col', 
                          parameters_col, 'model', model_str, 'anomaly_str', anomaly_str, 'output_type', 'full');
    let code = ```if 1:
        from log_cluster import log_reduce_predict
        result = log_reduce_predict.log_reduce_predict(df, kargs)
    ```;
    tbl
    | evaluate hint.distribution=per_node python(typeof(*), code, kwargs)
};
HDFS_log_100k
| extend Patterns='', Parameters=''
| take 10
| invoke log_reduce_predict_full_fl(models_tbl=ML_Models, model_name="HDFS_100K", reduce_col="data", pattern_col="Patterns", parameters_col="Parameters")
~~~

### [Stored](#tab/stored)

> [!IMPORTANT]
> For this example to run successfully, you must first run the [Function definition](#function-definition) code to store the function.

```kusto
HDFS_log_100k
| extend Patterns='', Parameters=''
| take 10
| invoke log_reduce_predict_full_fl(models_tbl=ML_Models, model_name="HDFS_100K", reduce_col="data", pattern_col="Patterns", parameters_col="Parameters")
```

---

**Output**

| data | Patterns | Parameters |
|--|--|--|
| 081110 | 215858 | 15485 INFO dfs.DataNode$PacketResponder: Received block blk_5080254298708411681 of size 67108864 from /10.251.43.21  081110 \<NUM> \<NUM> INFO dfs.DataNode$PacketResponder: Received block blk_\<NUM> of size \<NUM> from \<IP>  {"parameter_0": "215858", "parameter_1": "15485", "parameter_2": "5080254298708411681", "parameter_3": "67108864", "parameter_4": "/10.251.43.21"} |
| 081110 | 215858 | 15494 INFO dfs.DataNode$DataXceiver: Receiving block blk_-7037346755429293022 src: /10.251.43.21:45933 dest: /10.251.43.21:50010  081110 \<NUM> \<NUM> INFO dfs.DataNode$DataXceiver: Receiving block blk_\<NUM> src: \<IP> dest: \<IP>  {"parameter_0": "215858", "parameter_1": "15494", "parameter_2": "-7037346755429293022", "parameter_3": "/10.251.43.21:45933", "parameter_4": "/10.251.43.21:50010"} |
| 081110 | 215858 | 15496 INFO dfs.DataNode$PacketResponder: PacketResponder 2 for block blk_-7746692545918257727 terminating  081110 \<NUM> \<NUM> INFO dfs.DataNode$PacketResponder: PacketResponder \<NUM> for block blk_\<NUM> terminating  {"parameter_0": "215858", "parameter_1": "15496", "parameter_2": "2", "parameter_3": "-7746692545918257727"} |
| 081110 | 215858 | 15496 INFO dfs.DataNode$PacketResponder: Received block blk_-7746692545918257727 of size 67108864 from /10.251.107.227  081110 \<NUM> \<NUM> INFO dfs.DataNode$PacketResponder: Received block blk_\<NUM> of size \<NUM> from \<IP>  {"parameter_0": "215858", "parameter_1": "15496", "parameter_2": "-7746692545918257727", "parameter_3": "67108864", "parameter_4": "/10.251.107.227"} |
| 081110 | 215858 | 15511 INFO dfs.DataNode$DataXceiver: Receiving block blk_-8578644687709935034 src: /10.251.107.227:39600 dest: /10.251.107.227:50010  081110 \<NUM> \<NUM> INFO dfs.DataNode$DataXceiver: Receiving block blk_\<NUM> src: \<IP> dest: \<IP>  {"parameter_0": "215858", "parameter_1": "15511", "parameter_2": "-8578644687709935034", "parameter_3": "/10.251.107.227:39600", "parameter_4": "/10.251.107.227:50010"} |
| 081110 | 215858 | 15514 INFO dfs.DataNode$DataXceiver: Receiving block blk_722881101738646364 src: /10.251.75.79:58213 dest: /10.251.75.79:50010  081110 \<NUM> \<NUM> INFO dfs.DataNode$DataXceiver: Receiving block blk_\<NUM> src: \<IP> dest: \<IP>  {"parameter_0": "215858", "parameter_1": "15514", "parameter_2": "722881101738646364", "parameter_3": "/10.251.75.79:58213", "parameter_4": "/10.251.75.79:50010"} |
| 081110 | 215858 | 15517 INFO dfs.DataNode$PacketResponder: PacketResponder 2 for block blk_-7110736255599716271 terminating  081110 \<NUM> \<NUM> INFO dfs.DataNode$PacketResponder: PacketResponder \<NUM> for block blk_\<NUM> terminating  {"parameter_0": "215858", "parameter_1": "15517", "parameter_2": "2", "parameter_3": "-7110736255599716271"} |
| 081110 | 215858 | 15517 INFO dfs.DataNode$PacketResponder: Received block blk_-7110736255599716271 of size 67108864 from /10.251.42.246  081110 \<NUM> \<NUM> INFO dfs.DataNode$PacketResponder: Received block blk_\<NUM> of size \<NUM> from \<IP>  {"parameter_0": "215858", "parameter_1": "15517", "parameter_2": "-7110736255599716271", "parameter_3": "67108864", "parameter_4": "/10.251.42.246"} |
| 081110 | 215858 | 15533 INFO dfs.DataNode$DataXceiver: Receiving block blk_7257432994295824826 src: /10.251.26.8:41803 dest: /10.251.26.8:50010  081110 \<NUM> \<NUM> INFO dfs.DataNode$DataXceiver: Receiving block blk_\<NUM> src: \<IP> dest: \<IP>  {"parameter_0": "215858", "parameter_1": "15533", "parameter_2": "7257432994295824826", "parameter_3": "/10.251.26.8:41803", "parameter_4": "/10.251.26.8:50010"} |
| 081110 | 215858 | 15533 INFO dfs.DataNode$DataXceiver: Receiving block blk_-7771332301119265281 src: /10.251.43.210:34258 dest: /10.251.43.210:50010  081110 \<NUM> \<NUM> INFO dfs.DataNode$DataXceiver: Receiving block blk_\<NUM> src: \<IP> dest: \<IP>  {"parameter_0": "215858", "parameter_1": "15533", "parameter_2": "-7771332301119265281", "parameter_3": "/10.251.43.210:34258", "parameter_4": "/10.251.43.210:50010"} |

::: zone-end

::: zone pivot="azuremonitor, fabric"

This feature isn't supported.

::: zone-end