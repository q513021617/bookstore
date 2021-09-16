# 项目截图
![微信截图_20210916152816.png](http://cdn.zhiyigo.cn/bb1807ab5c4f4f58aeb2f8ce958e73ef微信截图_20210916152816.png)
## 开发语言 
go
## 开发框架 
无
## 项目依赖

```

github.com/go-sql-driver/mysql v1.5.0 h1:ozyZYNQW3x3HtqT1jira07DN2PArx2v7/mN66gGcHOs=

```
## 项目结构
![le5le.topology.png](http://cdn.zhiyigo.cn/4603f8c3c7a84b5ea208f3a92d44d757le5le.topology.png)


main.go文件内容如下:

```
package main //该包名

import (
	"bookstores2/controller" //项目中的controller包引入
	"net/http" //sdk内置 http库
	"fmt" 
)

//主方法初始化，处理静态资源
func main() {
	http.Handle("/static/",
		http.StripPrefix("/static/",http.FileServer(http.Dir("views/static/")))) //将 /static/映射成 服务器的文件目录路径并且去掉多余的static路径名

	http.Handle("/pages/",
		http.StripPrefix("/pages/",http.FileServer(http.Dir("views/pages/"))))
	//处理请求
	//用户相关
	http.HandleFunc("/main",controller.IndexHandler) //映射请求地址和对应的实现方法
	//图书相关
	http.HandleFunc("/getPageBooks",controller.GetPageBooks)
	http.HandleFunc("/toUpdateBookPage",controller.ToUpdateBookPage)
	http.HandleFunc("/deleteBook",controller.DeleteBookById)
	http.HandleFunc("/updateOraddBook",controller.AddOrUpdateBook)
	http.HandleFunc("/queryPrice",controller.QueryPrice)

	//订单相关（结账，发货，收货）
	http.HandleFunc("/checkout",controller.Checkout)	

	////获取SSL 证书和 RSA 私钥
	//utils.GetTLS("utils/pem/cert.pem","utils/pem/key.pem")
	
	fmt.Println("服务启动成功!服务器地址 http://127.0.0.1:8080/main")
	//设置服务器路径,使用默认多路服务器
	http.ListenAndServe(":8080",nil) //启动http服务器并设置服务器监听的ip和端口
}

```
#### IndexHandler

```
package controller

import (
	"bookstores2/dao" //项目中的数据层
	"bookstores2/model" //项目中的实体类
	"bookstores2/utils" // 项目中的工具类
	"fmt" //控制台输出
	"html/template" // 页面模板引擎
	"net/http" //http请求库
	"strconv"// 字符串库
)

func IndexHandler(w http.ResponseWriter, r *http.Request) {
	//mapValue := r.URL.Query() 获取所有url参数
	//pageNo := mapValue.Get("PageNo") 从map集合中获取 PageNo
	// ==
	pageNo := r.FormValue("PageNo") //从提交表单中获取 PageNo
	if pageNo == "" {
		pageNo = "1"
	}
	indexPage, _ := strconv.ParseInt(pageNo, 10, 64) //将 pageNo转换为 64位 10进制整数
	pages, err := dao.GetPageBooks(indexPage) //从数据库中查询 第几页的 书单并返回数据
	if err != nil {
		fmt.Println("分页查询全部图书出现异常,err：", err)
	}

	isLogin, session, errFindSession := dao.IsLogin(r) // 判断是否登录
	if !isLogin || session == nil { 
		fmt.Println("数据库中没查找到该session相关记录，err", errFindSession)
		pages.IsLogin = false
	} else {
		pages.IsLogin = true
		pages.Username = session.Username
	}
	t := template.Must(template.ParseFiles("views/index.html")) // 函数可以检测模板分析是否有错误，并返回解析好的模板对象
	t.Execute(w, pages) //模板变量赋值，并返回给浏览器
}

```

### bookDao 图书数据层

```
import (
	"bookstores2/model"
	"bookstores2/utils"
	"fmt"
)
//分页查询图书，
//返回的 Page 包含页码，该页包含的图书等信息
func GetPageBooks(IndexPage int64) ( *model.Page , error) {
	//查询总记录
	sqlStr := "select count(*) from books"
	var pageSize int64 = 4

	pages  ,count,err := getPage(sqlStr,pageSize)
	if err !=nil {
		fmt.Println("获取页数页码等出现异常，err:",err)
		return nil, err
	}
	//是否可以传参，不用每次都查询总记录数，减少IO?

	//分页查询
	sqlStr2  := "select * from books limit ? ,?"
	books ,err :=getBooksForPage(sqlStr2,(IndexPage -1) *pageSize,pageSize)
	if err != nil {
		fmt.Println("获取图书集出现异常，err:",err)
		return nil ,err
	}
	//将model中的Page属性赋值并将地址赋值给page
	page := &model.Page{
		Books: books,
		Pages: pages,
		PageSize: pageSize,
		IndexPage: IndexPage,
		Count: count,
	}
	return page,nil
}
```
#### golang中没有类的概念只能用结构体
```
type Book struct {
	ID int
	Title string   //书名
	Author string   //作者
	Price float64  //价格
	Sales int   //已销售
	Stock int   //库存
	ImagePath string  //图片路径

}

```
**Page实体类**
```
package model

type Page struct {
	Pages 		int64		//总页数
	PageSize	int64	//每页显示条数
	Count 		int64		//记录数
	IndexPage 	int64		//当前页
	Books 		[]*Book		//图书结果集
	MinPrice float64
	MaxPrice float64
	IsLogin bool
	Username string
}

func (p *Page)IsHasPrev() bool {
	return p.IndexPage > 1
}

func (p *Page)IsHasNext() bool {
	return p.IndexPage < p.Pages
}

func (p *Page)GetPrevPageNo() int64 {
	return p.IndexPage -1
}

func (p *Page)GetNextPageNo() int64 {
	return p.IndexPage +1
}

```

**数据库基础链接**
```
package utils

import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"
)

var (
	Db *sql.DB
	err error
)

//初始化数据库连接，获取数据库连接
func init()  {
	// ?parseTime=true&loc=Local 设置本地时区
	Db ,err = sql.Open("mysql",
					"root:root@tcp(localhost:3306)/bookstore?parseTime=true&loc=Local")
	if err != nil{
		panic(err)
	}
}

```

```
package utils

import (
	"crypto/md5"
	"fmt"
)

func Md5(unEncry string) (encryption string){
	byte16 := []byte(unEncry)
	encryption = fmt.Sprintf( "%x",md5.Sum(byte16) )
	return
}

```
**生成sls证书** 
```
package utils

import (
	rand "crypto/rand"
	"crypto/rsa"
	"crypto/x509"
	"crypto/x509/pkix"
	"encoding/pem"
	"math/big"
	"net"
	"os"
	"time"
)

//生成 SSL证书、RSA 私钥
func GetTLS(certPath , keyPath string)  {
	//生成序列号
	max := new(big.Int).Lsh(big.NewInt(1),128)
	serialNumber, _ := rand.Int(rand.Reader, max)

	// x509识别
	subject := pkix.Name{
		Organization: []string{"github.com/L1ng14"},
		OrganizationalUnit: []string{"l1ng14"},
		CommonName: "github.com",
	}

	// x509证书
	tempalte := x509.Certificate{
		SerialNumber: serialNumber,
		Subject: subject,
		NotBefore: time.Now(),
		NotAfter: time.Now().Add(365*24*time.Hour),
		KeyUsage: x509.KeyUsageDataEncipherment | x509.KeyUsageDigitalSignature,
		ExtKeyUsage: []x509.ExtKeyUsage{x509.ExtKeyUsageClientAuth},
		IPAddresses: []net.IP{net.ParseIP("127.0.0.1")},  // 证书只允许在 127.0.0.1上运行
	}

	//生成 RSA 私钥
	pk ,_ := rsa.GenerateKey(rand.Reader ,2048)

	// 公钥，生成 SSL 证书时需要用到
	darBtyes,_ := x509.CreateCertificate(rand.Reader,&tempalte,&tempalte,&pk.PublicKey,pk)

	//结合公钥，写入证书到相应路径
	certOut , _ := os.Create(certPath)
	defer certOut.Close()
	pem.Encode(certOut, &pem.Block{
		Type: "CERTIFICATE",
		Bytes: darBtyes,
	})


	//写入私钥,写入到相应路径
	keyOut ,_:= os.Create(keyPath)
	defer keyOut.Close()
	pem.Encode(keyOut,&pem.Block{
		Type: "RSA PRIVATE KEY",
		Bytes: x509.MarshalPKCS1PrivateKey(pk),
	})


}


```
**UUID生成**
```
package utils

import (
	"crypto/rand"
	"fmt"
	"log"
)

//CreateUUID 生成UUID
func CreateUUID() (uuid string) {
	u := new([16]byte)
	_, err := rand.Read(u[:])
	if err != nil {
		log.Fatalln("Cannot generate UUID", err)
	}
	u[8] = (u[8] | 0x40) & 0x7F
	u[6] = (u[6] & 0xF) | (0x4 << 4)
	uuid = fmt.Sprintf("%x-%x-%x-%x-%x", u[0:4], u[4:6], u[6:8], u[8:10], u[10:])
	return
}

```

获取总页数
```
func getPage(sqlStr string,pageSize int64,args ...interface{}) (pages int64 ,count int64,err error)   {
	//查询总记录
	res  :=utils.Db.QueryRow(sqlStr,args...)
	err = res.Scan(&count)
	if err != nil{
		fmt.Println("扫描输入 总记录数出现异常，err:",err)
		return 0,0,err
	}
	//计算总页数
	if count % pageSize == 0{
		pages = count / pageSize
	}else {
		pages = count / pageSize +1
	}
	return
}

```

### getBooksForPage 方法

```

//为查询到的图书集合赋值
func getBooksForPage(sqlStr string ,args ...interface{})( []*model.Book , error)  {
	rows ,errQuery := utils.Db.Query(sqlStr,args...) //通过内置数据库查询方法执行sql语句并返回结果
	if errQuery != nil{
		fmt.Println("分页获取部分记录出现异常，err:",errQuery)
		return nil ,errQuery
	}
	var books []*model.Book
	for rows.Next() {
		book := &model.Book{}
		errNotFound :=rows.Scan(&book.ID,&book.Title,&book.Author,&book.Price,&book.Sales,&book.Stock,&book.ImagePath)
		if errNotFound != nil{
			fmt.Println("赋值结果时出现异常，err :",errNotFound)
			return nil ,errNotFound
		}
		books = append(books, book)
	}//循环赋值每一个实体类对象
	return books,nil
}
```


# 总结

1,golang没有内置mysql驱动需要加入依赖去下载</br>
2,golang内置了web服务器，数据库操作api使得可以不需要任何框架即可编写一个简单的web服务</br>
3,项目地址: <a href="https://github.com/q513021617/bookstore">https://github.com/q513021617/bookstore</a>