## Stack

Microsoft Fabric Data & Compute Stack Diagram (DP-600 Focus) 
This stack shows how data moves from sources to reports in Fabric.


[ CONSUMPTION ] -> Power BI (Direct Lake / Reports)
      ^
      |
[ MODELING ]    -> Semantic Model (DAX, Star Schema, Measures)
      ^
      |
[ SERVING ]     -> Lakehouse (Gold) / Data Warehouse
      ^
      |
[ STORAGE ]     -> ONE LAKE (Delta Lake / Parquet Files)
      ^
      |
[ TRANSFORMATION] -> Notebooks (PySpark/SQL) / Dataflows Gen2
      ^
      |
[ INGESTION ]   -> Data Pipelines / Shortcuts / Mirroring
      ^
      |
[ SOURCES ]     -> Azure SQL, ADLS Gen2, S3, SaaS Apps


## Key Technical Focus Areas for DP-600

- Direct Lake Mode: Understand how it offers Import-level performance without copying data.
- Dataflows Gen2: Specifically for Data Ingestion and Transformation (Power Query).
- OneLake: The unified storage for all data.
- Notebooks vs. Dataflows: Knowing when to use Spark (complex) vs. Dataflows (simple/low-code).
- DAX Optimization: Performance tuning via Calculation Groups and DAX Studio.

- How I passed DP-600 : https://medium.com/towards-data-engineering/how-i-passed-the-microsoft-fabric-data-engineer-associate-exam-in-just-15-days-bd0d8d683a14
