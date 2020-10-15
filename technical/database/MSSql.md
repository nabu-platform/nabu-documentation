# Timestamp datatype

Based on the [microsoft documentation](https://docs.microsoft.com/en-us/sql/connect/jdbc/using-basic-data-types?view=sql-server-ver15), SQL server has a data type called "timestamp" that is entirely unrelated to the JDBC concept of timestamp.  An actual JDBC timestamp is sql data type "datetime" or "datetime2".

The SQL server data type "timestamp" is a hexadecimal value which is used for row versioning. It may actually have been renamed to "rowversion" since 2008.
According to the above mentioned microsoft documentation, in JDBC it is actually represented as a binary and should be represented in java as a byte array.
