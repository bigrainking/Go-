



> 本文根据本目录下原来笔记**[go搭建web服务器](../2go搭建web服务器.md)** 重新理解并总结。 （着重于Handle、HandleFunc、Handler）
>
> 2022.05.05



各个函数官网解释

[Handler路由](https://pkg.go.dev/net/http?utm_source=gopls#Handler)

[HandleFunc](https://pkg.go.dev/net/http?utm_source=gopls#HandleFunc)

[Handle](https://pkg.go.dev/net/http?utm_source=gopls#Handle)





# 一、handler路由器



### 1.1 Handler

路由器

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

- Handler 是路由器接口

  创建一个Handler实例，就是创建一个路由器

- `ServeHTTP()` 将回应内容(header & data)写到ResponseWriter中。



**实现一个路由器**

```go
// 创建实例
type index struct {
	Title string
	Data  string
}
//实现接口方法
func (h *index) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 注意必须是指针类型的
	fmt.Fprint(w, h.Title, h.Data)
}

func main() {
	h := &index{"永辉超市", "每样亿元"} //创建路由
	if err := http.ListenAndServe(":8081", h); err != nil {
		log.Println("出错了")
	}
}
```



### 1.2 Handle()

```go
func Handle(pattern string, handler Handler)
```

将handler绑定到对应的pattern(路径)，并注册到路由表中。  路由表是`DefaultServeMux`(后面解释)

**Example**：

```go
//创建实例
type han struct { 
	Title string
}
//实现handler的接口
func (h *han) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Title:%s", h.Title)
}

func main() {
	http.Handle("/eat", &han{"eat"}) //成为一个handler，注册路由
	http.Handle("/sleep", &han{"sleep"}) 
	http.ListenAndServe(":8080", nil)
}

====================输出================================
访问http://127.0.0.1:8080/sleep
Title:sleep
访问http://127.0.0.1:8080/eat
Title:eat
```



**为什么要使用Handle()?**

> 有上面的 ListenAndServe(pattern string, handler Handler) 可以直接绑定监听

因为一个ip地址下有多个网页则需要多个路由(Handler)， 我们可以用一个Handler路由处理多个页面`switch case`（具体见下面`解决方法1：Switch case`）这样只用监听一个Handler。

- 但我们希望每个页面对应的Handler分开。

  因此，需要一个Handler集合器：多路复用器`ServeMux`：`见解决方法2:多路复用器`

- 由于ServeMux使用较为麻烦，有个默认的全局集合器：DefaultServeMux更加方便。

  `http.ListenAndServe(":8080", nil)` 中`Handler = nil` 则直接绑定到`DefaultServeMux` 上面。 并且每次Handle() 都直接将路由器注册到全局集合器上。



### 1.3 HandleFunc()

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request))
```

函数作用与上面的`Handle()` 相同

- 但 `Handle()`需要自己实现Handler接口。
- 本函数简化了实现接口部分， 只需要实现一个`func(ResponseWriter, *Request)`的func即可。【实际上这个func与handler接口中的ServeHTTP相似】

 

**example**

```go
//一个固定的普通的function
func handIndex(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "title: %s", r.Host)
}

func main() {
	http.HandleFunc("/", handIndex)
	http.ListenAndServe(":8080", nil)
}
```











# 二、附录

#### 解决方法1：Switch case

Q：如果我们有多个页面需要处理呢？比如进入:8001/mike页面只显示牛奶的商品， 进入:8001/eat 显示其他的商品信息。实现进入不同的页面。



下面有两种方法：

1. 修改了原来的Handler， 使其可以处理多种请求， 分析Client的请求， 不同的请求选择不同的处理方式。
2. 使用多路复用器



**解决方法一**：用switch-case， 分析request， 根据不同的request的网址进行不同的处理

- 修改了原来的Handler， 使其可以处理多种请求

```go
func (db *DB) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	switch r.URL.Path { // 分析request
	case "/list":
		for name := range *db {
			fmt.Fprintf(w, "商品: %s \n", name)
		}
	case "/price":
		for _, price := range *db {
			fmt.Fprintf(w, "价格 ： %d \n", price)
		}
	default:
		fmt.Fprintln(w, "欢迎光临全家！")
	}
}
```



<img src="image/3 go如何让web工作.pic/image-20210908120718627.png" alt="image-20210908120718627" style="zoom: 25%;" />

<img src="image/3 go如何让web工作.pic/image-20210908120751023.png" alt="image-20210908120751023" style="zoom:25%;" />

<img src="image/3 go如何让web工作.pic/image-20210908120845804.png" alt="image-20210908120845804" style="zoom:25%;" />





#### **解决方法2：多路复用器**

每次增加一个需求，都去修改Handler是一件麻烦的事情， 我们使用一个多路复用器ServeMux。 net/http提供的ServeMux可以将多个Handler聚合起来，我们可以根据不同的URL路径设置不停的Handler.



下面代码中，我们将两个不同的Handler聚集在一起，最后将这个Handler的集合ServeMux放入ListenAndServe()中。

```go
type DB map[string]int

func (db DB) list(w http.ResponseWriter, r *http.Request) {
}

func (db DB) price(w http.ResponseWriter, r *http.Request) {
}

func main() {
	db := DB{"牛奶": 65, "奶皮": 99, "奶豆腐": 888} // handler接口实例
	mux := http.NewServeMux() // new一个ServeMux
	mux.Handle("/list", http.HandlerFunc(db.list)) // 添加Handler
	mux.Handle("/price", http.HandlerFunc(db.price))
	http.ListenAndServe(":8001", mux)
}
```

- `mux.Handle("/price", http.HandlerFunc(db.price))` 作用是

  将自定义的Handler`http.HandlerFunc(db.price)  `  绑定到路径  `/price`上

- Handle源码：

  ```go
  func (mux *ServeMux) Handle(pattern string, handler Handler) {
  	mux.mu.Lock()
  defer mux.mu.Unlock()
  
  	if pattern == "" {
  		panic("http: invalid pattern")
  	}
  	if handler == nil {
  		panic("http: nil handler")
  	}
  	if _, exist := mux.m[pattern]; exist {
  		panic("http: multiple registrations for " + pattern)
  	}
  
  	if mux.m == nil {
  		mux.m = make(map[string]muxEntry)
  	}
  	e := muxEntry{h: handler, pattern: pattern}
  	mux.m[pattern] = e
  	if pattern[len(pattern)-1] == '/' {
  		mux.es = appendSorted(mux.es, e)
  	}
  
  	if pattern[0] != '/' {
  		mux.hosts = true
  	}
  }
  ```

  

#### DefaultServeMux

显示地将每一个Handler聚集起来， 并传入ListenAndServe中是一件麻烦的事情。

net/http包提供了**全局的ServeMux**即 `DefaultServeMux` 和**包级别的Mux.Handle**，可以不用传入ListenAndServe

- 当`ListenAndServe(":8001", nil)` 中参数是nil时， 就默认调DefaultServeMux作为参数

上面的多路复用器的代码可以改成如下：

```go
func main() {
	db := DB{"牛奶": 65, "奶皮": 99, "奶豆腐": 888} // handler接口实例
    http.HandleFunc("/list", db.list)
    http.HandleFunc("/price", db.price)
	http.ListenAndServe(":8001", nil)
}
```

现在的代码与最开始搭建服务器的代码类似了。