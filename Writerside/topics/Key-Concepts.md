# Key-Concepts

## Primary Indexes in ClickHouse
We will first understand why Primary Indexes are important by running following example ourselves .
`Create a table`
```SQL
CREATE TABLE hits_NoPrimaryKey
(
    `UserID` UInt32,
    `URL` String,
    `EventTime` DateTime
)
ENGINE = MergeTree
PRIMARY KEY tuple();
```

`Insert data` <a id="insert">into table </a>
```SQL
INSERT INTO hits_NoPrimaryKey SELECT
   intHash32(UserID) AS UserID,
   URL,
   EventTime
FROM url('https://datasets.clickhouse.com/hits/tsv/hits_v1.tsv.xz', 'TSV', 'WatchID UInt64,  JavaEnable UInt8,  
Title String,  GoodEvent Int16,  EventTime DateTime,  EventDate Date,  CounterID UInt32,  ClientIP UInt32,  
ClientIP6 FixedString(16),  RegionID UInt32,  UserID UInt64,  CounterClass Int8,  OS UInt8,  UserAgent UInt8,  
URL String,  Referer String,  URLDomain String,  RefererDomain String,  Refresh UInt8,  IsRobot UInt8,  
RefererCategories Array(UInt16),  URLCategories Array(UInt16), URLRegions Array(UInt32),  RefererRegions Array(UInt32),  
ResolutionWidth UInt16,  ResolutionHeight UInt16,  ResolutionDepth UInt8,  FlashMajor UInt8, FlashMinor UInt8,  
FlashMinor2 String,  NetMajor UInt8,  NetMinor UInt8, UserAgentMajor UInt16,  UserAgentMinor FixedString(2),  
CookieEnable UInt8, JavascriptEnable UInt8,  IsMobile UInt8,  MobilePhone UInt8,  MobilePhoneModel String,  
Params String,  IPNetworkID UInt32,  TraficSourceID Int8, SearchEngineID UInt16,  SearchPhrase String,  
AdvEngineID UInt8,  IsArtifical UInt8,  WindowClientWidth UInt16,  WindowClientHeight UInt16,  ClientTimeZone Int16,  
ClientEventTime DateTime,  SilverlightVersion1 UInt8, SilverlightVersion2 UInt8,  SilverlightVersion3 UInt32,  
SilverlightVersion4 UInt16,  PageCharset String,  CodeVersion UInt32,  IsLink UInt8,  IsDownload UInt8,  
IsNotBounce UInt8,  FUniqID UInt64,  HID UInt32,  IsOldCounter UInt8, IsEvent UInt8,  IsParameter UInt8,  
DontCountHits UInt8,  WithHash UInt8, HitColor FixedString(1),  UTCEventTime DateTime,  Age UInt8,  Sex UInt8,  
Income UInt8,  Interests UInt16,  Robotness UInt8,  GeneralInterests Array(UInt16), RemoteIP UInt32,  
RemoteIP6 FixedString(16),  WindowName Int32,  OpenerName Int32,  HistoryLength Int16,  BrowserLanguage FixedString(2),  
BrowserCountry FixedString(2),  SocialNetwork String,  SocialAction String,  HTTPError UInt16, SendTiming Int32,  
DNSTiming Int32,  ConnectTiming Int32,  ResponseStartTiming Int32,  ResponseEndTiming Int32,  FetchTiming Int32,  
RedirectTiming Int32, DOMInteractiveTiming Int32,  DOMContentLoadedTiming Int32,  DOMCompleteTiming Int32,  
LoadEventStartTiming Int32,  LoadEventEndTiming Int32, NSToDOMContentLoadedTiming Int32,  FirstPaintTiming Int32,  
RedirectCount Int8, SocialSourceNetworkID UInt8,  SocialSourcePage String,  ParamPrice Int64, ParamOrderID String,  
ParamCurrency FixedString(3),  ParamCurrencyID UInt16, GoalsReached Array(UInt32),  OpenstatServiceName String,  
OpenstatCampaignID String,  OpenstatAdID String,  OpenstatSourceID String,  UTMSource String, UTMMedium String,  
UTMCampaign String,  UTMContent String,  UTMTerm String, FromTag String,  HasGCLID UInt8,  RefererHash UInt64,  
URLHash UInt64,  CLID UInt32,  YCLID UInt64,  ShareService String,  ShareURL String,  ShareTitle String,  
ParsedParams Nested(Key1 String,  Key2 String, Key3 String, Key4 String, Key5 String,  ValueDouble Float64),  
IslandID FixedString(16),  RequestNum UInt32,  RequestTry UInt8')
WHERE URL != '';
```


`Run a query to find top 10 cicked URLs by`
```SQL
SELECT URL, count(URL) as Count
FROM hits_NoPrimaryKey
WHERE UserID = 749927693
GROUP BY URL
ORDER BY Count DESC
LIMIT 10;
```
> Result output indicates that ClickHouse executed a full table scan! Each single row of the 8.87 million rows of our 
> table was streamed into ClickHouse.

>To make this (way) more efficient and (much) faster, we need to use a table with an appropriate primary key
> These tables are designed to receive millions of row inserts per second and store very large 
> (100s of Petabytes) volumes of data. Data is quickly written to a table part by part, with rules applied for 
> merging the parts in the background. In ClickHouse each part has its own primary index. When parts are merged, 
> then the merged part’s primary indexes are also merged. At the very large scale that ClickHouse is designed for, 
> it is paramount to be very disk and memory efficient. Therefore, instead of indexing every row, the primary index 
> for a part has one index entry (known as a ‘mark’) per group of rows (called ‘granule’) 
> this technique is called sparse index.


`Create a table with compound primary key`
```SQL
CREATE TABLE hits_UserID_URL
(
    `UserID` UInt32,
    `URL` String,
    `EventTime` DateTime
)
ENGINE = MergeTree
PRIMARY KEY (UserID, URL)
ORDER BY (UserID, URL, EventTime);
```

`Insert data`[using above mentioned query](#insert)

`OPTIMIZE TABLE `
```SQL
OPTIMIZE TABLE hits_UserID_URL FINAL;
```

`Run same query`
```SQL
SELECT URL, count(URL) AS Count
FROM hits_UserID_URL
WHERE UserID = 749927693
GROUP BY URL
ORDER BY Count DESC
LIMIT 10;
```
>The output for the ClickHouse client is now showing that instead of doing a full table scan, only 8.19 thousand rows 
> were streamed into ClickHouse.

> Data is stored on disk ordered by primary key column(s)
- If we had specified only the sorting key, then the primary key would be implicitly defined to be equal to the 
sorting key.
- The primary index that is based on the primary key is completely loaded into the main memory.
- In order to have consistency in the guide’s diagrams and in order to maximise compression ratio we defined a separate 
sorting key that includes all of our table's columns (if in a column similar data is placed close to each other, 
for example via sorting, then that data will be compressed better).
- The primary key needs to be a `prefix` of the sorting key if both are specified.

>the 8.87 million rows are stored on disk in lexicographic ascending order by the primary key columns 
>(and the additional sort key columns) i.e. in this case
>first by UserID,
>then by URL,
>and lastly by EventTime:
> 
> As the primary key defines the lexicographical order of the rows on disk, a table can only have one primary key.

- A table's column values are logically divided into granules. 
  - A granule is the smallest indivisible data set that is streamed into ClickHouse for data processing. 
  This means that instead of reading individual rows, ClickHouse is always reading a whole group (granule) of rows.
  - Column values are not physically stored inside granules, they just a logical organization for query processing

- The primary index has one entry per granule
  - The primary index is used for selecting granules
  - the primary index stores the primary key column values from first row(0th row) and then each 8192nd row of the table. 

## Using multiple primary indexes
- Secondary key columns can (not) be inefficient
>When a query is filtering on a column that is part of a compound key and is the first key column, then ClickHouse is 
> running the binary search algorithm over the key column's index marks. In case it is not the first key column and 
> clickhouse will use generic exclusion search algorithm (instead of binary search) over the URL column's index marks.
> The effectiveness of this algorithm is dependent on the cardinality difference

`Run query show below, it filters on URL instead of first primary key.`<b id="secondPrimary"> It will scan the whole table </b>
```SQL
SELECT UserID, count(UserID) AS Count
FROM hits_UserID_URL
WHERE URL = 'http://public_search'
GROUP BY UserID
ORDER BY Count DESC
LIMIT 10;
```
> As a consequence, if we want to significantly speed up our sample query that filters for rows with a specific URL 
> then we need to use a primary index optimized to that query.

- Options for creating additional primary indexes
  - Creating a second table with a different primary key.
  - Creating a materialized view on our existing table.
  - Adding a projection to our existing table.

- We will use Projections

`Create a projection on our existing table:`
```SQL
ALTER TABLE hits_UserID_URL
    ADD PROJECTION prj_url_userid
    (
        SELECT *
        ORDER BY (URL, UserID)
    );
```
`Materialize the projection:`
```SQL
ALTER TABLE hits_UserID_URL
    MATERIALIZE PROJECTION prj_url_userid;
```
> the projection is creating a hidden table whose row order and primary index is based on the given
> ORDER BY clause of the projection.  
> 
> we use the MATERIALIZE keyword in order to immediately populate the hidden table with all 8.87 million rows from the 
> source table hits_UserID_URL.
> 
> If new rows are inserted into the source table hits_UserID_URL, then that rows are automatically also inserted
> into the hidden table
> 
> A query is always (syntactically) targeting the source table hits_UserID_URL, but if the row order and primary index 
> of the hidden table allows a more effective query execution, then that hidden table will be used instead.

[Run the same query to filter on URL column](#secondPrimary)

[Reference](https://clickhouse.com/docs/en/optimize/sparse-primary-indexes)