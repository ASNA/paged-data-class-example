
### The `ASNA.DataGateHelper.PagedData` class 

The ASNA.DataGateHelper assembly is [available here](https://github.com/ASNA/ASNA.DataGateHelper). 

> This class requires at least .NET Framework 4.7.2. Be sure your project is set to at least that .NET Framework level.

> This class has a dependency on the `shortid` nuget.org package. Download its zip file (with "Download Package") and unzip it (although it doesn't end in `.zip` the package file is a zip file.) Then set a reference in your project to its DLL located here:
> `[your download folder]\shortid.3.0.1\lib\netstandard1.0\shortid.dll`


The `ASNA.DataGateHelper.PagedData` class provides logic to read through an IBM i file a page at a time using SQL. The results of a page read is a strongly-typed collection of objects--typically used to populate a list (ie, a `GridView` in ASP.NET or a `DataGate` in Windows). 

PagedData has two primary underlying enablers

1. A simple ILE RPG program that executes an SQL statement. This SQL statement queries a file on the IBM i and write the results of that query to uniquely-named temporary file (usually writing to the QTEMP library).

2. The ASNA.DataGateHelper.DGFileReader. This object provides dynamic file read capabilities. No `DclDiskFile` is required with the `DGFileReader`. It is what makes it easy to read the temporary work file created by the RPG program.

The following SQL in Figure 1 is an example of the SQL you'd use to fetch a page of data.

```
CREATE TABLE {libraryName}/{uniqueObjectId} as 
(
    WITH result AS ( 
        SELECT cmcustno, cmname 
        FROM examples/cmastnewL2 
        ORDER BY cmname, cmcustno 
        LIMIT {limit} 
        OFFSET {offset} 
    ) 
    SELECT * FROM result 
) WITH DATA
```

<small>Figure 1. Example SQL to fetch a page of data.</small>

> A tip of the SQL beanie to Eugene Belkin who provided some valuable SQL expertise that helped make this technique work.  

This SQL uses the IBM i's SQL LIMIT/OFFSET feature to fetch data a page at a time. SQL does all of the hard work fetching a page. "Pages" are calculated based on the LIMIT (the page size) and the OFFSET (a zero-based page number). LIMIT/OFFSET requires ORDER BY clause. This clause ensures correct page selection. 

At runtime, your application provides the SELECT, FROM, ORDER BY, and optionally, the WHERE clauses.

AVR can't read an IBM i SQL result set directly, so the SQL writes the result set to a specified temporary table. Immediately after the RPG program to create this temporary table, AVR then reads that file and creates a strongly-list collection of objects derived from that file's record contents. 

#### Annotated `PagedClass` Example

See the pages below for fully annotated examples using the `PagedData` class


[Test scaffolding code](https://asna.github.io/paged-data-class-example/PagedDataClassExample.vr.html)

The test scaffolding code instances one of these two classes: 

[Page through a customers file](https://asna.github.io/paged-data-class-example/PageCustomers.vr.html)

[Page through a states file](https://asna.github.io/paged-data-class-example/PageStates.vr.html)

<a href=" target="_blank">See this page</a> for a fully annotated example for 

how to use the `PagedData` class.




### The `ASNA.DataGateHelper.IBMiSqlPage` class 

This class used by PagedData to call the RPG program that executes SQL. The source code for that RPG program is below. `IBMiSqlPage's` `Call` method receives an SQL statement and several status parameters. The RPG program uses ILE RPG's embedded SQL to do an `EXECUTE IMMEDATE` on the SQL provided. 

You probably won't ever use this class directly. It is used in the `PagedData` class.

```
ctl-opt option(*nodebugio:*srcstmt:*nounref) dftactgrp(*no) ;

Dcl-Pi *N;
    Sql Char(1024);
    xMessageId char(10) ;
    xMessageId1 char(7) ;
    xMessageId2 char(10);
    xMessageLength Int(10);
    xMessageText char(1000) ;
    xReturnedSQLCode char(5) ;
    xReturnedSQLState char(5) ;
    xRowsCount int(10) ;
End-Pi;

Dcl-S RowsReturned Int(10);

// Diagnostic info
dcl-s MessageId char(10);
dcl-s MessageId1 varchar(7);
dcl-s MessageId2 like(MessageId1);
dcl-s MessageLength int(5);
dcl-s MessageText varchar(32740);
dcl-s ReturnedSQLCode char(5);
dcl-s ReturnedSQLState char(5);
dcl-s RowsCount int(10);

ConfigureSQL();

ExecSql(Sql);

Return;

Dcl-Proc ConfigureSQL;
    Dcl-Pi *N Int(10);
    End-Pi;

    EXEC SQL
        SET OPTION COMMIT = *NONE;

    EXEC SQL
        SET OPTION NAMING = *SYS;

    return 0;
End-Proc;

Dcl-Proc ExecSql;
    Dcl-Pi *N Int(10);
        Sql Char(1024);
    End-Pi;

    If Sql = *Blanks;
        Return 0;
    EndIf;

    EXEC SQL
        EXECUTE IMMEDIATE :Sql;
        Diagnostics();
    Return 0;
End-Proc;

dcl-proc Diagnostics ;
    exec sql GET DIAGNOSTICS
        :RowsCount = ROW_COUNT;

    exec sql GET DIAGNOSTICS CONDITION 1
        :ReturnedSqlCode = DB2_RETURNED_SQLCODE,
        :ReturnedSQLState = RETURNED_SQLSTATE,
        :MessageLength = MESSAGE_LENGTH,
        :MessageText = MESSAGE_TEXT,
        :MessageId = DB2_MESSAGE_ID,
        :MessageId1 = DB2_MESSAGE_ID1,
        :MessageId2 = DB2_MESSAGE_ID2 ;

    AssignDiags();
end-proc;

dcl-proc AssignDiags;
    xMessageId          = MessageId;
    xMessageId1         = MessageId1;
    xMessageId2         = MessageId2;
    xMessageLength      = MessageLength;
    xMessageText        = MessageText;
    xReturnedSQLCode    = ReturnedSQLCode;
    xReturnedSQLState   = ReturnedSQLState;
    xRowsCount          = RowsCount;
end-proc;
```
<small>Figure 2. IBMiSqlPage class source code.</small>

### The `ASNA.DataGateHelper.IBMiCmdExec` class 

This is a utility class you can use to call IBM i's QCMDEXC API to execute a command on the IBM i. 

Using the `IBMiCmdExec` class is very simple. Declare the class, passing it a DataGate DB object and then use the `Call` method to execute the IBM i command provided. The DG object you passed to the constructor _must_ be called before using the `Call` method.

DclFld cmdExec Type(IBMiCmdExec) New(DGDB) 

Connect DGDB 

CmdExec.Call('CLRLIB RP_DATA') 

