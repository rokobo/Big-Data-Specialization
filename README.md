# Catch the Pink Flamingo game analytics using Big Data

- [Catch the Pink Flamingo game analytics using Big Data](#catch-the-pink-flamingo-game-analytics-using-big-data)
  - [Data Exploration](#data-exploration)
    - [Data Set Overview](#data-set-overview)
    - [Aggregations on Splunk](#aggregations-on-splunk)
    - [Visualizations on Splunk](#visualizations-on-splunk)
  - [Classification model in Knime for big spenders](#classification-model-in-knime-for-big-spenders)
  - [K-means clustering in Spark](#k-means-clustering-in-spark)

The game's data model is shown in the following entity relationship diagram (ERD):

<img src="images/ERD.png?raw=true" alt="ERD" width="50%"/>

- Each user is a member of at most one team. When a new user starts playing the game, they are on a team by themselves for the first level and may join a team on subsequent levels.

- When a user is in a team, playing, they have a unique user session that starts when they start playing and ends when they stop playing.

- Whenever a user is in a team, they will have a team-assignment recorded when they first joined the team.

- Whenever the user is in a team, they can level up whenever the team finishes playing a level. When they level up, two level-up events are recorded for the team, one for the end of the previous level and the other for the start of the current one. At the same time, all the users who are playing would end their current session, record it, and start new sessions with updated team_level values.

## Data Exploration

Data exploration was done in a Splunk docker container with directory monitor on `/data` and the following docker `run` command:

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

    <img src="images/purchases-per-item.png?raw=true" alt="purchases-per-item" width="75%"/>

- A histogram showing how much money was made from each item:

    ```spl
    source="/data/buy-clicks.csv" | stats sum(price) as Revenue by buyId
    ```

    <img src="images/revenue-per-item.png?raw=true" alt="revenue-per-item" width="75%"/>

- A histogram showing total amount of money spent by the top ten users (ranked by how much money they spent).

    ```spl
    source="/data/buy-clicks.csv" | stats sum(price) as "Revenue" by userId | sort - "Revenue" | head 10
    ```

    <img src="images/revenue-per-user-top10.png?raw=true" alt="revenue-per-user-top10" width="75%"/>

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

<img src="images/knime-workflow.svg?raw=true" alt="knime-workflow.svg" width="75%"/>

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

<img src="images/knime-decision-tree.svg?raw=true" alt="knime-decision-tree.svg" width="75%"/>

The overall accuracy of the model was:

| consumer_type | PennyPinchers | HighRollers |
| --- | --- | --- |
| PennyPinchers | 285 | 10 |
| HighRollers | 63 | 207 |

*Accuracy: 87.08%, Cohen`s kappa: 0.739%*

### Recommendations

From the analysis of the decision tree, the most reliant metric to determine if a user is a high spender or low spender is the platform type of the user. My recommendations to increase revenue include:

- <u>Increase</u> the quantity, price and exclusives of the item market in the IOS platform.
- <u>Increase</u> ad spending for IOS devices.
- <u>Eliminate</u> ad spending for Linux users and <u>decrease</u> ad spending for Android, Windows and Mac users.

## K-means clustering in Spark

The K-means clustering was done in PySpark. The code can be seen in the notebook `kmeans-clustering.ipynb`. The features selected for this analysis were:

- `User spending`: Enables the interpretation of each cluster based on this metric, clusters are sorted based on this feature for easier interpretation by segmenting users into spending tiers.
- `Team strength` and `Team level`: Might provide insight into how a user's community affects their spending patterns.
- `Hit average`: Might provide insight into how a user's proviciency and game mechanisms affect their spending patterns.
- `Ad clicks`: Might provide insight into how user targeted ads affects their spending patterns.
- `Platform type`: Might provide insight into how a user's platform type affects their spending patterns.

The analysis can be customized by changing the number of clusters. The results can then be visualized in a bar graph that compares each cluster by feature:


<p align="center">
<img src="images/spark-clustering-2.png?raw=true" alt="spark-clustering-2.png" width="70%"/>
</p>

Looking at this initial  graph with 2 clusters, we can note the followind:

- Cluster 1 is associated with users who spend a lot of money.
- Platform type seems to be highly correlated to how much a user spends.
  - Iphone users are more correlated with big spenders.
  - Android, Linux, Windows and Mac users are more correlated with small spenders.
- Users on weaker teams have a slightly higher tendency to spend more money.
- Users with higher hit average and higher ad clicks have a slightly higher tendency to spend more money.

<p align="center">
<img src="images/spark-clustering-5.png?raw=true" alt="spark-clustering-5.png" width="70%"/>
</p>

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

- <u>Increase</u> the quantity, price and exclusives of the item market in the IOS platform.
- <u>Increase</u> ad spending for IOS devices.
- <u>Eliminate</u> ad spending for Linux users and <u>decrease</u> ad spending for Android, Windows and Mac users.
- <u>Increase</u> ad spending for users with high hit averages, <u>decrease</u> ad spending for users with low hit averages.
- <u>Develop</u> new game mechanisms that incentivize high hit averages and continued engagement.
- <u>Increase</u> ad spending for users in low to mid strength teams and <u>decrease</u> ad spending for users in high-strength teams.
- <u>Increase</u> ad spending progressively for users with high spending.
