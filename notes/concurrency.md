# Concurrency

### การใช้งาน concurrency ทั่วๆไป
1. สร้าง channel สำหรับส่งข้อมูลที่เราต้องการ เช่น sumChannel
2. สร้าง goroutine function สำหรับงานที่เราต้องการให้ทำไปพร้อมๆ กัน
3. ข้างใน goroutine function ให้ส่งค่าที่คำนวณได้ไปใน channel จากข้อ 1)
4. นำค่าจาก channel ออกมาเก็บในตัวแปร และนำไปใช้ตามต้องการ

```go
package main
import "fmt"
  
func sum(data []int, ch chan int) {
  result := 0
  for _, v := range data {
    result += v
  }
  ch <- result
}

func main() {
  sales := []int{200, 200, 280, 320, 200, 460, 400, 280, 320, 160, 200, 600}
  sumChannel := make(chan int)
  
  go sum(sales[:len(sales)/2], sumChannel)
  go sum(sales[len(sales)/2:], sumChannel)

  x, y := <-sumChannel, <-sumChannel
  fmt.Println(x, y, x+y)
  // 1960 1660 3620
}
```

### ใช้ WaitGroup และ for/select เมื่อมี goroutine หลายชุด (Classic)
1. สร้าง channel สำหรับส่งข้อมูลที่เราต้องการ
2. สร้าง doneChanel เอาไว้ส่งสัญญาณเมื่อ channel อื่นๆ ทำงานเสร็จหมดแล้ว โดยทั่วไป doneChannel เป็นแค่ struct เปล่าๆ ก็พอเช่น `doneChan := make(chan struct{})`
3. สร้าง WaitGroup เพื่อให้เราสามารถรอจนกระทั่ง goroutines ทั้งหมดทำงานจนเสร็จ ก่อนที่จะดำเนินการอย่างอื่นต่อไป
4. สร้าง goroutine function สำหรับงานที่เราต้องการให้ทำไปพร้อมๆ กัน โดยให้เพิ่มค่า counter ของ WaitGroup ก่อนที่จะสร้าง goroutine แบบนี้ `wg.Add(1)` และภายใน goroutine function ให้เพิ่ม `defer wg.Done()` เข้าไปด้วยเพื่อบอก WaitGroup ให้ลด counter ลงเมื่อ goroutine นี้ทำงานเสร็จ
5. ข้างใน goroutine function ให้ส่งค่าที่คำนวณได้ไปใน channel จากข้อ 1)
6. สร้าง goroutine สำหรับ WaitGroup goroutine ตัวนี้จะเป็นกลไกที่คอยตรวจสอบ WaitGroup counter ว่ามีงานเหลืออยู่หรือไม่ ถ้าไม่มีงานเหลือแล้ว ก็จะ close `doneChannel` ซึ่งจะเป็นการส่ง zero value ไปที่ receiver ของ doneChannel ต่อไป
7. สร้าง for/select เพื่อจัดการกับข้อมูลที่อยู่ใน channel เช่น 
  - นำค่าที่ได้จาก channel ที่เราต้องการไปใช้
  - handle error จาก errorChannel
  - กำหนดว่าจะทำอะไร ในกรณีที่ doneChannel ถูก trigger (ในกรณีนี้คือการ close doneChannel)

ตัวอย่าง
```go
func run(filenames []string, operation string, column int, out io.Writer) error {
    // ... other code before
  
    consolidate := make([]float64, 0)

    // define channels
    resultChan := make(chan []float64)
    errorChan := make(chan error)
    doneChan := make(chan struct{})
    wg := sync.WaitGroup{}

    for _, fn := range filenames {
        wg.Add(1)
        go func(fn string) {
            defer wg.Done()

            file, err := os.OpenFile(fn, os.O_RDONLY, 0644)
            if err != nil {
                errorChan <- os.ErrNotExist
                return
            }

            data, err := csv2float(file, column)
            if err != nil {
                errorChan <- err
            }

            file.Close()
            resultChan <- data
        }(fn)
    }

    go func(){
        wg.Wait()
        close(doneChan)
    }()

    for {
        select {
        case err := <-errorChan:
            return err
        case data := <-resultChan:
            consolidate = append(consolidate, data...)
        case <-doneChan:
            opResult := opFunc(consolidate)
            _, err := fmt.Fprintln(out, opResult)
            return err
        }
    }
}
```

### ใข้ errgroup package ในการรันและควบคุม goroutine และจัดการ error (Modern)
เราสามารถใช้ errgroup package แทนที่การใช้ waitgroup และ multiple channel ในการทำ concurrency ได้ดังนี้

```go
func run(filenames []string, operation string, column int, out io.Writer) error {
    // Create a channel to collect results from each goroutine
    resultChan := make(chan []float64, len(filenames))
    
    g, _ := errgroup.WithContext(context.Background())
    
    for _, fn := range filenames {
        filename := fn
        g.Go(func() error {
            file, err := os.OpenFile(filename, os.O_RDONLY, 0644)
            if err != nil {
                return os.ErrNotExist
            }
            defer file.Close()
            
            data, err := csv2float(file, column)
            if err != nil {
                return err
            }
            
            resultChan <- data  // Send data to channel instead of appending
            return nil
        })
    }
    
    // Wait for all goroutines and collect any errors
    if err := g.Wait(); err != nil {
        return err
    }
    
    // Close the channel now that all goroutines are done
    close(resultChan)
    
    // Collect all results from the channel
    var consolidate []float64
    for data := range resultChan {
        consolidate = append(consolidate, data...)
    }
    
    opResult := opFunc(consolidate)
    _, err := fmt.Fprintln(out, opResult)
    return err
}
```
