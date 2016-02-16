---
layout: post
title:  "เพิ่มความปลอดภัยให้กับข้อมูลด้วย mgo"
date:   2016-02-16 22:33:33
categories: Go
image: mgo_encrypt.PNG
---

วันนี้มานำเสนอวิธีการสำหรับคนที่อาจจะได้ requirements แปลกๆ จากงานที่ต้องทำเช่นการ encrypt ข้อมูลก่อนการเขียนลง database อันเนื่องมาจากเหตุผลเรื่อง security ^^
<!--more-->

จากคราวก่อนผมนำเสนอวิธีการใช้ mgo แบบที่คล้ายๆ กับการใช้ Mongo CLI ไปคราวนี้ขอต่อยอดงานที่แล้วโดยเพิ่มการ encrypt/decrypt data ก่อนการเขียนและอ่านบน MongoDB ครับ
โดย Algorithm ที่จะเอามาใช้ encrypt/decrypt นั้นจะใช้ [AES Encryption] แบบ CFB

วิธีการที่จะนำมาใช้นั้นก็คือการ implement interface [Getter] และ [Setter] จาก [mgo/bson]

Getter
{% highlight go %}
type Getter interface {
    GetBSON() (interface{}, error)
}
{% endhighlight %}

Setter
{% highlight go %}
type Setter interface {
    SetBSON(raw Raw) error
}
{% endhighlight %}

### AES Encryption in Go
ก่อนอื่นต้องเริ่มจากการเขียน util ในการ encrypt และ decrypt ด้วย AES Encryption แบบ CFB ดังตัวอย่าง code ด้านล่าง
{% highlight go %}
package util
import (
	"crypto/aes"
	"encoding/base64"
	"io"
	"crypto/rand"
	"crypto/cipher"
	"fmt"
)

func Encrypt(key, text string) (string, error) {
	block, err := aes.NewCipher([]byte(key))
	if err != nil {
		return "", err
	}

	b := []byte(text)
	ciphertext := make([]byte, aes.BlockSize+len(b))
	iv := ciphertext[:aes.BlockSize]
	if _, err := io.ReadFull(rand.Reader, iv); err != nil {
		return "", err
	}

	cfb := cipher.NewCFBEncrypter(block, iv)
	cfb.XORKeyStream(ciphertext[aes.BlockSize:], []byte(b))
	return base64.StdEncoding.EncodeToString(ciphertext), nil
}

func Decrypt(key, cryptoText string) (string, error) {
	block, err := aes.NewCipher([]byte(key))
	if err != nil {
		return "", err
	}

	ciphertext, err := base64.StdEncoding.DecodeString(cryptoText)
	if err != nil {
		return "", err
	}

	if len(ciphertext) < aes.BlockSize {
		panic("ciphertext too short")
	}
	iv := ciphertext[:aes.BlockSize]
	ciphertext = ciphertext[aes.BlockSize:]

	stream := cipher.NewCFBDecrypter(block, iv)

	// XORKeyStream can work in-place if the two arguments are the same.
	stream.XORKeyStream(ciphertext, ciphertext)

	return fmt.Sprintf("%s", ciphertext), nil
}
{% endhighlight %}
อธิบาย code ข้างบนซักนิดหน่อยก่อนนะครับ หลักๆ ใน function Encrypt นั้นจะรับ Key และ Text ที่ต้องการ Encrpyt เข้ามา
เมื่อทำการ Encrypt เสร็จแล้วจะ encode แบบ Base64 กลายเป็น string เพื่อทำการ write ลง MongoDB เป็น field หนึ่งซึ่งมี data type เป็น string ครับ
หากใครไม่ต้องการใช้ Base64 encode ก็สามารถเขียนลง MongoDB เป็น Binary (byte array) ได้เลยเช่นกันครับ MongoDB รองรับ field ที่เป็น Binary เช่นกัน แต่ข้อเสียคืออาจจะเจอปัญหาหากบางภาษาโปรแกรมที่ใช้อาจจะไม่ค่อยถูกกับ Binary format เท่าไหร่ครับ

ในส่วนของการ decrypt นั้นจะรับ string เข้ามาแล้วทำการ Base64 decode จากนั้นค่อย decrypt ด้วย key ที่ผ่านมาครับ เช่นกันหากใครอยากใช้เป็น binary เลยก็ได้ครับ

### mgo Encrypt on Write, Decrypt on Read
ทีนี้เรามาดูกันว่าจะ encrypt ข้อมูลเมื่อเวลาที่เราต้องการเขียนลง MongoDB และ decrypt ข้อมูลตอนอ่านได้อย่างไร
ตามที่กล่าวไว้ข้างต้น วิธีการก็คือการ implement Getter/Setter interface ของ mgo/bson โดยให้เรียก function Encrypt ใน Getter และ Decrypt ใน Setter ดังตัวอย่าง code ด้านล่าง

เริ่มต้นจากการสร้าง data type ใหม่โดยมี base มาจาก string
{% highlight go %}
type EncryptedString string
{% endhighlight %}

จากนั้น implement Getter/Setter ของ mgo/bson
{% highlight go %}
type EncryptedString string

func (e EncryptedString) GetBSON() (interface{}, error) {
	...
}

func (e *EncryptedString) SetBSON(raw bson.Raw) error {
	...
}
{% endhighlight %}

เรียก function Encrypt ใน Getter และ Decrypt ใน Setter
โดย GetBSON จะถูกเรียกโดย mgo ณ เวลาที่กำลังจะ write ข้อมูลลง MongoDB 
ส่วน SetBSON นั่นจะถูกเรียกโดย mgo ณ เวลาที่ mgo read ข้อมูลจาก MongoDB เพื่อจะ populate ค่าใส่ object เป้าหมายครับ
{% highlight go %}
func (e EncryptedString) GetBSON() (interface{}, error) {
	return util.Encrypt(key, string(e))
}

func (e *EncryptedString) SetBSON(raw bson.Raw) error {
	var str string
	raw.Unmarshal(&str)
	decrypted, err := util.Decrypt(key, str)
	if err != nil {
		return err
	}

	*e = EncryptedString(decrypted)
	return nil
}
{% endhighlight %}

ในขั้นตอนสุดท้าย เราก็สามารถระบุ field ที่เราต้องการ encrypt/decrypt ใน struct ที่เราต้องการได้ โดย code รวมจะออกมาประมาณด้านล่างนี้ครับ
ในส่วนของ Key ต้องเป็น 32 bytes นะครับ หากต้องการเปลียนขนาดต้องไปแก้ code ในส่วนของการทำ AES Encryption ที่ระบุไว้ด้านบน
{% highlight go %}
package models
import (
	"gopkg.in/mgo.v2/bson"
	"github.com/osataken/go-mgo/util"
)

var (
	key = "32 bytes secret key for aes CFB!"
)

type SecretInfo struct {
	Regular string              `bson:"regular"`
	Encrypted EncryptedString   `bson:"encrypted"`
}

type EncryptedString string

func (e EncryptedString) GetBSON() (interface{}, error) {
	return util.Encrypt(key, string(e))
}

func (e *EncryptedString) SetBSON(raw bson.Raw) error {
	var str string
	raw.Unmarshal(&str)
	decrypted, err := util.Decrypt(key, str)
	if err != nil {
		return err
	}

	*e = EncryptedString(decrypted)
	return nil
}
{% endhighlight %}

เมื่อทำการ insert ข้อมูล SecretInfo ลง MongoDB จะได้ข้อมูลที่ถูก encrypt ประมาณนี้ครับ
{% highlight go %}
{ 
    "_id" : ObjectId("56c330cdc88ff0d47dab4722"), 
    "regular" : "regular data", 
    "encrypted" : "MigscfJOeczIW7L5cghS9WP/F0Mv15lhOR3DdyX0"
}
{% endhighlight %}

เป็นอย่างไรบ้างครับ mgo นั้นนอกจากจะใช้งานง่ายแล้วก็ยังสามารถต่อยอดหรือ customize ได้ค่อนข้างสะดวก หลายคนอาจจะนำวิธีการข้างต้นไปประยุกต์ใช้ในรูปแบบอื่นๆ ได้เช่นใช้การเข้ารหัสในรูปแบบอื่นๆ 
หากใครสนใจอยากดู code เพิ่มเติมสามารถเข้าไปดูได้ที่ [https://github.com/osataken/go-mgo] ครับ

[AES Encryption]: https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation
[Getter]: https://godoc.org/gopkg.in/mgo.v2/bson#Getter
[Setter]: https://godoc.org/gopkg.in/mgo.v2/bson#Setter
[mgo/bson]: https://godoc.org/gopkg.in/mgo.v2/bson
[https://github.com/osataken/go-mgo]: https://github.com/osataken/go-mgo