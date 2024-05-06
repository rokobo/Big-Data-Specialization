# Catch the Pink Flamingo game analytics using Big Data

- [Catch the Pink Flamingo game analytics using Big Data](#catch-the-pink-flamingo-game-analytics-using-big-data)
  - [Data Exploration](#data-exploration)
    - [Data Set Overview](#data-set-overview)
    - [Aggregations on Splunk](#aggregations-on-splunk)
    - [Visualizations on Splunk](#visualizations-on-splunk)
  - [Classification model in Knime for big spenders](#classification-model-in-knime-for-big-spenders)
  - [K-means clustering in Spark](#k-means-clustering-in-spark)

Catch the Pink Flamingo is a cross-platform game that needs Big Data analysis to increase the game's revenue. The game's data model is shown in the following entity relationship diagram (ERD):

<img src="images/ERD.png?raw=true" alt="ERD" width="75%"/>

- Each user is a member of at most one team. When a new user starts playing the game, they are on a team by themselves for the first level and may join a team on subsequent levels.

- When a user is in a team, playing, they have a unique user session that starts when they start playing and ends when they stop playing.

- Whenever a user is in a team, they will have a team-assignment recorded when they first joined the team.

- Whenever the user is in a team, they can level up whenever the team finishes playing a level. When they level up, two level-up events are recorded for the team, one for the end of the previous level and the other for the start of the current one. At the same time, all the users who are playing would end their current session, record it, and start new sessions with updated team_level values.

## Data Exploration

Data exploration was done in a Splunk Docker container with directory monitor on `/data` and the following Docker `run` command:

```bash
docker run -d \
    --publish=8000:8000 --publish=8089:8089 \
    --env='SPLUNK_START_ARGS=--accept-license' \
    --env='SPLUNK_PASSWORD=password' \
    --volume=$(pwd)/data:/data \
    --name splunk splunk/splunk:latest
```

### Data Set Overview

The table below lists each of the files available for analysis with a short description of what is found in each one:

| File name | Description | Fields |
| --- | --- | --- |
| `ad-clicks.csv` | Player clicks on an advertisement | - `timestamp`: When the click occurred <br> - `txId`: A unique id (within ad-clicks.log) for the click <br> - `userSessionId`: Id of the user session for the user who made the click <br> - `teamid`: The current team id of the user who made the click <br> - `userId`: The user id of the user who made the click <br> - `adId`: The id of the ad clicked on <br> - `adCategory`: The category/type of ad clicked on
| `buy-clicks.csv` | Player in-app purchase | - `timestamp`: When the purchase was made <br> - `txId`: A unique id (within buy-clicks.log) for the purchase <br> - `userSessionId`: The id of the user session for the user who made the purchase <br> - `team`: The current team id of the user who made the purchase <br> - `userId`: The user id of the user who made the purchase <br> - `buyId`: The id of the item purchased <br> - `price`: The price of the item purchased
| `users.csv` | Users playing the game. | - `timestamp`: When user first played the game <br> - `userId`: The user id assigned to the user <br> - `nick`: The nickname chosen by the user <br> - `twitter`: The twitter handle of the user <br> - `dob`: The date of birth of the user <br> - `country`: The two-letter country code where the user lives
| `team.csv` | Teams terminated in the game. | - `teamid`: The id of the team <br> - `name`: The name of the team <br> - `teamCreationTime`: The timestamp when the team was created <br> - `teamEndTime`: The timestamp when the last member left the team <br> - `strength`: A measure of team strength <br> - `currentLevel`: The current level of the team
| `team-assignments.csv` | User team assignment (1 per user) | - `timestamp`: When the user joined the team <br> - `team`: The id of the team <br> - `userId`: The id of the user <br> - `assignmentId`: A unique id for this assignment
| `level-events.csv` | Events when a team starts/finishes level | - `timestamp`: When the event occurred <br> - `eventId`: A unique id for the event <br> - `teamid`: The id of the team <br> - `teamLevel`: The level started or completed <br> - `eventType`: The type of event, either start or end
| `user-session.csv` | User sessions | - `timestamp`: A `timestamp` denoting when the event occurred.  <br> - `userSessionId`: A unique id for the session <br> - `userId`: The current user's ID <br> - `teamid`: The current user's team <br> - `assignmentId`: The team assignment id for the user to the team <br> - `sessionType`: Whether the event is the start or end of a session <br> - `teamLevel`: The level of the team during this session.  <br> - `platformType`: The type of platform of the user during this session
| `game-clicks.csv` | User clicks in game | - `timestamp`: When the click occurred <br> - `clickId`: A unique id for the click <br> - `userId`: The id of the user performing the click <br> - `userSessionId`: The id of the session of the user when the click is performed <br> - `isHit`: Denotes if the click was on a flamingo `1` or missed the flamingo `0` <br> - `teamid`: The id of the team of the user <br> - `teamLevel`: The current level of the team of the user

### Aggregations on Splunk

| Query | Result | Splunk code |
| ----- | ------ | ---- |
| Amount spent buying items | $21407 | source="/data/buy-clicks.csv" \| stats sum(price) |
| Number of unique items available to be purchased | 6 | source="/data/buy-clicks.csv" \| stats dc(buyId) |

### Visualizations on Splunk

- A histogram showing how many times each item is purchased:

    ```spl
    source="/data/buy-clicks.csv" | stats count by buyId
    ```

    <img src="images/purchases-per-item.png?raw=true" alt="purchases-per-item" width="100%"/>

- A histogram showing how much money was made from each item:

    ```spl
    source="/data/buy-clicks.csv" | stats sum(price) as Revenue by buyId
    ```

    <img src="images/revenue-per-item.png?raw=true" alt="revenue-per-item" width="100%"/>

- A histogram showing total amount of money spent by the top ten users (ranked by how much money they spent).

    ```spl
    source="/data/buy-clicks.csv" | stats sum(price) as "Revenue" by userId | sort - "Revenue" | head 10
    ```

    <img src="images/revenue-per-user-top10.png?raw=true" alt="revenue-per-user-top10" width="100%"/>

- user id, platform, and hit-ratio percentage for the top three buying users:

    ```spl
    source="/data/buy-clicks.csv"
    | stats sum(price) as revenue by userId
    | sort - revenue
    | head 3
    | streamstats count as Rank
    | fields userId, Rank, revenue
    | join type=inner userId [
        search source="/data/user-session.csv"
        | fields userId, platformType]
    | join type=inner userId [
        search source="/data/game-clicks.csv"
        | stats avg(isHit) as hit_ratio by userId
        | eval hit_ratio = sigfig(hit_ratio * 100.0)]
    | sort Rank
    | table Rank, userId, platformType, hit_ratio
    ```

    | Rank | UserId | Platform | Hit-ratio (%) |
    | ---- | ------ | -------- | -------------- |
    | 1 | 2229 | iphone | 11.60 |
    | 2 | 12 | iphone | 13.07 |
    | 3 | 471 | iphone | 14.50 |

## Classification model in Knime for big spenders

The classification Knime workflow can be imported to Knime via the `big_spenders.knwf` file. The workflow design is as follows:

<img src="images/knime-workflow.svg?raw=true" alt="knime-workflow.svg" width="100%"/>

This workflow was desined to categorize users into `HighRollers` (average item purchase > $5.00) and `PennyPinchers` (average item purchase < $5.00). The steps taken are as follows:

1. Import the data from `combined-data.csv`.
2. Transform the `count_buyId` column into an integer column and the `avg_price` column into a double column.
3. Filter missing values, from users who did not make a single purchase.
4. Create the two categories mentioned above.
5. Remove unnecessary columns.
    - `avg_price` was removed since the categories are derived from this column.
    - `userId` was removed since it has no statistical relevance.
    - `userSessionId` was removed since it has no statistical relevance.
6. Parition the data into training set and test set.
7. Train a decision tree model using the training set.
8. Evaluate the model using the test set.

<img src="images/knime-decision-tree.svg?raw=true" alt="knime-decision-tree.svg" width="100%"/>

The overall accuracy of the model was:

| consumer_type | PennyPinchers | HighRollers |
| --- | --- | --- |
| PennyPinchers | 285 | 10 |
| HighRollers | 63 | 207 |

*Accuracy: 87.08%, Cohen`s kappa: 0.739%*

### Recommendations

From the analysis of the decision tree, the most reliant metric to determine if a user is a high spender or low spender is the platform type of the user. My recommendations to increase revenue include:

- **Increase** the quantity, price and exclusives of the item market in the IOS platform.
- **Increase** ad spending for IOS devices.
- **Eliminate** ad spending for Linux users and **decrease** ad spending for Android, Windows and Mac users.

## K-means clustering in Spark

The K-means clustering was done in PySpark. The code can be seen in the notebook `kmeans-clustering.ipynb`. The features selected for this analysis were:

- `User spending`: Enables the interpretation of each cluster based on this metric, clusters are sorted based on this feature for easier interpretation by segmenting users into spending tiers.
- `Team strength` and `Team level`: Might provide insight into how a user's community affects their spending patterns.
- `Hit average`: Might provide insight into how a user's proviciency and game mechanisms affect their spending patterns.
- `Ad clicks`: Might provide insight into how user targeted ads affects their spending patterns.
- `Platform type`: Might provide insight into how a user's platform type affects their spending patterns.

The analysis can be customized by changing the number of clusters. The results can then be visualized in a bar graph that compares each cluster by feature:


<img src="images/spark-clustering-2.png?raw=true" alt="spark-clustering-2.png" width="100%"/>

Looking at this initial  graph with 2 clusters, we can note the followind:

- Cluster 1 is associated with users who spend a lot of money.
- Platform type seems to be highly correlated to how much a user spends.
  - Iphone users are more correlated with big spenders.
  - Android, Linux, Windows and Mac users are more correlated with small spenders.
- Users on weaker teams have a slightly higher tendency to spend more money.
- Users with higher hit average and higher ad clicks have a slightly higher tendency to spend more money.

<img src="images/spark-clustering-5.png?raw=true" alt="spark-clustering-5.png" width="100%"/>

This graph with 5 clusters provides a more detailed view of the patterns:

- Although Iphone users are the users who spend more money, we can now see more detail in the other platforms:
  - Linux is highly correlated with very low spending.
  - Windows shows slightly more spending than Linux.
  - Android and Mac show slightly more spending than Windows, even showing correlation to the 3rd cluster, which is the cluster of medium spenders.
- Looking at team strength, we can postulate that increasing team strength is initially a boost in user spending, however, that boost quickly diminishes after the initial team strength levels.
- Team level shows no correlation to user spending.
- Hit average and ad clicks show a positive rising correlation with higher user spending.

### Recommendations

From the analysis of the K-Means model in Spark, my recommendations to increase revenue include:

- **Increase** the quantity, price and exclusives of the item market in the IOS platform.
- **Increase** ad spending for IOS devices.
- **Eliminate** ad spending for Linux users and **decrease** ad spending for Android, Windows and Mac users.
- **Increase** ad spending for users with high hit averages, **decrease** ad spending for users with low hit averages.
- **Develop** new game mechanisms that incentivize high hit averages and continued engagement.
- **Increase** ad spending for users in low to mid strength teams and **decrease** ad spending for users in high-strength teams.
- **Increase** ad spending progressively for users with high spending.
- **Use** the red clusters to find users to give discounts for in-game purchases.
- **Use** the green clusters to find users to increase the price for in-game purchases.

## Modelling chat data on Neo4j

The chat data was modeled in ordeer to capture user trends in chat data and recommend actions to the company. The modelling was done in Neo4j using a Docker container created by the following Docker run command:

```bash
docker run -d \
    --name neo4j-container \
    --publish=7474:7474 --publish=7687:7687 \
    --env=NEO4J_AUTH=none \
    --volume=$(pwd)/data-neo4j:/var/lib/neo4j/import \
    neo4j
```

And the following Cypher load script:

```cypher
CREATE CONSTRAINT FOR (u:User) REQUIRE u.id IS UNIQUE;
CREATE CONSTRAINT FOR (t:Team) REQUIRE t.id IS UNIQUE;
CREATE CONSTRAINT FOR (c:TeamChatSession) REQUIRE c.id IS UNIQUE;
CREATE CONSTRAINT FOR (i:ChatItem) REQUIRE i.id IS UNIQUE;

// Create team chat sessions nodes that are created by an user nodes and owned by team nodes
LOAD CSV FROM "file:///chat_create_team_chat.csv" AS row
MERGE (u:User {id: toInteger(row[0])})
MERGE (t:Team {id: toInteger(row[1])})
MERGE (c:TeamChatSession {id: toInteger(row[2])})
MERGE (u)-[:CreatesSession{timeStamp: row[3]}]->(c)
MERGE (c)-[:OwnedBy{timeStamp: row[3]}]->(t);

// Create user nodes that join team chat sessions nodes
LOAD CSV FROM "file:///chat_join_team_chat.csv" AS row
MERGE (u:User {id: toInteger(row[0])})
MERGE (c:TeamChatSession {id: toInteger(row[1])})
MERGE (u)-[:Joins{timeStamp: row[2]}]->(c);

// Create user nodes that leave team chat sessions nodes
LOAD CSV FROM "file:///chat_leave_team_chat.csv" AS row
MERGE (u:User {id: toInteger(row[0])})
MERGE (c:TeamChatSession {id: toInteger(row[1])})
MERGE (u)-[:Leaves{timeStamp: row[2]}]->(c);

// Create user nodes that create chat item nodes that are part of team chat session nodes
LOAD CSV FROM "file:///chat_item_team_chat.csv" AS row
MERGE (u:User {id: toInteger(row[0])})
MERGE (c:TeamChatSession {id: toInteger(row[1])})
MERGE (i:ChatItem {id: toInteger(row[2])})
MERGE (u)-[:CreateChat{}]->(i)
MERGE (i)-[:PartOf{timeStamp: row[3]}]->(c);

// Create user nodes that are mentioned by a chat item node
LOAD CSV FROM "file:///chat_mention_team_chat.csv" AS row
MERGE (i:ChatItem {id: toInteger(row[0])})
MERGE (u:User {id: toInteger(row[1])})
MERGE (i)-[:Mentioned{timeStamp: row[2]}]->(u);

// Create chat item nodes that are a response to another chat item node
LOAD CSV FROM "file:///chat_respond_team_chat.csv" AS row
MERGE (i1:ChatItem {id: toInteger(row[0])})
MERGE (i2:ChatItem {id: toInteger(row[1])})
MERGE (i1)-[:ResponseTo{timeStamp: row[2]}]->(i2);

// Create edge that represents a user interacting with another user, either through mentions or responses
MATCH (u1:User)-[:CreateChat]->(c:ChatItem)-[:Mentioned]->(u2:User)
WHERE u1 <> u2
MERGE (u1)-[:InteractsWith]->(u2);

MATCH (u1:User)-[:CreateChat]->(c:ChatItem)-[:ResponseTo]-(:ChatItem)<-[:CreateChat]-(u2: User)
WHERE u1 <> u2
MERGE (u1)-[:InteractsWith]->(u2);
```

Here is an example graph from `CALL db.schema.visualization()`:

<img src="images/neo4j-schema.svg?raw=true" alt="neo4j-schema.svg"/>


### Analysis of user activity on Neo4j

+ What are the characteristics of the top 10 chattiest users?

    ```cypher
    // Get top 10 chattiest teams
    MATCH (c:ChatItem)-[:PartOf]->(:TeamChatSession)-[:OwnedBy]->(t:Team)
    WITH t, COUNT(c) AS NumberOfTeamChatItems
    ORDER BY NumberOfTeamChatItems DESC
    LIMIT 10

    // Get top 10 chattiest users and check if they are in the top 10 teams
    WITH COLLECT(t) AS topTeams
    MATCH (u:User)-[r:CreateChat]-(c:ChatItem)-[:PartOf]->(:TeamChatSession)-[:OwnedBy]->(t:Team)
    WITH u, COUNT(c) AS ChatItemsCreated, t, topTeams
    ORDER BY ChatItemsCreated DESC
    LIMIT 10
    WITH u, ChatItemsCreated, t, t IN topTeams AS IsInTopChattiestTeams, COLLECT(u) as topChattiestUsers

    // Get the clustering coefficient of the users
    MATCH (u:User)-[r:InteractsWith]->(u2:User)
    WITH u, count(u2) as NeighborCount, collect(u2) as neighbors, ChatItemsCreated, t, IsInTopChattiestTeams
    UNWIND neighbors as neighbor
    MATCH (neighbor)-[r:InteractsWith]->(u2:User)
    WHERE u2 in neighbors
    WITH u, NeighborCount, count(r) AS SecondaryDegree, ChatItemsCreated, t, IsInTopChattiestTeams
    RETURN
        u.id AS UserID,
        ChatItemsCreated,
        t.id AS TeamID,
        IsInTopChattiestTeams,
        (1.0*SecondaryDegree)/(NeighborCount * (NeighborCount-1)) AS ClusteringCoefficient
    ```

    | UserID | NumberOfChatItems | TeamID | IsInTopChattiestTeams | ClusteringCoefficient |
    |--------|-------------------|--------|----------------------|-----------------------|
     | 394   | 115               | 63     | false                | 0.9166666666666666    |
     | 2067  | 111               | 7      | false                | 0.7678571428571429    |
     | 209   | 109               | 7      | false                | 0.9523809523809523    |
     | 1087  | 109               | 77     | false                | 0.7666666666666667    |
     | 554   | 107               | 181    | false                | 0.8095238095238095    |
     | 999   | 105               | 52     | true                 | 0.8194444444444444    |
     | 1627  | 105               | 7      | false                | 0.7678571428571429    |
     | 516   | 105               | 7      | false                | 0.9523809523809523    |
     | 668   | 104               | 89     | false                | 1.0                   |
     | 461   | 104               | 104    | false                | 1.0                   |

+ What are the longest conversation chains in the chat data using the `:ResponseTo` edge label?

    ```cypher
    // Match the longest paths of ResponseTo relationships
    MATCH path=()-[:ResponseTo*]->()
    WITH path, LENGTH(path) AS pathLength
    ORDER BY pathLength DESC
    LIMIT 10

    // Match auxiliary information for the path
    UNWIND NODES(path) AS chatItem
    MATCH (user:User)-[:CreateChat]->(chatItem)
    WITH pathLength + 1 as pathLength, COLLECT(DISTINCT user.id) AS UserIDs, nodes(path) as pathNodes
    RETURN pathLength, SIZE(UserIDs) as UserCount, UserIDs, pathNodes
    ```

    | maxPathLength | UserCount | UserIDs | pathNodes |
    | --- | --- | --- | --- |
    | 10 | 5 | [1514, 1192, 853, 1153, 1978] | ... |
    | 9  | 4 | [1192, 853, 1153, 1978]       | ... |
    | 9  | 5 | [1514, 1192, 853, 1153, 1978] | ... |
    | 8  | 4 | [1192, 1978, 853, 1153]       | ... |
    | 8  | 4 | [1192, 1978, 853, 1153]       | ... |
    | 8  | 4 | [1514, 1192, 853, 1153]       | ... |
    | 8  | 4 | [1192, 853, 1153, 1978]       | ... |
    | 8  | 4 | [1192, 853, 1153, 1978]       | ... |
    | 7  | 4 | [1978, 1192, 853, 1153]       | ... |
    | 7  | 4 | [1192, 853, 1153, 1978]       | ... |

    <img src="images/neo4j-longest-chat.svg?raw=true" alt="neo4j-longest-chat.svg"/>

### Recommendations

- **Increase** company engagement with top chattiest users, prioritizing those with high local clustering coefficients.
- **Implement** referral programs, beta testing and reward systems for chattier users with high cluster coefficinets.
- **Utilize** clusters with high clustering coefficients to:
  - **Improve** ads by providing a more personalized targetting mechanism.
  - **Improve** ads by testing different types of ads on members of different clusters in order to infer cluster interests.
- **Implement** a reward system for users involved in long conversation chains.
- **Implement** a influencer program for users in multiple long conversation chains.
