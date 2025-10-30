
***

### 1. Counter with Mutex

```go
package main

import (
	"fmt"
	"sync"
)

type Counter struct{
	m map[int]int
	mux sync.Mutex
}

func (c *Counter) Inc(){
	c.mux.Lock()
	c.m[1]++
	c.mux.Unlock()
}

func (c *Counter) Get() int{
	return c.m[1]
}

func main(){
	c := Counter{
		m:make(map[int]int),
	}
	for i:=0;i<10000000;i++{
		go func ()  {
			c.Inc()
		}()
	}
	fmt.Println(c.Get())
}
```

***

### 2. Channel with Goroutines

```go
package main

import (
	"fmt"
	"time"
)

func put(val int,ch chan int){
	time.Sleep(time.Millisecond*500)
	ch <- val
}

func main(){
	ch := make(chan int,4)
	for i:= 1;i<5;i++{
		go put(i*i,ch)
	}
	for v := range 7{
		fmt.Println(v, <-ch)
	}
}
```

***

### 3. Goroutine with Context Timeout

```go
package main

import (
	"context"
	"log"
	"time"
)

func Api(ch chan int) {
	time.Sleep(500 * time.Millisecond)
	ch <- 100
}

func main(){
	cntx := context.Background()
	cntx,cancel := context.WithTimeout(cntx,100 * time.Millisecond)
	defer cancel()

	ch := make(chan int)
	go Api(ch)

	select{
	case <-cntx.Done():
		log.Fatal("Timeout....")
	case v := <-ch:
		log.Println("api is called and its value is ",v)
	}
}
```

***

### 4. Goroutines with WaitGroup (no channels)

```go
package main

import (
	"fmt"
	"sync"
)

func count(n int, msg string) {
	fmt.Printf("%d th %s\n", n, msg)
}

func main() {
	var wg sync.WaitGroup

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			count(i, "ani")
			wg.Done()
		}()
	}

	wg.Wait()
	fmt.Println("Done executing....")
}
```

***

### 5. Blocking Operation with Channel

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan int)
	go func() {
		time.Sleep(1 * time.Second)
		h := <-ch
		fmt.Println(h)
	}()
	ch <- 12
	close(ch)
}
```

***

### 6. Using Select Statement

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch1 := make(chan string)
	ch2 := make(chan string)

	go func() {
		for {
			time.Sleep(500 * time.Millisecond)
			ch1 <- "For every 0.5 sec. " + time.Now().Format("15:04:05.000")
		}
	}()

	go func() {
		for {
			time.Sleep(2000 * time.Millisecond)
			ch2 <- "For every 2 sec. " + time.Now().Format("15:04:05.000")
		}
	}()

	for {
		select {
		case x := <-ch1:
			fmt.Println(x)
		case y := <-ch2:
			fmt.Println(y)
		}
	}
}
```

***

### 7. Using Concurrency with Worker Pool

```go
package main

import (
	"fmt"
	"time"
)

func sq(i int) int{
	time.Sleep(500 * time.Millisecond)
	return i*i
}

func main(){
	jobs := make(chan int,300)
	result := make(chan int,300)
	t1 := time.Now()

	for i:= range 300{
		jobs <- i
	}

	go worker(jobs,result)
	go worker(jobs,result)
	go worker(jobs,result)
	go worker(jobs,result)
	go worker(jobs,result)
	go worker(jobs,result)
	go worker(jobs,result)
	go worker(jobs,result)
	go worker(jobs,result)
	go worker(jobs,result)
	go worker(jobs,result)
	go worker(jobs,result)
	go worker(jobs,result)
	go worker(jobs,result)
	go worker(jobs,result)
	go worker(jobs,result)
	
	for i:=1;i<=300;i++{
		fmt.Println(<-result)
	}

	fmt.Println(t1)
	fmt.Println(time.Now())
	fmt.Println(time.Since(t1))
}

func worker(jobs <-chan int, result chan <-int){
	for j := range jobs{
		result <- sq(j)
	}
}
```

***

### 8. Need of WaitGroup and Mutex

```go
package main

import (
	"container/list"
	"fmt"
	"sync"
	"time"
)

func main(){
	var wg sync.WaitGroup
	var m sync.Mutex

	l := list.New()

	for i:= 1;i<=100;i++{
		wg.Add(1)
		go func(v int){
			defer wg.Done()
			time.Sleep(100 * time.Millisecond)
			m.Lock()
			l.PushBack(v)
			m.Unlock()
		}(i)
	}

	wg.Wait()
	fmt.Println(l.Len())
}
```

---