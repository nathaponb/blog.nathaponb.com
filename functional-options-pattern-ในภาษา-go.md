---
title: "Functional Options Pattern ในภาษา Go"
thumbnail: functional-options-pattern-ในภาษา-go.jpg
---

# Functional Options Pattern ในภาษา Go

**_10-12-2023_**

![thumbnail](/images/only-en.jpg)

> _"เช้าวันอาทิตย์ ที่ฝุ่น pm2.5 ลงปกคลุมกรุงเทพ จนแทบจะไม่สามารถทอดสายตามองทัศนะวิศัยได้ไกลเกินกว่า 1 กม. ตึกรามบ้านช่องที่เห็นเป็นเงาลางๆ ในสายฝุ่นสีเทาทะมึน มลภาวะทางอากาศที่เกิดขึ้น เป็นผลมาจากการทำลายและจัดการสิ่งแวดล้อมที่ไม่มีประสิทธิภาพ."_

เช่นเดียวกันกับ Codebase หากไม่มีการจัดการที่ดีก็อาจนำไปสู่สิ่งที่เรียกว่า "Code Pollution", ไม่เพียงแค่จะยากต่อการเพิ่มฟีเจอร์ใหม่ๆเข้าไป แต่การลบหรือ refactor กลับทำได้ยากยิ่งกว่า.

วันนี้ เราลองมาทำความรู้จักกับ Pattern หนึ่ง ที่จะช่วยลดมลภาวะ (Pollution) ให้กับ Codebase ของเรา และที่สำคัญยังช่วยให้เข้าใจง่าย ทั้งต่อผู้พัฒนาและผู้ใช้งาน

Pattern นี้มีชื่อว่า “**Functional Options**”.

ก่อนอื่นเลย เราลองมาทำความเข้าใจ Pattern นี้ จากชื่อกันก่อนสักหน่อย

- Function คือ ชุดคำสั่งหรือข้อมูลสำหรับปฏิบัติการใดๆ.
- Option คือ รูปแบบที่สามารถเลือกได้ อาทิเช่น `true` หรือ `false`, 0 หรือ 1, สิ่งนั้น หรือ สิ่งนี้ เป็นต้น.
- Pattern คือ รูปแบบหรือวิธีการใดๆ.

ดังนั้น หากรวมความ _( ในบริบทของ Programming )_ แล้ว ก็จะได้ความหมายว่า “รูปแบบการเขียนโค้ด ที่ใช้ "ฟังค์ชั่น" เป็น Optional Parameter”.

![Branching](/images/interesting.png)

เกริ่นกันไปพอแล้ว งั้นมาเขียน Http Server โดยใช้ Functional Options Pattern กัน

เริ่มด้วยการสร้าง struct ให้ชื่อว่า `Configs` ไว้เก็บค่า configuration ต่างๆที่ต้องใช้ สำหรับการสร้าง server.

```go
  // main.go

  package main

  type Configs struct {
    Addr string
    Port string
  }

  func main() {}

```

ต่อมา จะเป็นประกาศ “ฟังค์ชั่น type” ชื่อว่า `Option`
โดย signature ของ "ฟังค์ชั่น type" นี้ จะรับ parameter เป็น Pointer ไปหา Object ที่สร้างจาก `Configs` ข้างต้น.

```go
  // main.go

  package main

  type Configs struct {
    Addr string
    Port string
  }

  type Option func(*Configs)

  func main() {}

```

เมื่อเรากำหนดให้ `Configs` เป็น Object สำหรับเก็บค่า Configuration ต่างๆของ server เอาไว้

และกำหนดให้ `Option` รับ Parameter เป็น memory address ของ `Configs`

นั้นหมายความว่า เราอยากจะให้มีการเปลี่ยนค่า Configs อะไรก็แค่ไปเขียน Implementation ของ Option ได้เลย

โดยฟังค์ชั่นแรก task คือ เพื่อเปลี่ยนค่า Addr ใน Configs

แต่การเปลี่ยนก็ต้องรับ ค่า string มาจาก User ก่อน ดังนั้นเราจะใช้ Higher Order function เพื่อรับ string ที่ User ต้องการจะให้เป็นค่า Address ของ Server เข้ามา และจะ return `Option` ที่มีการ Implementation แล้วนั้นออกไป

```go

  // main.go

  func WithAddress(address string) Option {
    return func(s *Configs) {
      s.Addr = address
    }
  }

```

ฟังค์ชั่นที่สอง ก็จะคล้ายๆกัน โดยจะรับ `int` จาก User เพื่อเปลี่ยนค่า Port ของ Server

```go
  // main.go

  func WithPort(port string) Option {
    return func(s *Configs) {
      s.Port = port
    }
  }

```

และก็มาถึงในส่วนสำคัญ

คือการมีฟังค์ชั่นหลัก สำหรับรับ parameter type เป็น `Option` และเมื่อ `Option` ก็คือ "ฟังค์ชั่น type" ดังนั้นเราก็สามารถเรียกใช้ (invoke) ได้เลย

โดยการส่ง memory address ของ Object ที่ Instantiate จาก `Configs` เข้าไปตาม signature ของ `Option`

เราจะใช้ [Variadic Function](https://yourbasic.org/golang/variadic-function/) ในการเขียนฟังค์ชั่นนี้ เพื่อรับ parameter แบบ Optional กล่าวคือ _"จะไม่ส่งหรือส่งเท่าไหร่ก็ได้"_ และ parameter ที่ส่งเข้ามาจะอยู่ในรูปแบบของ slice

```go

  // main.go


  func CreateAndRunHttpServer(options ...Option) error {

    configs := &Configs{
      Addr: "localhost",
      Port: "5000",
    }

    for _, option := range options {
      option(configs)
    }


    server := &http.Server{
      Addr: fmt.Sprintf("%s:%s", configs.Addr, configs.Port),
      Handler: configs.Routes(),
    }

    return server.ListenAndServe()
  }

  func main() {}

```

### 🎉

และแล้ว Http Server เวอร์ชั่น "Functional Options Pattern" ก็เสร็จเรียบร้อย

ฟังค์ชั่น
`CreateAndRunHttpServer` , `WithAddress` , `WithPort` จะถูกปล่อยเป็น API ออกไป เพื่อให้ผู้ใช้งานสามารถเรียกใช้ สร้าง HTTP Server และมีอิสระในการ Config ค่าต่างๆของ Server

โดยหากต้องการเปลี่ยน Address หรือ Port ก็แค่ส่ง parameter ที่มี type เป็น `Option` โดยได้มาจากการเรียกใช้ฟังค์ชั่น `WithAddress` และ `WithPort` นั้นเอง.

```go

  // main.go

  func main() {

    err := CreateAndRunHttpServer(
      WithAddress("localhost") ,
      WithPort("8080"),
    )
    if err != nil {
      log.Panic(err)
    }

  }

```

---

❤️ source code. 👉 [gist](https://gist.github.com/nathaponb/33b5a3bb219f7360b3b9b05532dfb9e0)
