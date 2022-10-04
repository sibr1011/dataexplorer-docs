---
title: .show table data statistics - Azure Data Explorer
description: This article describes the .show table data statistics command in Azure Data Explorer.
ms.reviewer: alexans
ms.topic: reference
ms.date: 07/19/2022
---
# .show table data statistics

Displays table data statistics estimation by table column and storage engine version.

This command requires [Database monitor permission](../management/access-control/role-based-authorization.md).

> [!NOTE]
> This command only returns statistics for data stored in columnar stores. Row store data statistics are not returned.

## Syntax

`.show` `table` *TableName* `data` `statistics` [ `with` `(` *PropertyName* `=` *PropertyValue* `)` ]

## Arguments

* *TableName*: The name of the table on which you want to return statistics.

## Properties

The following properties can be specified. All properties are optional.

|Property name| Description | Property values| 
|--|--|--|
| `samplepercent` | Scans the specified percentage of data. | Integer between 0 and 100 |
| `scope` | Defines if only the hot cache is scanned or if the whole data is scanned.| Specify `"hotcache"` to only scan hot cache data, or `"all"` to scan all the data. |
| `from` | Scans data extents created after this time. | datetime |
| `to` | Scan data extents created before this time. | datetime |

If no options are provided, only 1% of hot cache data is scanned.

## Returns

The command returns a result table that contains the following information.

| Column Name | Type | Description |
|--|--|--|
| `ColumnName` | string   | Table column name. |
| `ColumnType` | string   | Table column data type. |
| `ColumnId` | guid | Table column internal ID. |
| `OriginalSize` | long | Estimation of ingested data size in bytes. |
| `ExtentSize` | long | Estimation of total size of extents (data size + index size) in bytes. |
| `CompressionRatio` | real | Ratio of original data size to extent data size. |
| `DataCompressedSize` | long | Extents data size in bytes, excluding indexes. |
| `SharedIndexSize` | long | For [Engine V3](../../engine-v3.md), text index size in bytes. The text index is shared by all table columns in [Engine V3](../../engine-v3.md). |
| `IndexSize` | long | Size of numeric indexes in bytes. For Engine V2, the value includes text index size as well. |
| `IndexSizePercent` | real | Ratio of the index size to original data size in bytes. |
| `StorageEngineVersion` | string | Engine version. If data exists for multiple engine versions, the output includes an entry for each version. |
| `PresentRowCount`	| long | Total rows count, excluding [deleted](../concepts/data-soft-delete.md) rows. |
| `DeletedRowCount` | long | [Deleted](../concepts/data-soft-delete.md) rows count. |
| `SamplePercent` | real | Actual data sampling percentage, which may differ from the hint provided in `samplepercent` option. |
| `IncludeColdData`	| bool | Whether cold data was taken into account when calculating statistics estimate. |

> [!NOTE]
> For [Engine V3](../../engine-v3.md), `ExtentSize` doesn't include shared text index size. There are two ways to calculate total table data size:
> * Sum all `ExtentSize` values and add the `SharedIndexSize` value.
> * Run the [.show table details](show-table-details-command.md) command, and use the `TotalExtentSize` value.

## Example

```kusto
.show table Users data statistics with (scope="all", samplepercent=80)
```

**Output**

|ColumnName|	ColumnType|	ColumnId|	OriginalSize|	ExtentSize|	CompressionRatio|	DataCompressedSize|	SharedIndexSize|	IndexSize|	IndexSizePercent|	StorageEngineVersion|	PresentRowCount|	DeletedRowCount|	SamplePercent|	IncludeColdData|
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|
|Name|	string|	45c15cfe-ef38-4153-97e7-c18d21651fc2|	60907|	137305|	0.44|	137305|	27787|	0|	0|	V3|	2500|	0|	80|	True|
|Usage|	real|	c0aa6668-0f73-46a1-874e-f1b19123eb61|	20000|	20282|	0.99|	20102|	27787|	180|	0.9|	V3|	2500|	0|	80|	True|
|ID|	long|	eb01bab7-da60-4515-a5d4-4780a827bd85|	20000|	5722|	3.49|	5152|	27787|	570|	2.85|	V3|	2500|	0|	80|	True|