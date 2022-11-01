### ksqlDB intro: a terrible, horrible, no good, very bad tutorial


<br />

🚀🐋

---

Here's the repo y'all really want https://github.com/VaradGhodake/ksql-demo

```
git clone https://github.com/VaradGhodake/ksql-demo.git
cd ksql-demo
```

Launch 3 terminals, preferably with TMUX or Windows Terminal for convenience.

#### Terminal 1

Boot up everything

```
docker-compose up -d
docker-compose ps
```
Pull up a shell. The `--network` flag isn't something to be overlooked. This simulates intranet. Everything we do, should be confined within this same network.

```
docker run --network docker_default --rm --interactive --tty confluentinc/ksqldb-cli:0.6.0 ksql http://ksql-server:8088
```

#### Terminal 2:
Generate page views, push them to the `pageviews` topic <br />
DO NOT FORGET `sudo` unless you're madly in love with Linux error code _#126_ 

```sh
cd setup/docker
sudo ./generate_pageviews.sh
```

#### Terminal 1

```sql
show topics;
```

If it doesn't show you the `pageview` topic, you're doing something wrong. Go back and perform all the steps again, slowly this time.

#### Terminal 1
Create a stream for `pageview` topic called `pageviews_original`
```sql
CREATE STREAM pageviews_original (rowkey string key, viewtime bigint, userid varchar, pageid varchar) WITH (kafka_topic='pageviews',
value_format='DELIMITED');
```

Details can be seen with `DESCRIBE pageviews_original;` <br />

#### Terminal 1

Create another topic for users, `users_original` <br />
The `key` portion took a while for me to figure out, make sure we're on the same page here.

```sql
CREATE TABLE users_original (rowkey string key, viewtime bigint, userid varchar, regionid varchar, gender varchar) WITH (kafka_topic='users', key='userid',value_format='DELIMITED',partitions=1);
```

#### Terminal 3

Now bombard the newly created topic with users moving all around the regions.

```sh
sudo ./generate_users.sh
```

#### Terminal 1
In case you'd want to run the query for everything from the start,
```sh
set 'auto.offset.reset'='earliest';
# 'latest' if there's too much to process
```

#### SQL Examples

`LEFT JOIN` example:
```sql
SELECT users_original.userid AS userid, pageid, regionid, gender FROM pageviews_original LEFT JOIN users_original ON pageviews_original.userid = users_original.userid EMIT CHANGES;
```

`Filter` example:
```sql
SELECT * FROM users_original WHERE gender = 'FEMALE' EMIT CHANGES;
```

`Aggregation` example:
```sql
SELECT userid, COUNT(pageid) AS p_count FROM pageviews_original
GROUP BY userid EMIT CHANGES;
```
