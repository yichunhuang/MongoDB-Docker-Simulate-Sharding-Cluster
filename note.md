--- 建 config-server replica
docker-compose -f config-server/docker-compose.yaml up -d

docker volume ls
docker-compose -f config-server/docker-compose.yaml ps

mongo mongodb://192.168.0.12:40001
rs.initiate(
  {
    _id: "cfgrs",
    configsvr: true,
    members: [
      { _id : 0, host : "192.168.0.12:40001" },
      { _id : 1, host : "192.168.0.12:40002" },
      { _id : 2, host : "192.168.0.12:40003" }
    ]
  }
)

--- 建立第一個 shard replica

docker-compose -f shard1/docker-compose.yaml up -d

mongo mongodb://192.168.0.12:50001
rs.initiate(
  {
    _id: "shard1rs",
    members: [
      { _id : 0, host : "192.168.0.12:50001" },
      { _id : 1, host : "192.168.0.12:50002" },
      { _id : 2, host : "192.168.0.12:50003" }
    ]
  }
)

--- 建立 MONGOS 並設定 config server(在 compose 中的 command) & 加入 shard，MONGOS 可以不止一台

docker-compose -f mongos/docker-compose.yaml up -d

mongo mongodb://192.168.0.12:60000

sh.addShard("shard1rs/192.168.0.12:50001,192.168.0.12:50002,192.168.0.12:50003")
sh.status()

--- 建立第二個 shard replica

docker-compose -f shard2/docker-compose.yaml up -d

mongo mongodb://192.168.0.12:50004
rs.initiate(
  {
    _id: "shard2rs",
    members: [
      { _id : 0, host : "192.168.0.12:50004" },
      { _id : 1, host : "192.168.0.12:50005" },
      { _id : 2, host : "192.168.0.12:50006" }
    ]
  }
)

--- MONGOS 加入第二個 shard
mongo mongodb://192.168.0.12:60000

sh.addShard("shard2rs/192.168.0.12:50004,192.168.0.12:50005,192.168.0.12:50006")
sh.status()

--- shard a empty mongodb collection
mongo mongodb://192.168.0.12:60000
show dbs
use sharddemo
sh.enableSharding("sharddemo") // database must enable sharding first for collection to be sharded later
sh.shardCollection("sharddemo.movies2", {"title": "hashed"}) // try again
db.movies2.getShardDistribution()

for i in {1..50}; do echo -e "use sharddemo \n db.movies2.insertOne({\"title\": \"Spider Man $i\", \"language\": \"English\"})" | mongo mongodb://192.168.0.12:60000; done

db.movies2.find()
db.movies2.getShardDistribution()

--- shard a non empty mongodb collection
for i in {1..5}; do echo -e "use sharddemo \n db.movies1.insertOne({\"title\": \"Spider Man $i\", \"language\": \"English\"})" | mongo mongodb://192.168.0.12:60000; done

db.movies1.createIndex({"title": "hashed"})
sh.shardCollection("sharddemo.movies1", {"title": "hashed"}) // dont know why
