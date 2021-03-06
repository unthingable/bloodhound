Bloodhound [![TravisCI](https://travis-ci.org/bitemyapp/bloodhound.svg)](https://travis-ci.org/bitemyapp/bloodhound) [![Hackage](https://img.shields.io/hackage/v/bloodhound.svg?style=flat)](https://hackage.haskell.org/package/bloodhound)
==========

![Bloodhound (dog)](./bloodhound.jpg)

Elasticsearch client and query DSL for Haskell
==============================================

Why?
----

Search doesn't have to be hard. Let the dog do it.

Endorsements
------------

"Bloodhound makes Elasticsearch almost tolerable!" - Almost-gruntled user

"ES is a nightmare but Bloodhound at least makes it tolerable." - Same user, later opinion.

Version compatibility
---------------------

Elasticsearch \>= 1.0 is recommended. Bloodhound mostly works with 0.9.x, but I don't recommend it if you expect everything to work. As of Bloodhound 0.3 all \>=1.0 versions of Elasticsearch work.

Current versions we test against are 1.0.3, 1.1.2, 1.2.3, 1.3.2, and 1.4.0. We also check that GHC 7.6 and 7.8 both build and pass tests. See our [TravisCI](https://travis-ci.org/bitemyapp/bloodhound) to learn more.

Stability
---------

Bloodhound is stable for production use. I will strive to avoid breaking API compatibility from here on forward, but dramatic features like a type-safe, fully integrated mapping API may require breaking things in the future.

Hackage page and Haddock documentation
======================================

<http://hackage.haskell.org/package/bloodhound>

Examples
========

Index Operations
----------------

### Create Index

``` {.haskell}

-- Formatted for use in ghci, so there are "let"s in front of the decls.

-- if you see :{ and :}, they're so you can copy-paste
-- the multi-line examples into your ghci REPL.

:set -XDeriveGeneric
:{
import Database.Bloodhound
import Data.Aeson
import Data.Either (Either(..))
import Data.Maybe (fromJust)
import Data.Time.Calendar (Day(..))
import Data.Time.Clock (secondsToDiffTime, UTCTime(..))
import Data.Text (Text)
import GHC.Generics (Generic)
import Network.HTTP.Client
import qualified Network.HTTP.Types.Status as NHTS

-- no trailing slashes in servers, library handles building the path.
let testServer = (Server "http://localhost:9200")
let testIndex = IndexName "twitter"
let testMapping = MappingName "tweet"

-- defaultIndexSettings is exported by Database.Bloodhound as well
let defaultIndexSettings = IndexSettings (ShardCount 3) (ReplicaCount 2)

-- createIndex returns IO Reply

-- response :: Reply, Reply is a synonym for Network.HTTP.Conduit.Response
response <- createIndex testServer defaultIndexSettings testIndex
:}

```

### Delete Index

#### Code

``` {.haskell}

-- response :: Reply
response <- deleteIndex testServer testIndex

```

#### Example Response

``` {.haskell}

-- print response if it was a success
Response {responseStatus = Status {statusCode = 200, statusMessage = "OK"}
        , responseVersion = HTTP/1.1
        , responseHeaders = [("Content-Type", "application/json; charset=UTF-8")
                           , ("Content-Length", "21")]
        , responseBody = "{\"acknowledged\":true}"
        , responseCookieJar = CJ {expose = []}
        , responseClose' = ResponseClose}

-- if the index to be deleted didn't exist anyway
Response {responseStatus = Status {statusCode = 404, statusMessage = "Not Found"}
        , responseVersion = HTTP/1.1
        , responseHeaders = [("Content-Type", "application/json; charset=UTF-8")
                           , ("Content-Length","65")]
        , responseBody = "{\"error\":\"IndexMissingException[[twitter] missing]\",\"status\":404}"
        , responseCookieJar = CJ {expose = []}
        , responseClose' = ResponseClose}

```

### Refresh Index

#### Note, you **have** to do this if you expect to read what you just wrote

``` {.haskell}

resp <- refreshIndex testServer testIndex

```

#### Example Response

``` {.haskell}

-- print resp on success
Response {responseStatus = Status {statusCode = 200, statusMessage = "OK"}
        , responseVersion = HTTP/1.1
        , responseHeaders = [("Content-Type", "application/json; charset=UTF-8")
                           , ("Content-Length","50")]
        , responseBody = "{\"_shards\":{\"total\":10,\"successful\":5,\"failed\":0}}"
        , responseCookieJar = CJ {expose = []}
        , responseClose' = ResponseClose}

```

Mapping Operations
------------------

### Create Mapping

``` {.haskell}

-- don't forget imports and the like at the top.

data TweetMapping = TweetMapping deriving (Eq, Show)

-- I know writing the JSON manually sucks.
-- I don't have a proper data type for Mappings yet.
-- Let me know if this is something you need.

:{
instance ToJSON TweetMapping where
  toJSON TweetMapping =
    object ["tweet" .=
      object ["properties" .=
        object ["location" .=
          object ["type" .= ("geo_point" :: Text)]]]]
:}

resp <- putMapping testServer testIndex testMapping TweetMapping

```

### Delete Mapping

``` {.haskell}

resp <- deleteMapping testServer testIndex testMapping

```

Document Operations
-------------------

### Indexing Documents

``` {.haskell}

-- don't forget the imports and derive generic setting for ghci
-- at the beginning of the examples.

:{
data Location = Location { lat :: Double
                         , lon :: Double } deriving (Eq, Generic, Show)

data Tweet = Tweet { user     :: Text
                   , postDate :: UTCTime
                   , message  :: Text
                   , age      :: Int
                   , location :: Location } deriving (Eq, Generic, Show)

exampleTweet = Tweet { user     = "bitemyapp"
                     , postDate = UTCTime
                                  (ModifiedJulianDay 55000)
                                  (secondsToDiffTime 10)
                     , message  = "Use haskell!"
                     , age      = 10000
                     , location = Location 40.12 (-71.34) }

-- automagic (generic) derivation of instances because we're lazy.
instance ToJSON   Tweet
instance FromJSON Tweet
instance ToJSON   Location
instance FromJSON Location
:}

-- Should be able to toJSON and encode the data structures like this:
-- λ> toJSON $ Location 10.0 10.0
-- Object fromList [("lat",Number 10.0),("lon",Number 10.0)]
-- λ> encode $ Location 10.0 10.0
-- "{\"lat\":10,\"lon\":10}"

resp <- indexDocument testServer testIndex testMapping exampleTweet (DocId "1")

```

#### Example Response

``` {.haskell}

Response {responseStatus =
  Status {statusCode = 200, statusMessage = "OK"}
    , responseVersion = HTTP/1.1, responseHeaders =
    [("Content-Type","application/json; charset=UTF-8"),
     ("Content-Length","75")]
    , responseBody = "{\"_index\":\"twitter\",\"_type\":\"tweet\",\"_id\":\"1\",\"_version\":2,\"created\":false}"
    , responseCookieJar = CJ {expose = []}, responseClose' = ResponseClose}

```

### Deleting Documents

``` {.haskell}

resp <- deleteDocument testServer testIndex testMapping (DocId "1")

```

### Getting Documents

``` {.haskell}

-- n.b., you'll need the earlier imports. responseBody is from http-conduit

resp <- getDocument testServer testIndex testMapping (DocId "1")

-- responseBody :: Response body -> body
let body = responseBody resp

-- you have two options, you use decode and just get Maybe (EsResult Tweet)
-- or you can use eitherDecode and get Either String (EsResult Tweet)

let maybeResult = decode body :: Maybe (EsResult Tweet)
-- the explicit typing is so Aeson knows how to parse the JSON.

-- use either if you want to know why something failed to parse.
-- (string errors, sadly)
let eitherResult = eitherDecode body :: Either String (EsResult Tweet)

-- print eitherResult should look like:
Right (EsResult {_index = "twitter"
               , _type = "tweet"
               , _id = "1"
               , _version = 2
               , found = Just True
               , _source = Tweet {user = "bitemyapp"
               , postDate = 2009-06-18 00:00:10 UTC
               , message = "Use haskell!"
               , age = 10000
               , location = Location {lat = 40.12, lon = -71.34}}})

-- _source in EsResult is parametric, we dispatch the type by passing in what we expect (Tweet) as a parameter to EsResult.

-- use the _source record accessor to get at your document
fmap _source eitherResult
Right (Tweet {user = "bitemyapp"
            , postDate = 2009-06-18 00:00:10 UTC
            , message = "Use haskell!"
            , age = 10000
            , location = Location {lat = 40.12, lon = -71.34}})

```

Bulk Operations
---------------

### Bulk create, index

``` {.haskell}

-- don't forget the imports and derive generic setting for ghci
-- at the beginning of the examples.

:{
-- Using the earlier Tweet datatype and exampleTweet data

-- just changing up the data a bit.
let bulkTest = exampleTweet { user = "blah" }
let bulkTestTwo = exampleTweet { message = "woohoo!" }

-- create only bulk operation
-- BulkCreate :: IndexName -> MappingName -> DocId -> Value -> BulkOperation
let firstOp = BulkCreate testIndex
              testMapping (DocId "3") (toJSON bulkTest)

-- index operation "create or update"
let sndOp   = BulkIndex testIndex
              testMapping (DocId "4") (toJSON bulkTestTwo)

-- Some explanation, the final "Value" type that BulkIndex,
-- BulkCreate, and BulkUpdate accept is the actual document
-- data that your operation applies to. BulkDelete doesn't
-- take a value because it's just deleting whatever DocId 
-- you pass.

-- list of bulk operations
let stream = [firstDoc, secondDoc]

-- Fire off the actual bulk request
-- bulk :: Server -> [BulkOperation] -> IO Reply
resp <- bulk testServer stream
:}

```

### Encoding individual bulk API operations

``` {.haskell}
-- the following functions are exported in Bloodhound so
-- you can build up bulk operations yourself
encodeBulkOperations :: V.Vector BulkOperation -> L.ByteString
encodeBulkOperation :: BulkOperation -> L.ByteString

-- How to use the above:
data BulkTest = BulkTest { name :: Text } deriving (Eq, Generic, Show)
instance FromJSON BulkTest
instance ToJSON BulkTest

_ <- insertData
let firstTest = BulkTest "blah"
let secondTest = BulkTest "bloo"
let firstDoc = BulkIndex testIndex
               testMapping (DocId "2") (toJSON firstTest)
let secondDoc = BulkCreate testIndex
               testMapping (DocId "3") (toJSON secondTest)
let stream = V.fromList [firstDoc, secondDoc] :: V.Vector BulkOperation

-- to encode yourself
let firstDocEncoded = encode firstDoc :: L.ByteString

-- to encode a vector of bulk operations
let encodedOperations = encodeBulkOperations stream

-- to insert into a particular server
-- bulk :: Server -> V.Vector BulkOperation -> IO Reply
_ <- bulk testServer stream

```

Search
------

### Querying

#### Term Query

``` {.haskell}

-- exported by the Client module, just defaults some stuff.
-- mkSearch :: Maybe Query -> Maybe Filter -> Search
-- mkSearch query filter = Search query filter Nothing False 0 10

let query = TermQuery (Term "user" "bitemyapp") Nothing

-- AND'ing identity filter with itself and then tacking it onto a query
-- search should be a null-operation. I include it for the sake of example.
-- <||> (or/plus) should make it into a search that returns everything.

let filter = IdentityFilter <&&> IdentityFilter

-- constructing the search object the searchByIndex function dispatches on.
let search = mkSearch (Just query) (Just filter)

-- you can also searchByType and specify the mapping name.
reply <- searchByIndex testServer testIndex search

let result = eitherDecode (responseBody reply) :: Either String (SearchResult Tweet)

λ> fmap (hits . searchHits) result
Right [Hit {hitIndex = IndexName "twitter"
          , hitType = MappingName "tweet"
          , hitDocId = DocId "1"
          , hitScore = 0.30685282
          , hitSource = Tweet {user = "bitemyapp"
                             , postDate = 2009-06-18 00:00:10 UTC
                             , message = "Use haskell!"
                             , age = 10000
                             , location = Location {lat = 40.12, lon = -71.34}}}]

```

#### Match Query

``` {.haskell}

let query = QueryMatchQuery $ mkMatchQuery (FieldName "user") (QueryString "bitemyapp")
let search = mkSearch (Just query) Nothing

```

#### Multi-Match Query

``` {.haskell}

let fields = [FieldName "user", FieldName "message"]
let query = QueryMultiMatchQuery $ mkMultiMatchQuery fields (QueryString "bitemyapp")
let search = mkSearch (Just query) Nothing

```

#### Bool Query

``` {.haskell}

let innerQuery = QueryMatchQuery $
                 mkMatchQuery (FieldName "user") (QueryString "bitemyapp")
let query = QueryBoolQuery $
            mkBoolQuery [innerQuery] [] []
let search = mkSearch (Just query) Nothing

```

#### Boosting Query

``` {.haskell}

let posQuery = QueryMatchQuery $
               mkMatchQuery (FieldName "user") (QueryString "bitemyapp")
let negQuery = QueryMatchQuery $
               mkMatchQuery (FieldName "user") (QueryString "notmyapp")
let query = QueryBoostingQuery $
            BoostingQuery posQuery negQuery (Boost 0.2)

```

#### Rest of the query/filter types

Just follow the pattern you've seen here and check the Hackage API documentation.

### Sorting

``` {.haskell}

let sortSpec = DefaultSortSpec $ mkSort (FieldName "age") Ascending

-- mkSort is a shortcut function that takes a FieldName and a SortOrder
-- to generate a vanilla DefaultSort.
-- checkt the DefaultSort type for the full list of customizable options.

-- From and size are integers for pagination.

-- When sorting on a field, scores are not computed. By setting TrackSortScores to true, scores will still be computed and tracked.

-- type Sort = [SortSpec]
-- type TrackSortScores = Bool
-- type From = Int
-- type Size = Int

-- Search takes Maybe Query
--              -> Maybe Filter
--              -> Maybe Sort
--              -> TrackSortScores
--              -> From -> Size

-- just add more sortspecs to the list if you want tie-breakers.
let search = Search Nothing (Just IdentityFilter) (Just [sortSpec]) False 0 10

```

### Filtering

#### And, Not, and Or filters

Filters form a monoid and seminearring.

``` {.haskell}

instance Monoid Filter where
  mempty = IdentityFilter
  mappend a b = AndFilter [a, b] defaultCache

instance Seminearring Filter where
  a <||> b = OrFilter [a, b] defaultCache

-- AndFilter and OrFilter take [Filter] as an argument.

-- This will return anything, because IdentityFilter returns everything
OrFilter [IdentityFilter, someOtherFilter] False

-- This will return exactly what someOtherFilter returns
AndFilter [IdentityFilter, someOtherFilter] False

-- Thanks to the seminearring and monoid, the above can be expressed as:

-- "and"
IdentityFilter <&&> someOtherFilter

-- "or"
IdentityFilter <||> someOtherFilter

-- Also there is a NotFilter, it only accepts a single filter, not a list.

NotFilter someOtherFilter False

```

#### Identity Filter

``` {.haskell}

-- And'ing two Identity
let queryFilter = IdentityFilter <&&> IdentityFilter

let search = mkSearch Nothing (Just queryFilter)

reply <- searchByType testServer testIndex testMapping search

```

#### Boolean Filter

Similar to boolean queries.

``` {.haskell}

-- Will return only items whose "user" field contains the term "bitemyapp"
let queryFilter = BoolFilter (MustMatch (Term "user" "bitemyapp") False)

-- Will return only items whose "user" field does not contain the term "bitemyapp"
let queryFilter = BoolFilter (MustNotMatch (Term "user" "bitemyapp") False)

-- The clause (query) should appear in the matching document.
-- In a boolean query with no must clauses, one or more should
-- clauses must match a document. The minimum number of should
-- clauses to match can be set using the minimum_should_match parameter.
let queryFilter = BoolFilter (ShouldMatch [(Term "user" "bitemyapp")] False)

```

#### Exists Filter

``` {.haskell}

-- Will filter for documents that have the field "user"
let existsFilter = ExistsFilter (FieldName "user")

```

#### Geo BoundingBox Filter

``` {.haskell}

-- topLeft and bottomRight
let box = GeoBoundingBox (LatLon 40.73 (-74.1)) (LatLon 40.10 (-71.12))

let constraint = GeoBoundingBoxConstraint (FieldName "tweet.location") box False GeoFilterMemory

```

#### Geo Distance Filter

``` {.haskell}

let geoPoint = GeoPoint (FieldName "tweet.location") (LatLon 40.12 (-71.34))

-- coefficient and units
let distance = Distance 10.0 Miles

-- GeoFilterType or NoOptimizeBbox
let optimizeBbox = OptimizeGeoFilterType GeoFilterMemory

-- SloppyArc is the usual/default optimization in Elasticsearch today
-- but pre-1.0 versions will need to pick Arc or Plane.

let geoFilter = GeoDistanceFilter geoPoint distance SloppyArc optimizeBbox False

```

#### Geo Distance Range Filter

Think of a donut and you won't be far off.

``` {.haskell}

let geoPoint = GeoPoint (FieldName "tweet.location") (LatLon 40.12 (-71.34))

let distanceRange = DistanceRange (Distance 0.0 Miles) (Distance 10.0 Miles)

let geoFilter = GeoDistanceRangeFilter geoPoint distanceRange

```

#### Geo Polygon Filter

``` {.haskell}

-- I think I drew a square here.
let points = [LatLon 40.0 (-70.00),
              LatLon 40.0 (-72.00),
              LatLon 41.0 (-70.00),
              LatLon 41.0 (-72.00)]

let geoFilter = GeoPolygonFilter (FieldName "tweet.location") points

```

#### Document IDs filter

``` {.haskell}

-- takes a mapping name and a list of DocIds
IdsFilter (MappingName "tweet") [DocId "1"]

```

#### Range Filter

##### Full Range

``` {.haskell}

-- RangeFilter :: FieldName
--                -> Either HalfRange Range
--                -> RangeExecution
--                -> Cache -> Filter

let filter = RangeFilter (FieldName "age")
             (Right (RangeLtGt (LessThan 100000.0) (GreaterThan 1000.0)))
             RangeExecutionIndex False

```

##### Half Range

``` {.haskell}

let filter = RangeFilter (FieldName "age")
             (Left (HalfRangeLt (LessThan 100000.0)))
             RangeExecutionIndex False

```

#### Regexp Filter

``` {.haskell}

-- RegexpFilter
--   :: FieldName
--      -> Regexp
--      -> RegexpFlags
--      -> CacheName
--      -> Cache
--      -> CacheKey
--      -> Filter
let filter = RegexpFilter (FieldName "user") (Regexp "bite.*app")
             AllRegexpFlags (CacheName "test") False (CacheKey "key")

-- n.b.
-- data RegexpFlags = AllRegexpFlags
--                 | NoRegexpFlags
--                 | SomeRegexpFlags (NonEmpty RegexpFlag) deriving (Eq, Show)

-- data RegexpFlag = AnyString
--                | Automaton
--                | Complement
--                | Empty
--                | Intersection
--                | Interval deriving (Eq, Show)

```

### Aggregations

#### Adding aggregations to search

Aggregations can now be added to search queries, or made on their own.

``` {.haskell}
type Aggregations = M.Map Text Aggregation
data Aggregation
  = TermsAgg TermsAggregation
  | DateHistogramAgg DateHistogramAggregation
```

For convenience, \`\`\`mkAggregations\`\`\` exists, that will create an \`\`\`Aggregations\`\`\` with the aggregation provided.

For example:

``` {.haskell}
 let a = mkAggregations "users" $ TermsAgg $ mkTermsAggregation "user"
 let search = mkAggregateSearch Nothing a
```

Aggregations can be added to an existing search, using the \`\`\`aggBody\`\`\` field

``` {.haskell}
 let search  = mkSearch (Just (MatchAllQuery Nothing)) Nothing
 let search' = search {aggBody = Just a}
```

Since the \`\`\`Aggregations\`\`\` structure is just a Map Text Aggregation, M.insert can be used to add additional aggregations.

``` {.haskell}
 let a' = M.insert "age" (TermsAgg $ mkTermsAggregation "age") a
```

#### Extracting aggregations from results

Aggregations are part of the reply structure of every search, in the form of

``` {.haskell}
-- Lift decode and response body to be in the IO monad.
let decode' = liftM decode
let responseBody' = liftM responseBody
let reply = searchByIndex testServer testIndex search
let response = decode' $ responseBody' reply :: IO (Maybe (SearchResult Tweet))

-- Now that we have our response, we can extract our terms aggregation result -- which is a list of buckets.

let terms = do { response' <- response; return $ response' >>= aggregations >>= toTerms "users" }
terms
Just (Bucket {buckets = [TermsResult {termKey = "bitemyapp", termsDocCount = 1, termsAggs = Nothing}]})
```

Note that bucket aggregation results, such as the TermsResult is a member of the type class :

``` {.haskell}
class BucketAggregation a where
  key :: a -> Text
  docCount :: a -> Int
  aggs :: a -> Maybe AggregationResults
```

haskell

You can use the function to get any nested results, if there were any. For example, if there were a nested terms aggregation keyed to "age" in a TermsResult named , you would call

#### Terms Aggregation

``` {.haskell}
data TermsAggregation
  = TermsAggregation {term :: Either Text Text,
                      termInclude :: Maybe TermInclusion,
                      termExclude :: Maybe TermInclusion,
                      termOrder :: Maybe TermOrder,
                      termMinDocCount :: Maybe Int,
                      termSize :: Maybe Int,
                      termShardSize :: Maybe Int,
                      termCollectMode :: Maybe CollectionMode,
                      termExecutionHint :: Maybe ExecutionHint,
                      termAggs :: Maybe Aggregations}
```

Term Aggregations have two factory functions, , and , and can be used as follows:

``` {.haskell}
let ta = TermsAgg $ mkTermsAggregation "user"
```

There are of course other options that can be added to a Terms Aggregation, such as the collection mode:

``` {.haskell}
let ta   = mkTermsAggregation "user"
let ta'  = ta { termCollectMode = Just BreadthFirst }
let ta'' = TermsAgg ta'
```

For more documentation on how the Terms Aggregation works, see <http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html>

#### Date Histogram Aggregation

``` {.haskell}
data DateHistogramAggregation
  = DateHistogramAggregation {dateField :: FieldName,
                              dateInterval :: Interval,
                              dateFormat :: Maybe Text,
                              datePreZone :: Maybe Text,
                              datePostZone :: Maybe Text,
                              datePreOffset :: Maybe Text,
                              datePostOffset :: Maybe Text,
                              dateAggs :: Maybe Aggregations}
```

haskell

The Date Histogram Aggregation works much the same as the Terms Aggregation.

Relevant functions include , and

``` {.haskell}
let dh = DateHistogramAgg (mkDateHistogram (FieldName "postDate") Minute)
```

Date histograms also accept a :

``` {.haskell}
FractionalInterval :: Float -> TimeInterval -> Interval
-- TimeInterval is the following:
data TimeInterval = Weeks | Days | Hours | Minutes | Seconds
```

It can be used as follows:

``` {.haskell}
let dh = DateHistogramAgg (mkDateHistogram (FieldName "postDate") (FractionalInterval 1.5 Minutes))
```

The is defined as:

``` {.haskell}
data DateHistogramResult
  = DateHistogramResult {dateKey :: Int,
                         dateKeyStr :: Maybe Text,
                         dateDocCount :: Int,
                         dateHistogramAggs :: Maybe AggregationResults}
```

It is an instance of , and can have nested aggregations in each bucket.

Buckets can be extracted from a using

For more information on the Date Histogram Aggregation, see: <http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-aggregations-bucket-datehistogram-aggregation.html>

Possible future functionality
=============================

Span Queries
------------

Beginning here: <http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-span-first-query.html>

Function Score Query
--------------------

<http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html>

Node discovery and failover
---------------------------

Might require TCP support.

Support for TCP access to Elasticsearch
---------------------------------------

Pretend to be a transport client?

Bulk cluster-join merge
-----------------------

Might require making a lucene index on disk with the appropriate format.

GeoShapeQuery
-------------

<http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-geo-shape-query.html>

GeoShapeFilter
--------------

<http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-geo-shape-filter.html>

Geohash cell filter
-------------------

<http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-geohash-cell-filter.html>

HasChild Filter
---------------

<http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-has-child-filter.html>

HasParent Filter
----------------

<http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-has-parent-filter.html>

Indices Filter
--------------

<http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-indices-filter.html>

Query Filter
------------

<http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-query-filter.html>

Script based sorting
--------------------

<http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-request-sort.html#_script_based_sorting>

Collapsing redundantly nested and/or structures
-----------------------------------------------

The Seminearring instance, if deeply nested can possibly produce nested structure that is redundant. Depending on how this affects ES perforamnce, reducing this structure might be valuable.

Runtime checking for cycles in data structures
----------------------------------------------

check for n \> 1 occurrences in DFS:

<http://hackage.haskell.org/package/stable-maps-0.0.5/docs/System-Mem-StableName-Dynamic.html>

<http://hackage.haskell.org/package/stable-maps-0.0.5/docs/System-Mem-StableName-Dynamic-Map.html>

Photo Origin
============

Photo from HA! Designs: <https://www.flickr.com/photos/hadesigns/>

