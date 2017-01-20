---
title: "Understanding Query Optimization Statistics (SQL Server PDW)"
ms.custom: na
ms.date: 07/27/2016
ms.reviewer: na
ms.suite: na
ms.tgt_pltfrm: na
ms.topic: article
ms.assetid: 54fe978e-b2fc-4315-9f5d-88f107aa7429
caps.latest.revision: 20
author: BarbKess
---
# Understanding Query Optimization Statistics (SQL Server PDW)
SQL Server PDW uses statistics to improve the quality of the data movement and Transact\-SQL query plans, and therefore to improve query performance.  
  
## Contents  
  
-   [Statistics Basics](#StatisticsBasics)  
  
-   [Statistics Differences From SQL Server](#StatisticsDifferences)  
  
-   [Recommendations](#StatisticsBestPractices)  
  
## <a name="StatisticsBasics"></a>Statistics Basics  
statistics objects  
Statistics are objects that contain statistical information about the distribution of values in one or more columns of a table. The query optimizer uses these statistics to estimate the cardinality, or number of rows, in the query result. These cardinality estimates enable the query optimizer to create a high-quality query plan.  
  
In SQL Server PDW, statistics can help improve both the plan for moving data and the plans for running SQL Server PDW queries on the Compute nodes.  
  
For example, on the Compute nodes, the SQL Server query optimizer could use cardinality estimates to choose the index seek operator instead of the more resource-intensive index scan operator, and in doing so improve query performance. On the Control node, the dsql query optimizer could use cardinality estimates to choose a shuffle move instead of a broadcast move, and therefore improve query performance.  
  
Compute node statistics  
SQL Server PDW auto-generates statistics on the Compute nodes, and uses them to improve query performance for the local queries that run on the Compute nodes. The Compute node statistics are SMP statistics generated by SQL Server with auto-create and auto-update on. These statistics are not visible to the user through the SQL Server PDW metadata .  
  
For full control over which statistics are generated on the Compute nodes, we recommend creating single column statistics prior to running queries. This enables you to manage the statistics through the statistics DDL statements, and to view the statistics in the system views.  
  
Control node statistics  
The Control node merges the Compute node statistics and stores them as a single statistics object. You can call them MPP statistics or merged statistics. SQL Server PDW uses these MPP statistics to estimate the number of rows that need to be moved among the Compute nodes in order to satisfy the Transact\-SQL query. Good cardinality estimates allow the dsql query optimizer to choose a high quality plan for data movement operations that reduces the data movement and therefore improves query performance.  
  
The Control node does not auto-create and auto-update single column MPP statistics. You need to do this manually by calling CREATE STATISTICS. When you create statistics on a distributed table, SQL Server PDW creates an SMP statistics object on each Compute node, and one merged statistics object on the Control node. Since all replicated tables have the same statistics, generating merged statistics only applies to distributed tables.  
  
External table statistics  
SQL Server PDW supports both full scan and sampled statistics on statistics on external tables.  
  
To create sampled statistics on an external table, SQL Server PDW first determines the sample size by using a method similar to the one used by SQL Server. Then SQL Server PDW imports the sampled rows according to the determined sample size. Only the columns specified in the CREATE STATISTICS statement are imported; the entire row does not get imported. After the data is imported, SQL Server PDW generates SQL Server statistics and stores them on the Control node.  
  
To create full scan statistics, SQL Server PDW creates statistics on all of the rows, using only the columns specified by the CREATE STATISTICS statement. Only the columns specified are imported into SQL Server PDW.  
  
auto-create statistics  
For the statistics on the Compute nodes, SQL Server PDW has AUTO_CREATE_STATISTICS set to on. Using this setting, the SQL Server query optimizer, running on each Compute node, generates single-column statistics as necessary to create a high quality SQL Server query plan.  
  
For the statistics on the Control node, the AUTO_CREATE_STATISTICS setting does not apply. Single-column statistics auto-created on Compute nodes do not get merged and stored on the Control node. Since the MPP plan, including data movement operations, is generated on the Control node, the cost-based MPP query optimizer does not auto-create single-column statistics to improve the query plan for data movement operations.  
  
Therefore, to get the benefit of the cost-based MPP query optimizer for data movement and other MPP-related operations, you will need to create statistics on strategic columns of distributed tables by using the CREATE STATISTICS statement.  
  
auto-update statistics  
  
## <a name="StatisticsDifferences"></a>Statistics Differences From SQL Server  
  
### <a name="CreateDifferences"></a>Creating Statistics Differences From SQL Server  
Creating statistics in SQL Server PDW has a few differences from creating statistics in SQL Server. In SQL Server, you can create statistics in one of the following ways:  
  
1.  Create an index. Statistics are always created for each index.  
  
2.  Create statistics with the CREATE STATISTICS (Transact-SQL) statement. This creates statistics on one or more columns in a table or an indexed view.  
  
3.  Set the AUTO_CREATE_STATISTICS setting to ON. This creates single-column statistics on columns in the query predicate, as needed to create a high quality query plan.  
  
The following table describes the differences between creating statistics in SQL Server and SQL Server PDW.  
  
|Method for Creating Statistics|SQL Server|SQL Server PDW|  
|----------------------------------|--------------|------------------|  
|Create a row store index on a table.|Creates SMP statistics on the index.|Creates SMP statistics on each Compute node.<br /><br />Merges the SMP statistics and stores the merged statistics object on the Control node.|  
|Create a column store index on a table|Creates SMP statistics on the first column of the table|Creates SMP statistics on each Compute node.<br /><br />Does not create merged statistics on the Control node|  
|Run the CREATE STATISTICS statement on a table.|Creates statistics on the table.|Creates SMP statistics on the associated tables stored on the Compute nodes.<br /><br />Merges the SMP statistics and stores the merged statistics on the Control node.|  
|AUTO_CREATE_STATISTICS set to ON.|User-configurable. When on, the query optimizer creates SMP statistics on individual columns in the query predicate, as necessary, to improve cardinality estimates for the query plan. These single-column statistics are created on columns that do not already have a histogram in an existing statistics object.|Always on for user databases on the Compute nodes. The query optimizer, running on each Compute node, creates SMP statistics on individual columns in the query predicate, as necessary, to improve cardinality estimates. The single-column statistics are created on columns that do not already have a histogram in an existing statistics object.and is not user-configurable.|  
  
### <a name="UpdateDifferences"></a>Updating Statistics Differences FROM SQL Server  
Using SQL Server, you can update statistics in the following ways:  
  
1.  Update statistics with the UPDATE STATISTICS (Transact-SQL) statement, or a stored procedure that calls UPDATE STATISTICS.  
  
2.  Set the AUTO_UPDATE_STATISTICS setting to ON. With this method, the SQL Server query optimizer determines when statistics might be out-of-date and then updates them when they are needed for a query plan.  
  
The following table describes the similarities and differences between SQL Server and SQL Server PDW for updating query optimization statistics.  
  
|Method for Updating Statistics|SQL Server|SQL Server PDW|  
|----------------------------------|--------------|------------------|  
|Set the AUTO_UPDATE_STATISTICS setting to ON.|The SQL Server query optimizer determines when statistics might be out-of-date and then updates them when they are needed for a query plan.|AUTO_UPDATE_STATISTICS is not configurable in SQL Server PDW. All SQL Server databases on the Compute nodes have AUTO_UPDATE_STATISTICS set to ON.<br /><br />When SQL Server updates statistics on the Compute nodes, as a result of the AUTO_CREATE_STATISTICS setting, SQL Server PDW***does not*** create merged statistics on the Control node.|  
|Run the UPDATE STATISTICS statement.|Updates statistics on a table. The parameters determine whether the update applies to all statistics objects on a table, one statistics object, or one index.|Updates SQL Server statistics on the associated tables stored on the Compute nodes.<br /><br />Updates the merged statistics on the Control node.|  
  
## <a name="StatisticsBestPractices"></a>Recommendations  
  
### Creating Statistics  
  
1.  Create statistics on distributed tables, particularly columns in query join predicates. This enables you to gain the benefit of the cost-based query optimizer for data movement and other MPP-related operations.  
  
    -   We recommend creating single-column statistics on each column in **ORDER BY** clauses, join predicates, **WHERE** clauses, and **GROUP BY** clauses.  
  
    -   For tables with many columns and rows, it usually takes too long to create and update single-column statistics on every column.  
  
2.  Follow the SQL Server guidance for creating and updating statistics. For example, you might have queries for which multi-column statistics will improve performance. For more information, see [Statistics](http://msdn.microsoft.com/en-us/library/ms190397.aspx) on MSDN. (Note, this article refers to a Database Tuning Advisor, which is not available in SQL Server PDW.)  
  
### Updating Statistics  
SQL Server automatically updates statistics on the Compute nodes as necessary to improve the SQL Server query plan. However, for distributed tables, SQL Server PDW does not automatically update statistics on the Control node. Using the [UPDATE STATISTICS &#40;SQL Server PDW&#41;](../sqlpdw/update-statistics-sql-server-pdw.md) statement is the only way to update statistics on the Control node.  
  
1.  Update statistics after loads or when significant changes have occurred.  
  
2.  Don’t update statistics “too” often for performance reasons.  
  
    Updating statistics ensures that queries compile with up-to-date statistics. However, updating statistics causes queries to recompile, which can be time-consuming. We recommend not updating statistics too often because there is a performance tradeoff between improving query plans and recompilation time. The specific tradeoffs depend on your application.  
  
3.  Follow the SQL Server guidance for when to update statistics. For more information, see [Statistics](http://msdn.microsoft.com/en-us/library/ms190397.aspx) on MSDN. (Note, this article refers to a Database Tuning Advisor, which is not available in SQL Server PDW.)  
  
## Related Statistics Tasks  
  
|Statistics Task|Description|  
|-------------------|---------------|  
|Create statistics.|[CREATE STATISTICS &#40;SQL Server PDW&#41;](../sqlpdw/create-statistics-sql-server-pdw.md)|  
|Update statistics.|[UPDATE STATISTICS &#40;SQL Server PDW&#41;](../sqlpdw/update-statistics-sql-server-pdw.md)|  
|Drop statistics.|[DROP STATISTICS &#40;SQL Server PDW&#41;](../sqlpdw/drop-statistics-sql-server-pdw.md)|  
  
## See Also  
[Common Metadata Query Examples &#40;SQL Server PDW&#41;](../sqlpdw/common-metadata-query-examples-sql-server-pdw.md)  
  