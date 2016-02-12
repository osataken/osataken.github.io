---
layout: post
title:  "Mgo it Mongo Way!!!"
date:   2016-02-12 22:22:22
categories: Go, MongoDB
image: go_meets_mongo.PNG
---

วันนี้มานำเสนอการเขียน Go ต่อ Mongo ด้วย Mgo แบบ MongooooO Mongo
<!--more-->

### เชื่อมต่อ Go กับ Mongo
หลายคนคงเคยลองเชื่อมต่อ Go กับ Mongo ดูกันแล้ว ด้วย library เจ้าดัง [mgo] จากคุณ Labix ซึ่งผลงานของแกก็เป็นที่น่าประทับใจ ใช้งานได้ค่อนข้างง่ายสะดวกรวดเร็ว 
ถ้าสังเกตการใช้งานหลักๆ ก็จะคล้ายคลึงกับการเชื่อมต่อ database ทั่วไปซึ่งก็คือ

1. ต่อไปยัง database host และ เปิด session (dial)
2. Execute query หรือ update
3. ปิด session

ดังตัวอย่าง code ด้านล่างเป็นการ insert document เข้า Mongo collection แบบง่ายๆ 

{% highlight go %}
package main
import (
        "fmt"
	    "log"
        "gopkg.in/mgo.v2"
        "gopkg.in/mgo.v2/bson"
)
type Villager struct {
        Name string
        Surname string
}
func main() {
        session, err := mgo.Dial("http://gophervillage.com")
        if err != nil {
            panic(err)
        }
        defer session.Close()
        c := session.DB("gopher").C("village")
        err = c.Insert(&Villager{"Ken", "Sama"},
	               &Villager{"Gopher", "Mongo"})	               
        if err != nil {
            log.Fatal(err)
        }
}
{% endhighlight %}

จะเห็นว่าในการใช้งานจริง หากเราต้องมาทำขั้นตอนดังกล่าวทุกครั้งนี่ก็แลดูจะไม่เป็นมิตรกับ programmer รุ่นเราและรุ่นต่อๆ ไปเท่าไหร่
คิดว่าหลายๆ คนคงมีวิธีการจัดการที่ดีกว่านี่แน่นอน (เนอะ)  


### ทีนี้ถ้าจะเอามาใช้งานจริงล่ะ? จะเขียนยังไงดี?
วิธีการที่ผมจะนำเสนอวันนี้คือวิธีการที่จะทำให้เราเขียน Go ต่อ Mongo โดยให้อารมณ์คล้ายๆ กับเรากำลังใช้ Mongo CLI อยู่
โดยปรกติ เวลาเราใช้ Mongo CLI เราก็จะใช้คำสั่ง db.{collection}.find() เช่นในกรณีของ collection ชื่อ villager ก็จะเป็น
{% highlight go %}
db.villager.find({name:"Ken"})
{% endhighlight %}

สิ่งที่ผมคาดหวังจากความพยายามครั้งนี้คือ เวลาเขียน code Go ก็จะเป็นรูปแบบที่ใกล้เคียงกับ code ด้านล่างนี้ให้มากที่สุด
{% highlight go %}
db.Villager.find(bson.M{"name": "Ken"})
{% endhighlight %}

เริ่มต้นจากการสร้าง package db และ file mongo.go และ datastore.go ภายใต้ package ที่สร้าง
โดย file mongo.go ผมจะเอาไว้สำหรับจัดการ connection และ session ของ mgo ส่วน datastore.go จะเป็นตัวที่เราเอาไว้ทำเป็นตัวเชื่อมไปสู่การ query ในรูปแบบต่างๆครับ เรามาดู source code กันครับ

mongo.go 
{% highlight go %}
package db

import (
	"gopkg.in/mgo.v2"
	"gopkg.in/mgo.v2/bson"
)

var (
	mongoSession *mgo.Session
)

func InitMongoDB() {
	conStr := fmt.Sprintf("mongodb://%v:%v@%v/%v", "username", "password", "host", "databasename")
	session, err := mgo.Dial(conStr)

	if err != nil {
		log.Fatal("Cannot connect to mongodb")
		return
	}
	session.SetMode(mgo.Monotonic, true)

	mongoSession = session
}

func Session() *mgo.Session {
	return mongoSession.Copy()
}

func WithCollection(collection string, s func(*mgo.Collection) error) error {
	session := Session()
	defer session.Close()
	c := session.DB(DatabaseName).C(collection)
	return s(c)
}
{% endhighlight %}

datastore.go
{% highlight go %}
package db

import (
	"gopkg.in/mgo.v2"
	"gopkg.in/mgo.v2/bson"
)

type DataStore struct {
	collection string
}

func NewDataStore(collection string) *DataStore {
	return &DataStore{collection}
}

func (ds *DataStore) Insert(model interface{}) (err error) {
	query := func(c *mgo.Collection) error {
		fn := c.Insert(model)
		return fn
	}

	create := func() error {
		return WithCollection(ds.collection, query)
	}
	err = create()
	return
}

func (ds *DataStore) Find(q interface{}, skip int, limit int, result interface{}) (err error) {
	query := func(c *mgo.Collection) error {
		fn := c.Find(q).Skip(skip).Limit(limit).All(result)
		if limit < 0 {
			fn = c.Find(q).Skip(skip).All(result)
		}
		return fn
	}
	find := func() error {
		return WithCollection(ds.collection, query)
	}
	err = find()

	return
}

var Villager = NewDataStore("villager")
{% endhighlight %}

วิธีการใช้งาน
{% highlight go %}
package main
import (
	"github.com/osataken/go-mgo/db"
	"github.com/osataken/go-mgo/models"
	"gopkg.in/mgo.v2/bson"
	"fmt"
)

func main() {
	db.InitMongoDB()

	db.Villager.Insert(&models.Villager{Name:"Ken", Surname:"Sama"})

	var villagers []models.Villager
	db.Villager.Find(bson.M{"name": "Ken"}, 0, -1, &villagers)

	fmt.Print(villagers)
}
{% endhighlight %}

#### อธิบายเพิ่มเติมนิดนึง
1. db.InitMongoDB() เรียกครั้งเดียวตอน app start หรือ web server start หลังจากนั้นเรียกผ่าน DataStore ตลอด ในที่นี้ผมเรียกที่บรรทัดแรกของ main.go
2. file mongo.go จะเห็นว่าจะมีการ dial ไปที่ Mongo Host เพียงครั้งเดียวหลังจากนั้นจะเรียก Session() ซึ่งเป็นการ copy session เพื่อนำไปใช้งานในงานอื่นๆ ต่อไป
3. file mongo.go จะมี function WithCollection() ซึ่งทำหน้าที่ copy session และเรียกใช้งาน database และ collection ที่ต้องการ
4. file datastore.go จะเห็นว่ามี function Insert() กับ Find() ซึ่งเป็นการแอบทำ closure อยู่ใน function เพื่อประสานกับ WithCollection() ทำให้สามารถสร้าง function เดียวกันกับ object ที่หลากหลายได้ครับ
5. หากอยากได้ collection ใหม่ก็เพิ่มเข้าไปใน datastore.go ครับ หรือจะแยกมาเป็นอีก file ก็ได้แล้วสะดวกจะจัดการเลยครับ


หากใครสนใจอยากดู source code เพิ่มเติมสามารถเข้าไปดูได้ที่ [go-mgo]
(code ยังไม่สวยเอาไปปรับก่อนใช้งานกันด้วยนะครับ ^^)

[mgo]: https://labix.org/mgo
[go-mgo]: https://github.com/osataken/go-mgo