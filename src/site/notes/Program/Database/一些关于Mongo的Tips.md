---
{"dg-publish":true,"permalink":"/Program/Database/一些关于Mongo的Tips/","noteIcon":""}
---

## 数据库连接

```bash
# connection
mongo "mongodb://root:123456@127.0.0.1:27017/"

```
### 删除重复数据

```javascript

// delete dupilicate keys
db.qy_user
  .aggregate([
    {
      $group: {
        _id: { user_id: '$user_id' },
        dups: { $push: '$_id' },
        count: { $sum: 1 }
      }
    },
    { $match: { count: { $gt: 1 } } }
  ])
  .forEach(function(doc) {
    doc.dups.shift()
    db.qy_user.remove({ _id: { $in: doc.dups } })
  })

// create unique index
db.wm_user.createIndex(
  {
    pid: 1
  },
  {
    unique: true
  }
)
// create user
db.createUser({
  user: 'booom',
  pwd: 'booom123',
  roles: [
    {
      role: 'readWrite',
      db: 'booom'
    }
  ]
})
// create index filter some columns
db.standard_hotel_aggregate.createIndex(
  {
    "info.bindedMap.meituan": 1,
  },
  {
    unique: true,
    partialFilterExpression: {
      'info.bindedMap.meituan': {
        $type: 'string',
      },
    },
  }
)

db.test.createIndex(
  {
    test: 1,
  },
  {
    unique: true,
    partialFilterExpression: {
      test: {
        $type: 'string',
      },
    },
  }
)

```
### 根据类型建立索引
```javascript
db.aggregation_hotels.createIndex(
  {
    'bindMap.bestwehotel': 1,
  },
  {
    unique: true,
    partialFilterExpression: {
      'bindMap.bestwehotel': {
        $type: 'string',
      },
    },
  }
)

```