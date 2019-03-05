# DatabasesAssignment5

<h2>Introduction <g-emoji class="g-emoji" alias="page_with_curl" fallback-src="https://github.githubassets.com/images/icons/emoji/unicode/1f4c3.png">📃</g-emoji></h2>
We used vagrant and docker <g-emoji class="g-emoji" alias="whale2" fallback-src="https://github.githubassets.com/images/icons/emoji/unicode/1f40b.png">🐋</g-emoji> to create the database,and we used coffe.stackexchange data from stackexchange for this assignment "https://archive.org/download/stackexchange/coffee.stackexchange.com.7z". The solution for the assignment are made in MySQL workbench.  


<h2>Exercises</h2>
Exercise 1: 

```sql
DELIMITER $$
CREATE DEFINER=`root`@`%` PROCEDURE `denormalizeComments`(idPost int(11))
BEGIN
select postId, json_arrayagg(JSON_OBJECT('id', Id, 'postId', PostId, "score", Score, "text", Text, "creationDate", CreationDate, "userId", UserId)) as jsoncomments from comments where postId = idPost;
END$$
DELIMITER ;
```

Exercise 2:
```sql
DELIMITER $$
CREATE TRIGGER before_comment_update
after insert ON comments
 FOR EACH ROW
 BEGIN
CALL denormalizeComments(new.PostId);
END$$
DELIMITER ;
```
Exercise 3:
```sql
DELIMITER $$
CREATE DEFINER=`root`@`%` PROCEDURE `commentPost`(cId int(11),pId int(11), textM Text, uId int(11))
BEGIN
insert into comments(id, PostId, Score,Text,CreationDate,UserId)values(cId, pId, 0, textM, NOW(), uId);
update posts set commentCount = commentCount+1 where Id = pId;
call denormalizeComments(pId);
END$$
DELIMITER ;
```
Exercise 4 (run this by right clicking 'views' and then insert code):
```sql
CREATE 
    ALGORITHM = UNDEFINED 
    DEFINER = `root`@`%` 
    SQL SECURITY DEFINER
VIEW `QuestionAndAnswers` AS
    SELECT 
        (SELECT 
                `users_table`.`DisplayName`
            FROM
                `users` `users_table`
            WHERE
                (`users_table`.`Id` = `posts_table`.`OwnerUserId`)) AS `questioner`,
        JSON_OBJECT('displayName',
                (SELECT 
                        `users`.`DisplayName`
                    FROM
                        `users`
                    WHERE
                        (`users`.`Id` = `posts_table`.`OwnerUserId`)),
                'question',
                `posts_table`.`Title`,
                'answers',
                (SELECT 
                        JSON_ARRAYAGG(JSON_OBJECT('displayName',
                                            (SELECT 
                                                    `users`.`DisplayName`
                                                FROM
                                                    `users`
                                                WHERE
                                                    (`users`.`Id` = `comments_table`.`UserId`)),
                                            'answer',
                                            `comments_table`.`Text`,
                                            'score',
                                            `comments_table`.`Score`))
                    FROM
                        `comments` `comments_table`
                    WHERE
                        (`comments_table`.`PostId` = `posts_table`.`Id`)),
                'score',
                `posts_table`.`Score`) AS `json`
    FROM
        `posts` `posts_table`
    WHERE
        (`posts_table`.`Title` <> '')
```
Exercise 5:
```sql
DELIMITER $$
CREATE DEFINER=`root`@`%` PROCEDURE `searchPostsWithKeyWord`(postKeyWord varchar(255))
BEGIN
SELECT 
    json_extract(json, '$.question') AS question,
    json_extract(json, '$.answers') AS answers,
    json_length(json_extract(json, '$.answers')) as amount
FROM QuestionAndAnswers where json_extract(json, '$.question') like concat('%', postKeyWord, '%');
END$$
DELIMITER ;
```



<h2>Set up the database if needed<g-emoji class="g-emoji" alias="checkered_flag" fallback-src="https://github.githubassets.com/images/icons/emoji/unicode/1f3c1.png">🏁</g-emoji></h2>

1- Open MySQL workbench and use the following quries then execute
```sql
create database stackoverflow DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;

use stackoverflow;

create table badges (
  Id INT NOT NULL PRIMARY KEY,
  UserId INT,
  Name VARCHAR(50),
  Date DATETIME
);

CREATE TABLE comments (
    Id INT NOT NULL PRIMARY KEY,
    PostId INT NOT NULL,
    Score INT NOT NULL DEFAULT 0,
    Text TEXT,
    CreationDate DATETIME,
    UserId INT NOT NULL
);

CREATE TABLE post_history (
    Id INT NOT NULL PRIMARY KEY,
    PostHistoryTypeId SMALLINT NOT NULL,
    PostId INT NOT NULL,
    RevisionGUID VARCHAR(36),
    CreationDate DATETIME,
    UserId INT NOT NULL,
    Text TEXT
);
CREATE TABLE post_links (
  Id INT NOT NULL PRIMARY KEY,
  CreationDate DATETIME DEFAULT NULL,
  PostId INT NOT NULL,
  RelatedPostId INT NOT NULL,
  LinkTypeId INT DEFAULT NULL
);


CREATE TABLE posts (
    Id INT NOT NULL PRIMARY KEY,
    PostTypeId SMALLINT,
    AcceptedAnswerId INT,
    ParentId INT,
    Score INT NULL,
    ViewCount INT NULL,
    Body text NULL,
    OwnerUserId INT NOT NULL,
    LastEditorUserId INT,
    LastEditDate DATETIME,
    LastActivityDate DATETIME,
    Title varchar(256) NOT NULL,
    Tags VARCHAR(256),
    AnswerCount INT NOT NULL DEFAULT 0,
    CommentCount INT NOT NULL DEFAULT 0,
    FavoriteCount INT NOT NULL DEFAULT 0,
    CreationDate DATETIME
);

CREATE TABLE tags (
  Id INT NOT NULL PRIMARY KEY,
  TagName VARCHAR(50) CHARACTER SET latin1 DEFAULT NULL,
  Count INT DEFAULT NULL,
  ExcerptPostId INT DEFAULT NULL,
  WikiPostId INT DEFAULT NULL
);


CREATE TABLE users (
    Id INT NOT NULL PRIMARY KEY,
    Reputation INT NOT NULL,
    CreationDate DATETIME,
    DisplayName VARCHAR(50) NULL,
    LastAccessDate  DATETIME,
    Views INT DEFAULT 0,
    WebsiteUrl VARCHAR(256) NULL,
    Location VARCHAR(256) NULL,
    AboutMe TEXT NULL,
    Age INT,
    UpVotes INT,
    DownVotes INT,
    EmailHash VARCHAR(32)
);

CREATE TABLE votes (
    Id INT NOT NULL PRIMARY KEY,
    PostId INT NOT NULL,
    VoteTypeId SMALLINT,
    CreationDate DATETIME
);

create index badges_idx_1 on badges(UserId);

create index comments_idx_1 on comments(PostId);
create index comments_idx_2 on comments(UserId);

create index post_history_idx_1 on post_history(PostId);
create index post_history_idx_2 on post_history(UserId);

create index posts_idx_1 on posts(AcceptedAnswerId);
create index posts_idx_2 on posts(ParentId);
create index posts_idx_3 on posts(OwnerUserId);
create index posts_idx_4 on posts(LastEditorUserId);

create index votes_idx_1 on votes(PostId);
```

2- Go to your vagrant or linux and run the following command to create a mysql container with user: root and password: 123
<pre>
<code>
docker run --name some-mysql -v $(pwd)/mysql_databasefiles:/var/mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123 -d mysql
 </code>
</pre>

3-After creating the mysql container acces it by using this command
<pre>
<code>
docker ecex -it some-mysql -bash
</code>
</pre>

4- When you are inside "some-mysql" container download the wget and p7zip 
first make a apt update:

<pre>
<code>
apt-get install update
</code>
</pre>

then download them using those two commands
<pre>
<code>
apt-get install p7zip-full
</code>
</pre>

<pre>
<code>
apt-get install wget
</code>
</pre>

5-Now download the code.stachexchange 7z file 

<pre>
<code>
wget https://archive.org/download/stackexchange/coffee.stackexchange.com.7z
</code>
</pre>

6-After downloading the file now we need to extract.
  <pre>
<code>
7z e coffee.stackexchange.com.7z  
  </code>
</pre>
 7-Now acces the MySQL using the following command
```sql
mysql -u myuser -p --local-infile stackoverflow
```
8-After accesing the Mysql you need to set global local_infile in to true
<pre>
<code>
set global local_infile = 1
</code>
</pre>

9-Now we need to infile the xml files to our database tables /n
*For the Badges table
```sql
load xml local infile 'Badges.xml' into table badges rows identified by '<row>';
```
*For the Comments table
```sql
load xml local infile 'Comments.xml' into table comments rows identified by '<row>';
```
*For the PostHistory table
```sql
  load xml local infile 'PostHistory.xml' into table post_history rows identified by '<row>';
  ```
  *For the PostLinks table
  ```sql
  load xml local infile 'PostLinks.xml' into table post_links rows identified by '<row>';
  ```
  *For the Posts table
  ```sql
  load xml local infile 'Posts.xml' into table posts identified by '<row>';
  ```
  *For the Tags table
  ```sql
  load xml local infile 'Tags.xml' into table tags identified by '<row>';
  ```
  *For the Users table
  ```sql
  load xml local infile 'Users.xml' into table users identified by '<row>';
  ```   
     *For the Votes table
     
```sql
     load xml local infile 'Votes.xml' into table votes identified by '<row>';
```
