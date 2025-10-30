# GO Routines and Channels

A goroutine is a lightweight thread managed by the Go runtime, not the operating system.
It’s how Go achieves concurrency — running many tasks independently, at the same time.

A goroutine is an independently executing function that can run concurrently with other goroutines.

You create it by adding the keyword go before a function call:

`go <function name>`


That single keyword tells Go:

“Run myFunction() in the background while the rest of the program continues.”


Every sqequential code that runs in main function will run only in the main go routine.

| Feature           | Goroutine                           | OS Thread                                |
| ----------------- | ----------------------------------- | ---------------------------------------- |
| Creation cost     | ~2 KB                               | ~1 MB                                    |
| Count             | Millions possible                   | Few thousand max                         |
| Managed by        | Go runtime                          | OS kernel                                |
| Scheduling        | Go’s M:N cooperative scheduler      | OS preemptive scheduler                  |
| Context switching | Done in user space, extremely cheap | Done in kernel space, expensive          |
| Stack             | Grows and shrinks dynamically       | Fixed size                               |
| Blocking I/O      | Non-blocking, goroutine gets parked | Thread blocked, consumes kernel resource |


every go program starts with main go rouytine. When the main function finishes or returns all the go routines are  abonded or collected by garbage collector.

## Don't communicate by sharing memory. share memory by communicating

`Don't communicate by sharing memory` - all other language share memory by using mutex , lock and syncronization acrros the variables.


`share memory by communicating` - share the data over channel in go so that only one routine can access the data. once the data is used by the routine it is removed from channel.



## What Are Channels in Go?

A channel in Go is like a pipe that allows goroutines to communicate with each other by sending and receiving values safely.

it is a special data type that is created using `make()` command

`variable := make(chan <type>)`

### `sending data into channel`

`ch <- data`

### `recieving data from channel`

`var := <-data`


### types

- `buffered channel` (size is specified while declaring)
- `unbuffered channel` (size is not declared while declaring)

### Unbuffered channel

```go
func main() {

	ch := make(chan int)

	go func() {
		time.Sleep(1 * time.Second)
		h := <-ch
		fmt.Println(h)
	}()

	ch <- 12

}
```

if we put data into the unbuffered channel before some routine ready to recieve it it makes a deadlock scenerio

like this.

```go
func main() {

	ch := make(chan int)

    ch <- 12 // no routine is ready to recieve

	go func() {
		time.Sleep(1 * time.Second)
		h := <-ch
		fmt.Println(h)
	}()
}
```

### Buffered channels

it works differently it allow to put values inside the channel untill the buffer is full. if some routine start consuming the data then we can again put the data inside it.


here is the syntax to declare buffered channel

```go
func put(val int,ch chan int){
	time.Sleep(time.Millisecond*500)
	ch <- val
}

func main(){

	ch := make(chan int,4)

	for i:= 1;i<5;i++{
		go put(i*i,ch)
	}

	for v := range 4{
		fmt.Println(v, <-ch)
	}

}
```

i this example we use put functions which waits for .5 sec and then puts the value inside it. we use go routine so the value is dropped inside the channel concorrectly.

once data is there we take that using loop. here we specified the size because of buffered channel.


## SCENERIO

```go

func main() {
	ch := make(chan int)
	ch <- 12     // (1) inserting data into channel
	h := <-ch    // (2) recieving data from channel
	fmt.Println(h)
}
```

this code causes deadlock

`reason:` here we use unbuffered channel inorder to put some data in the channel we need to make sure there is a reciever. we can only insert data if go finds there is a reciver waiting for the data since the reciever is in the next time go does not know there is a reciever while inserting so this causes dead lock.


`ch <- 12` code execution blocks here untill the reciever is ready.here the recever is next so program panics.

### solution 1

```go
func main() {
	ch := make(chan int, 1) // buffered channel
	ch <- 12
	h := <-ch
	fmt.Println(h)
}
```
use buffered channel because buffered channel has a storage like a queue so go knows there is a empty storage we can put the data when ever there is a reciever he can uswe it.

while unbuffered channel does not have a storage directing sending and recieveing.


### solution 2

```go
func main() {
	ch := make(chan int)
	go func() {
		ch <- 12
	}()
	h := <-ch
	fmt.Println(h)
}
```

this code uses go routine it uses routine to put data inside the channel . so the main routine is not blocked. and hence it knows there is a reciever and it works just fine.


```text
When a goroutine sends or receives on an unbuffered channel, Go’s runtime either instantly transfers the value if a partner exists — or parks the goroutine until a matching operation appears, then unparks both to continue.
```

----------

## SCENERIO
```go
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
		x := <-ch1 // blocks the main routine .5 sec
		y := <-ch2 // blocks the main rotine for 2 sec which makes the ch1 data recieve late to avoid this use select which is below
		fmt.Println(y) 
		fmt.Println(x)
	}
}
```

this is the simple code where there are 2 channels and 2 routines

routine 1 adds value to channel 1 for every `.5 sec` infinitely. 

routine 2 adds value to channel 2 for every `2 sec` infinitely.

when the `x<-ch1` executes it blocks the main go routine for .5 sec because for the first .5 sec there is no data inserted in the ch1 and the program waits in that line only for .5 sec. 

similarly the `y<-ch2` executes it blocks the main go routine for 2 sec because for the first 2 sec there is no data inserted in the ch2 and the program waits in that line only for 2 sec. even though the there are 4 values in the ch1 `.5 * 4 =. 2 sec ` we cannot consume the other data in the channel 1 because the `y<-ch` is a blocking call and it does not allow the to get remaining data from ch1 in the mean time 

#### output
```text
For every 2 sec. 23:59:04.263
For every 0.5 sec. 23:59:02.763
For every 2 sec. 23:59:06.264
For every 0.5 sec. 23:59:03.264
For every 2 sec. 23:59:08.265
For every 0.5 sec. 23:59:04.764
For every 2 sec. 23:59:10.266
For every 0.5 sec. 23:59:06.765
```

this is a big issue to avoid this we use  **`select`** stament which runs concurently and looking the data is available in the channel

```go
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

		select {
		case x := <-ch1:
			fmt.Println(x)
		case y := <-ch2:
			fmt.Println(y)
		}
}
```


#### output

```text
For every 0.5 sec. 00:11:26.483
For every 0.5 sec. 00:11:26.984
For every 0.5 sec. 00:11:27.485
For every 2 sec. 00:11:27.982
For every 0.5 sec. 00:11:27.985
For every 0.5 sec. 00:11:28.486
For every 0.5 sec. 00:11:28.986
For every 0.5 sec. 00:11:29.487
For every 2 sec. 00:11:29.983
For every 0.5 sec. 00:11:29.988
For every 0.5 sec. 00:11:30.489
For every 0.5 sec. 00:11:30.990
For every 0.5 sec. 00:11:31.491
```
here we can swee all the data is executed concurrently.



## closing a channel `close(ch)`

closing a channel tells go that there is not more data sent on this channel it notifies the range in for loop that there is no more data.

range without close will cause `deadlock`. because range did not know where the sending data is completed in the channel or not . It keeps on waiting thinking that data will be senf throh the channel.

we should use close(ch) for both buffered and unbufferd channels.

we can only use normal forloop to get data from buffered channel if we know the size of data inside it.

```go
func worker(ch <-chan int, done chan<- bool) {
    for n := range ch {
        fmt.Println("Processing", n)
    }
    done <- true
}

func main() {
    ch := make(chan int)
    done := make(chan bool)

    go worker(ch, done)

    for i := 1; i <= 5; i++ {
        ch <- i
    }
    close(ch) // important signal
    <-done
}
```


## SCENERIO

```go
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

This is the simple program where using wait groups is helput

## `what are wait groups?`

A WaitGroup in Go `(sync.WaitGroup)` is a synchronization mechanism that helps you wait for multiple goroutines to finish.

lets say we create many go routines each one take some time. we cannot close the main go routine. if we did not make the main routine to wait for all the routines that we have create then we dont have the desired result . Inorder to make the main go routine to wait till all the routines finsih their work we should use wait group


this basically works like

- before createing any go routine we should use `wg.Add(1)` where 1 is any random int.

- this registers a routine in wait group

- after that we have to create a go routine inside that we have to use `wg.Done()` to tell wg that task for this go routine has been completed

- wg internally works like a counter

- at last in the main function use  wg.Wait() which blocks the main go routine untill all the go routines are completed ( till counter indside becomes 0).

here is the example to demomonstartae

```go
func main(){
	var wg sync.WaitGroup
	var m sync.Mutex

	l := list.New()

	for i:= 1;i<=100;i++{
		wg.Add(1) // adding routine
		go func(v int){
			defer wg.Done() // removing routine/closing 
			time.Sleep(100 * time.Millisecond)
			m.Lock()
			l.PushBack(v)
			m.Unlock()
			
		}(i)
	}

	wg.Wait() // blocking main go routine till all other routine completes

	fmt.Println(l.Len())

}
```

here we are creating 100 routines and make them to add a value to the list.

**the reason why we have to use mutex here is list in go is not thread safe. if at the same time 2 values tries to push it inserts only one. so we lock each time we push a value so that no other routine can insert at that time since all the routines are runnign concurrently. and then unlock it.**

---


### without wg the length will be 0 because the main go routine gets executed before all the threads insets value in list.
```go
func main(){
	// var wg sync.WaitGroup
	var m sync.Mutex

	l := list.New()

	for i:= 1;i<=100;i++{
		// wg.Add(1)
		go func(v int){
			// defer wg.Done()
			time.Sleep(100 * time.Millisecond)
			m.Lock()
			l.PushBack(v)
			m.Unlock()
			
		}(i)
	}

	// wg.Wait()
	fmt.Println(l.Len())

}
```

---

## Speeding up task in go using jobs and workers

```go
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

this is the simele program

where we have jobs inside it we have to square a numbers for first 300 numbers. we need to calculate as soon as possible.

for that what we are doing is we create a 2 channels one for putting all the values to be squared which later on consumed bu multiple workers (routines) and square them and put them back in the result channel for final access.

we have created a worker func where we recieve the values and square them and put them in result routine.


first we inssert 300 values in jobs . and then we use multiple go routines to fast the process.

the more workers we have the fast we can compute.

```go
.
.
.
go worker(jobs,result)
go worker(jobs,result)
go worker(jobs,result)
go worker(jobs,result)
go worker(jobs,result)
go worker(jobs,result)
go worker(jobs,result)
go worker(jobs,result)
.
.
.
```

---

for using 8 workers it took 19 sec 

```sql
87616
89401
88209
88804
2025-10-31 01:01:15.867871 +0530 IST m=+0.000138459
2025-10-31 01:01:34.906659 +0530 IST m=+19.038856334
19.03872775s
```

---
using 12 workers it took 13 sec

```sql
88209
84681
85264
84100
83521
2025-10-31 01:03:19.244057 +0530 IST m=+0.000094418
2025-10-31 01:03:31.770709 +0530 IST m=+12.526700168
12.526615041s
```
---
using 16 workers it took 9 sec

```go
88209
85849
88804
2025-10-31 01:04:17.77591 +0530 IST m=+0.000163751
2025-10-31 01:04:27.295531 +0530 IST m=+9.519750042
9.519595791s
```

---
this is how fast we can make things
---
