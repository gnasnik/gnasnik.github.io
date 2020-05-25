---
layout:     post
title:      "mongodb 的使用"
subtitle:   ""
date:       2019-02-14 21:20:05
author:     "frank"
# header-style: text
header-img: "img/post-bg-halting.jpg"
tags:
    - golang
	- mongodb
---


## 记录一下 mongodb 在项目中的使用

golang 中推荐使用官方的 mongodb 驱动	
```
"go.mongodb.org/mongo-driver/mongo"
```

当然还有官方配套的其他工具	
```
"go.mongodb.org/mongo-driver/bson"

"go.mongodb.org/mongo-driver/mongo/options"
```

## 使用方法
在项目中，首先当然要想到连接的复用，不能每次操作就申请一个数据连接，所以通常会把 mongodb 的 client 暴露出来，看下项目结构	
```
├── config
│   └── config.go
├── main.go
└── sdk
    └── miner
        └── miner.go
```

把 `var Mongo *mongo.Client` 定义为全局的变量

config.go 

```
package config

import (
	"context"

	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/option"
)

const mongoURL = "mongodb://root:example@127.0.0.1:7017/center?authsource=admin"

var Mongo *mongo.Client

func Init() error {

	Mongo, err := mongo.NewClient(option.Client().ApplyURI(mongoURL))
	if err != nil {
		return err
	}

	err = Mongo.Connect(context.Background())
	if err != nil {
		return err
	}

	err = Mongo.Ping(context.Background(), nil)
	if err != nil {
		return err
	}

	return nil
}

func Database() *mongo.Database {
	return Mongo.Database("filecoin")
}

func Miner() *mongo.Collection {
	return Database().Collection("miner")
}

```

然后会把数据操作提出来，单独作 DAO 层, 下面是 `sdk/miner.go`	
```
package miner

import (
	"context"

	"github.com/pkg/errors"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/bson"
)

func GetMiner(ctx context.Context, c *mongo.Collection) (interface{}, error) {
	f := bson.D{}
	rs, err := c.Find(ctx, f)
	if err != nil {
		return nil, errors.WithMessage(err, "get all miner failed")
	}

	defer rs.Close(ctx)

	for rs.Next(ctx) {
		// dosomthing
	}

	return nil, nil
}

```

最后上层通过 DAO 来调用就行了 

最后是 `main.go` 伪代码 
```
package main

import (
	"context"
	"log"
	"vtest/mongodb/config"
	"vtest/mongodb/sdk/miner"
)

func main() {
	err := config.Init()
	if err != nil {
		log.Fatal(err)
	}

	_, err = miner.GetMiner(context.Background(), config.Miner())
	if err != nil {
		// handler error
	}

}
```