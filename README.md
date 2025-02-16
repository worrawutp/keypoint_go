# Keypoint Go

### Pointer and composite literals
Pointer จะถูกใช้กับ struct ค่อนข้างบ่อย ดังนั้น Go จึงมีข้อยกเว้นอย่างนึงสำหรับ struct นั้นคือเราไม่จำเป็นต้องเขียน
เครื่องหมาย dereference เมื่อต้องการจะ point ไปที่ตัวแปรที่เป็น struct
Go จะทำการ dereference ให้เองโดยอัตโนมัติ

ตัวอย่าง

```go
type book struct {
  name string
  pages int
}

learnGo := &book {
  name: "Learn Go",
  pages: 100
}

// ตอนเรียกใช้ไม่จำเป็นต้องเขียน dereference
// ไม่จำเป็นต้องเขียนแบบนี้
// fmt.Println((*leanGo).name)
fmt.Println(leanGo.name)
```

แต่กับข้อมูลชนิดอื่นอย่าง slice หรือ map Go จะไม่ทำ dereference ให้ เราต้องเขียน dereference เอง

```go
favouriteBooks := &[]string{"Wellground Ruby", "Get Programming with Go", "Programming Ruby"}
fmt.Println(*(favouriteBooks)[1])  // => Get Programming with Go
// fmt.Println(favouriteBooks[1])  // => invalid operation: cannot index favouriteBooks (variable of type *[]string)

books := &[]book{
    {name: "hello", pages: 100},
    {name: "trinity piano", pages: 25},
}
fmt.Printf("%v\n", (*books)[0].name)  // => hello
fmt.Printf("%v\n", books[0].name)  // => invalid operation: cannot index books (variable of type *[]book)
```


