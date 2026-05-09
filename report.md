**Created By:** Juan Yovian - 24911605

# 1. Introduction

# 2. Graph Database Design

## 2.1. Design Overview

The property graph consists of two node types and two relationship types.

##### Nodes

- `Airline` - representing a unique airline operation in the dataset. Each node stores the airline's `name` and `country` of registration as its properties.
- `Airport` - representing a unique airport. Each node stores the airport's `name`, `city`, and `country` as properties.

##### Relationships

- `OPERATES` - this relationship connects an `Airline` node to a departure `Airport` node, representing that the airline operates a route out of that airport. The `plane_name` property is stored on this relationship as it describes the aircraft used for that specific service, which cannot be attributed to either the airline or airport alone.
  `ROUTE` - this relationship connects a departure `Airport` node to an arrival `Airport` node, representing that a direct connection exists between the two airports. This relationship carries no properties as it solely captures airport connectivity.

## 2.2. Arrows App Diagram

![Graph Database Design](./assets/database_design.png)
**Figure 1**. Graph Database Design

## 2.3. Design Choices and Discussion

##### 1. Not having a separate `Location` node

The dataset only provides city and country names as plain text values with no additional attributes such as continent, population, or country code. Creating a dedicated `Location` node would only add unnecessary complexity to the graph without providing meaningful additional value for querying.

Therefore, `country` was retained as a simple property on both `Airline` and `Airport` nodes. The downside is that location-based queries ****rely on property matching (ex: `WHERE a.country = 'Australia'`) rather than relationship traversal, which makes the graph looking very simple but functionally equivalent for the dataset.

#### 2. Having a separate `ROUTE` relationship instead of just `OPERATES`

`OPERATES` only connects `Airline -> Airport`. It will only tell us which airline flies out of which airport. But specifically for query `e` (flight between Beijing and Perth), we will need an `Airport -> Airport` relationship. Without `ROUTE`, Cypher won't be able to jump between airports directly. It would have to go through airline nodes every time.

##### 3. Putting `plane_name` property on `OPERATES`

`plane_name` is stored as a property on the `OPERATES` relationship as it is required for query `d`, which involves counting distinct aircraft types per airport pair. It is not store on `Airline` or `Airport` because it describes the specific service operated between an airline and an airport, and can't be meaningfully attributed to either entity independently.

# 3. ETL Process

## 3.1. Dataset Overview

### 3.1.1. Data description by columns

![Code snippet of df.columns](./assets/column_names_check.png)
Based on the snippet above, we can see that the dataset contains information about airline routes and the aircraft used to operate them. Each row of the dataset represents a specific route that the airline operates on, detailing the departure and arrival airports, the country each airport is lcoated in, and the aircraft types used for that route. Additional information includes the country in whicih the airline is based and the city of each airport.

### 3.1.2. Data structure

![Code snippet of df.shape](./assets/structure_check.png)

Based on the snippet above we can see that the dataset has 9 columns with 57,301rows of observations.

### 3.1.3. Immediate Observations

![](assets/null_check.png)

The dataset contains no `null` values. Which means all columns and all rows contains a value.

![](assets/dup_check.png)

The dataset contains no duplicated rows, which means all records are unique.

![](./assets/first_five.png)Inspecting the first five rows reveals that the `Plane Name` column contains `;` values, indicating that a single route may be operated using multiple aircraft types. This would require special handling during ETL to correctly extract individual plane types for query `d`.

## 3.2. Data Cleaning

Based on the result of the initial inspections, the raw datasets are already clean of missing values and duplicates. However, additional cleaning checks were performed to ensure data consistency before generating the node and relationship CSVs.

### 3.2.1. Whitespace Check

![](assets/whitespace_check.png)

The whitespace check revealed that `Departure Airport City` and `Arrival Airport City` contained 4 and 5 values that contain leading or trailing whitespace respectively. These were then corrected by applying `str.strip()` to both columns.

![](assets/whitespace_trim.png)

### 3.2.2. Case Inconsistencies Check

![](assets/inconsistencies_check.png)

Case inconsistency checks on key columns such as `Airline Name`, `Airline Country`, `Departure Airport Name`, `Arrival Airport Name`, `Departure Airport Country/Region`, and `Arrival Airport Country/Region` resulted in no issues found. No further transformation was required.

With these cleaning steps done, the dataset was exported and ready to be used for node and relationship CSV generation.

## 3.3. Node CSV Generation

The nodes generated from the raw datasets are `Airlines` and `Airports`. Before the nodes were created, the previously cleaned dataset was read again.

![](assets/read_cleaned_data.png)

### 3.3.1. `Airlines` Node

`Airlines` node was designed to have information about the airline's name and the country it is based in. With that, the columns used to build the node are `Airline Name` and `Airline Country`.

![](assets/airline_node.png)

- `drop_duplicates()` is used to remove duplicates of `Airline Name-Airline Country` combination.
- `reset_index(drop=True)` is used to reset the sequential order after removing the duplicates. The `drop=True` ensures that the old index is discarded rather than added as an extra column.

### 3.3.2. `Airports` Node

`Airport` node was designed to have information of an airport's name and its geographical location, such as the city and country it's located in. In order to make sure we have all the airports in the dataset, we will be combining the departure airports and the arrival airports.

![](assets/airport_node.png)

- `dep_airport` and `arr_airport` are created separately by extracting departure and arrival airport columns respectively, then renaming them to a consistent schema (name, city, country).
- `pd.concat()` combines both DataFrames into one since airports appear on both sides of the dataset. An airport can be a departure airport in one row and an arrival airport in another.
- `drop_duplicates()` ensures each unique airport only appears once in the final CSV.
- reset_index(drop=True) resets the index to a clean sequential
  order after concatenation and deduplication.

## 3.4. Relationship CSV Generation

The relationships developed from the raw dataset are: `ROUTES` and `OPERATES`.

### 3.4.1. `ROUTES` Relationship

`ROUTES` relationship was designed to capture the information about the route of a flight. From `Departure Airport Name` to `Arrival Airport Name`.

![](assets/routes_rel.png)

- `.drop_duplicates()` is used the resulting combinations of departure and arrival airports will have resulted in thousands of duplicated rows. This is because multiple airlines and planes can have the same flight routes.

### 3.4.2. `OPERATES` Relationship

`OPERATES` relationship was designed to contain information about the routes an airlines operates on and which of their planes are used for that specific service.

![](assets/operates_rel.png)

- `drop_duplicates()` removes duplicate airline-route-plane combinations since the same service can appear multiple times in the raw dataset.
- `plane_name` is retained as it is required for query `d`, which involves counting distinct aircraft types per airport pair.

# 4. Graph Database Implementation

## 4.1. Neo4j Import and Load CSV

### 4.1.1. Import SV

The generated CSV filese were copied into Neo4j `import` directory located at the Path shown in the picture below. Neo4j requires the files to be placed in the designated folder in order to be accessed by Cypher.

![](assets/import_csv.png)

### 4.1.2 Load CSV

Below are the Cypher commands to load the CSVs for the nodes and relationships:

###### `Airlines` Node

![](assets/create_airlines.png)

###### `Airports` Node

![](assets/create_airports.png)

###### `ROUTE` Relationship

![](assets/create_route.png)

###### `OPERATES` Relationship

![](assets/create_operates.png)

The `LOAD CSV WITH HEADERS` was used to load the CSVs, which reads each row of a CSV file and maps the values to node properties or relationship attributes.

`MERGE` was used instead of `CREATE` when importing nodes to prevent duplicate nodes from being created. `CREATE` inserts a new node regardless of whether it already exists, whereas `MERGE` will check first if a node with the specified properties already exist in the database. If it does, it matches the existing one. If it doesn't it creates a new one.

## 4.2. Database Statistics

After importing and loading the CSV files for the nodes and realtionships, the graph database was verified to contain the expected the number of nodes and relationships. As shown in the screenshots below, the database contains 2,783 nodes across the two labels, with 488 nodes for `Airline` and 2,795 nodes for `Airport`, and 63,223 relationships across two types, with 30,735 for `OPERATES` and 32,488 for `ROUTE`

###### Node

![](assets/node_stats.png)

###### Relationship

![](assets/rel_stats.png)

# 5. Cypher Queries

In this section, we will be writing the Cypher queries to answer the given 6 queries.

## 5.1. Query 1

> List all distinct airline names where the airline's country is Australia

The first query requires us to find the distinct names of the airlines where the country they're based in is Austalia

![](assets/query1.png)

This query matches all `Airline` nodes where the `country` property is `Australia` and returns their distinct names. The result returned 3 Australian airlines:

- Whyalla Airlines
- Transpac Express
- Transaustralian Air Express

## 5.2. Query 2

> "How many route records are domestic, and how many are international? A route is domestic if the departure and arrival airports are in the same country/region."

### 5.2.1. Domestic Routes

![](assets/query2_domestic.png)

This query traverses the `ROUTE` relationship between two `Airport` nodes. Since each `Airport` node stores a `country` property, we can compare the departure and arrival airport countries diectly.

`WHERE dep.country = arr.country` filters the result to show only domestic routes, which returned **14,092 routes**.

### 5.2.2. International Routes

![](assets/query2_inter.png)

This query traverses the `ROUTE` relationship between two `Airport` nodes. Since each `Airport` node stores a `country` property, we can compare the departure and arrival airport countries diectly.

WHERE dep.country <> arr.country` filters the result to show only international routes, which returned **18,396 routes**.

## 5.3. Query 3

> "Find the airport pair with the greatest number of records. Treat A→B and B→A as the same airport pair."

![](assets/query3.png)

This query traverses the lines from `Airline` to departure airport via the`OPERATES` relationship, then connects to arrival airport via the `ROUTE` relationship. This query uses the `WITH` query to chain multiple queries together.

The `CASE WHEN` query compares the two airport names alphabetically, then put the "smaller" name as `airport` and the larger one as `airport2`. This ensures that Route `Perth -> Sydney` will always give the same result as `Sydney -> Perth`. `airport1` will be Perth and `airport2` will be Sydney.

The results are grouped into `airport1`, `airport2`, and `records` that counts all occurences, then have only the first record shown, which would have the highest number of records. In this case, it is the route between **Charles de Gaulle International Airport and Hartsfield Jackson Atlanta International Airport** with **702 records** .

## 5.4. Query 4

> Find the top 5 airport pairs that are served with the greatest number of distinct aircraft types across all rows and all airlines. Treat A→B and B→A as the same airport pair, and count distinct aircraft types across all rows for that pair.

![](assets/query4.png)

This query traverses the path from `Airline` to departure airport, via the `OPERATES` relationship, to arrival airport, via the `ROUTE` relationship. Plane information is stored in `OPERATES`, so we will access it and use `SPLIT` to turn them into a list, separated by `;`.  `UNWIND` was then used to turn each list elements as its own row, duplicating the other columns.

Like the previous query, `CASE WHEN` was used to ensure routes `A -> B` is treated the same as `B -> A`. `TRIM(plane)` to remove any trailing white spaces from the plane list.

Finally, we set the columns, and used `COUNT(DISTINCT plane)` to count unique values and then count the number plane types for each airport pair. This will return a number, which we will then sort `DESC` and take the top 5 result.

## 5.5. Query 5

> Find all possible travel routes from Beijing Capital International Airport to Perth International Airport where at most 3 hops (i.e., at most 3 ROUTE relationships) are traversed. How many such distinct routes exist?

![](assets/query5.png)

This query uses a varying-length path traversal to find all possible routes between Beijing Capital International Airport and Perth International Airport. `[:ROUTE*1..3]` instructs neo4j to follow up to 3 `ROUTE` relationships to reach the destination. Each unique sequence of airports visited is treated as a distinct route. The query returned **650 distinct routes**.

## 5.6. Query 6

> Find the top 5 pairs of airlines that compete head-to-head on the greatest number of shared routes. Two airlines are considered competitors if they both operate between the same two airports, regardless of direction. Return the airline pair names and the number of routes they share.

![](assets/query6.png)

This query identifies the top 5 pairs of competing airlines based on the number of shared routes. SAM Columbia and Zantom International Airlines has the lead, with 793 shared routes, significantly more than the second through fifth pairs. Sham Wing Airlines and Sheremetyevo-Cargo both appear twice in the top 5, indicating that they are highly competitive carriers operating across many shared routes with multiple airlines.

# 6. Self-Designed Queries

## 6.1. Self Query 1

> Which airport has the most IN and OUT `ROUTE` relationships?

![](assets/self_query1.png)

This self query was designed to find which airport has the most connections with other airports, both incoming and outgoing routes. The undirected relationship pattern `-[:ROUTE]-` traverses the `ROUTE` relationship in both directions, making sure an airport is counted regardless of whether it's the departure or arrival airport. `COUNT(DISTINCT other)` makes sure each connected airport is only counted once.

Based on this query, **Charles de Gaulle International Airport**[](https://) has the most connections with other aiports at **238 connections**, with **Istanbul Airport**and **Frankfurt am Main Airport** coming in close behind at both above 230 connections.

## 6.2. Self Query 2

> Find the numebr of distinct flight paths originating from Perth International Airport within 1 to 3 hops, categorized by domestic (paths that stay within Australia) and international (paths tha pass through at least one non-Australian airport).

![](assets/self_query2.png)

This query uses APOC's `apoc.path.expand` to do advanced path traversal from Perth International Airport. This method takes:

- the start node,
- a relationship filter (`ROUTE>` for outgoing ROUTE relationships),
- a label filter (`+Airport` to whitelist Airport node),
- and a minimum and maximum hop range (1 to 3)

The `YIELD path` clause captures every path found.

A `CASE WHEN ALL(...)` expression the categorizes each path. If all airports along the path is in Austrlia, the path is classified as `Domestic`, otherwise it will be `International`. The query groups by `flight_type` and `hops` to count the number of paths in each category.

**Result:**

This result show a clear contrast between domestic and international connectivity from Perth. Direct flights (1 hop) include 19 domestic and 16 international destinations. This indicates Perth's role as a major Australian gateway. At 2 hops, international paths grow signficantly from 134 domestic to 1,317 international, reflecting Perth's strong global reach. At 3 hops, international paths rose from 1,172 domestic to 76,261 international, which could be the result of compounding network effects through major global transits such as Singapore, Dubai, and Doha.

# 7. Graph Data Science Application

## 7.1. Application: Identifying Strategic Hub Airports

A practical application of this airline graph database is the identification of strategically important airports within the global air transport network. Airlines, airport authorities, and aviation regulators frequently need to assess which airports play the most critical in the network based on volume of flights and by how essential the airports are to overall connectivity. Identifying these hubs can support the people responsible in makin decisions around infrastructure investment, route planning, security prioritization, and disruption analysis [1].

## 7.2. Algorithm: PageRank

The most suitable algorithm for hub identification is **PageRank**. Originally developed by Larry Page and Sergey Brin to rank Google search results [2]. The main idea is that a node (an airport) is considered important if many *other important nodes* connect to it. So an airport isn't just ranked by how many flights it receives but by *who* it receives flights from.

If we apply this algorithm to the airline graph, each `Airport` would be a node and each `ROUTE` relationship would be a directed edge. The algorithm would then calculate a score. The algorithm would then calculate a score for every airport based on the number and importance of incoming routes. An airport that receives flights from many big hubs would score higher than airport that receives the same number of flights from small regional airports.

This is more useful than just counting connections, which is what the self-designed query in this report did. For example, there might be two airports that both have 100 incoming flights, but one of them was connected to major global airports while the other is connected to small regional airports. **PageRank captures this difference by weighing the importance of connections, not just counting them**.

In summary, PageRank fits airline networks well because:

- Flights have a clear direction (departure -> arrival)m which the algorithm uses
- Being connected to a major hub matters more than being connected to a small airport
- It scales well to large networks with thousands of airports [3]

## 7.3. Other Suitable Algorithms

# 8. References

# 9. Appendix - AI Usage
