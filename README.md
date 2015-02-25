# gitHub-WebRTC-analysis
An analysis of WebRTC trends on GitHub. See [webrtcHacks](https://webrtchacks.com/github-analysis) for more details

## WebRTC Extract from public GitHub timeline data on BigQuery
```
/* Pull interesting fields from pushes that have WebRTC related terms in the commit or repo description*/
SELECT
 repository_name, 
 repository_url, 
 repository_created_at AS created, 
 repository_organization, 
 repository_language, 
 repository_description, 
 repository_has_downloads, 
 repository_forks, 
 repository_fork, 
 repository_pushed_at, 
 repository_homepage, 
 repository_watchers, 
 payload_commit_msg, 
 actor_attributes_login, 
 actor_attributes_name, 
 actor_attributes_company
FROM [githubarchive:github.timeline]
WHERE type="PushEvent"
 AND (
 /*Check the repo name */ 
 LOWER(repository_name) CONTAINS "webrtc" OR
 LOWER(repository_name) CONTAINS "getusermedia" OR
 LOWER(repository_name) CONTAINS "peerconnection" OR
 LOWER(repository_name) CONTAINS "datachannel" OR 
 /*See if any keywords in the commit message */ 
 LOWER(payload_commit_msg) CONTAINS "webrtc" OR
 LOWER(payload_commit_msg) CONTAINS "getusermedia" OR
 LOWER(payload_commit_msg) CONTAINS "peerconnection" OR
 LOWER(payload_commit_msg) CONTAINS "datachannel" OR
 /*And check the repo description */ 
 LOWER(repository_description) CONTAINS "webrtc" OR
 LOWER(repository_description) CONTAINS "getusermedia" OR
 LOWER(repository_description) CONTAINS "peerconnection" OR
 LOWER(repository_description) CONTAINS "datachannel"
 )
 AND YEAR(repository_pushed_at) < 2015 /*2015 data has different schema*/
```

## Extract unique repos
```
/*extracts unique repos and selected fields */
SELECT 
 /*grab all the data in the subquery and perform some summary functions */
 *,
 REGEXP_EXTRACT(repo, '(.*)/{1}') AS owner, /*get the owner/account/org/top-level repo*/
 IF( (forks + watchers + contributors) > 2, FALSE, TRUE) AS solos, /*my measure of group activity*/
 DATEDIFF(firstWebrtc, createDate) as daysB4webrtc, /*for data checking */
 IF( MONTH(firstWebrtc) >= 10, 
  CONCAT( STRING(YEAR(firstWebrtc)), '-', STRING(MONTH(firstWebrtc)) ), 
  CONCAT( STRING(YEAR(firstWebrtc)), '-0', STRING(MONTH(firstWebrtc)) )
  ) AS yearMonth /*to help with data summarization*/
FROM (
 /*Grab the key data*/
 SELECT
  SUBSTR(repository_url, 20) AS repo,
  MIN(created) AS createDate,
  MIN(repository_pushed_at) AS firstWebrtc, /*WebRTC 'startDate*/
  MAX(repository_fork) AS fork, /*mark TRUE if it is ever true */
  MAX(repository_forks) AS forks, 
  MAX(repository_watchers) AS watchers, 
  COUNT( distinct actor_attributes_login) AS contributors,
  LAST( repository_language) AS language,
 FROM githubWebrtc.webrtcPushData
 GROUP BY repo
 ORDER BY createDate, fork, forks, watchers, language, contributors
 )
```

## New Owner
```
/*extracts unique owners and selected fields */
SELECT
 /*outer query to help with data summarization*/ 
 *,
 YEAR(createDate) AS startYear,
 MONTH(createDate) AS startMonth,
 IF( MONTH(createDate) >= 10, 
  CONCAT( STRING(YEAR(createDate)), '-', STRING(MONTH(createDate)) ), 
  CONCAT( STRING(YEAR(createDate)), '-0', STRING(MONTH(createDate)) )
  ) AS yearMonth,
FROM (
 SELECT
  REGEXP_EXTRACT(repository_url, 'github.com/(.*)/{1}') AS owner,
  COUNT(DISTINCT repository_url) as repos,
  MIN(repository_pushed_at) AS createDate, /*WebRTC 'startDate*/ 
  SUM(repository_forks) AS forks, 
  COUNT( distinct actor_attributes_login) AS contributors,
 FROM githubWebrtc.webrtcPushes
 GROUP BY owner, repository_url
 ORDER BY repos DESC, createDate, forks, contributors
 )
```

## New Contributors
```
/*extracts WebRTC contributors */
SELECT
 /*outer query to help with data summarization*/ 
 *,
 YEAR(firstPush) AS startYear,
 MONTH(firstPush) AS startMonth,
 IF( MONTH(firstPush) >= 10, 
  CONCAT( STRING(YEAR(firstPush)), '-', STRING(MONTH(firstPush)) ), 
  CONCAT( STRING(YEAR(firstPush)), '-0', STRING(MONTH(firstPush)) )
  ) AS yearMonth,
FROM (
 SELECT 
  actor_attributes_login AS user, 
  FIRST( actor_attributes_name) AS name, 
  FIRST( actor_attributes_company) AS company, 
  COUNT( distinct repository_url ) AS repos, 
  MIN( repository_pushed_at) AS firstPush, 
  MAX( repository_pushed_at) AS lastPush, 
  COUNT(distinct DATE(repository_pushed_at)) AS daysPushed, 
  DATEDIFF(CURRENT_DATE(), MAX( repository_pushed_at)) AS daysSincePush
  FROM githubWebrtc.webrtcPushes
  GROUP BY user
  ORDER BY daysPushed DESC, repos DESC, name, company, firstPush, lastPush, daysSincePush
 )
```

## Monthly Activity
### repos
```
/*Find how many repos have pushes in a given month */
SELECT 
 PushDate,
 'repos' as Type,
 COUNT(DISTINCT repo) as count
FROM (
 SELECT 
  SUBSTR(repository_url, 20) AS repo,
  IF( MONTH(repository_pushed_at) >= 10, 
   CONCAT( STRING(YEAR(repository_pushed_at)), '-', STRING(MONTH(repository_pushed_at)) ), 
   CONCAT( STRING(YEAR(repository_pushed_at)), '-0', STRING(MONTH(repository_pushed_at)) )
   ) AS PushDate,
 FROM githubWebrtc.webrtcPushes
 GROUP BY repo, PushDate)
GROUP BY PushDate
ORDER BY PushDate ASC
```
### owners
```
/*Find how many owners have pushes each month */
SELECT 
 PushDate,
 'owners' as type,
 COUNT(DISTINCT owner) as count
FROM (
 SELECT 
  REGEXP_EXTRACT(repository_url, 'com/(.*)/{1}') AS owner,
  IF( MONTH(repository_pushed_at) >= 10, 
   CONCAT( STRING(YEAR(repository_pushed_at)), '-', STRING(MONTH(repository_pushed_at)) ), 
   CONCAT( STRING(YEAR(repository_pushed_at)), '-0', STRING(MONTH(repository_pushed_at)) )
   ) AS PushDate,
 FROM githubWebrtc.webrtcPushes
 GROUP BY owner, PushDate)
GROUP BY PushDate
ORDER BY PushDate ASC
```
### contributors
```
/*Find how many users make pushes each day */
SELECT 
 PushDate,
 'contributors' AS type,
 COUNT(DISTINCT user) as count
FROM (
 SELECT 
  actor_attributes_login AS user, 
  IF( MONTH(repository_pushed_at) >= 10, 
   CONCAT( STRING(YEAR(repository_pushed_at)), '-', STRING(MONTH(repository_pushed_at)) ), 
   CONCAT( STRING(YEAR(repository_pushed_at)), '-0', STRING(MONTH(repository_pushed_at)) )
   ) AS PushDate,
 FROM githubWebrtc.webrtcPushes
 GROUP BY user, PushDate)
GROUP BY PushDate
ORDER BY PushDate ASC
```


