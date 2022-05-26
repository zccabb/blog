---
title: golang写一个简单的网页应用
date: 2020-06-28 16:19:02
categories: Golang
tags:
  - golang基础
  - net
- 
---

下边的程序在端口 8088 上启动了一个网页服务器；SimpleServer() 会处理 url/test1 使它在浏览器输出 hello world。FormServer 会处理 url/test2：如果 url 最初由浏览器请求，那么它是一个 GET 请求，返回一个 form 常量，包含了简单的 input 表单，这个表单里有一个文本框和一个提交按钮。当在文本框输入一些东西并点击提交按钮的时候，会发起一个 POST 请求。FormServer() 中的代码用到了 switch 来区分两种情况。请求为 POST 类型时，name 属性为 inp 的文本框的内容可以这样获取：request.FormValue("inp")。然后将其写回浏览器页面中。在控制台启动程序，然后到浏览器中打开 url http://localhost:8088/test2 来测试这个程序：

```golang
package main

import (
	"io"
	"net/http"
)

const form = `
	<html><body>
		<form action="#" method="post" name="bar">
			<input type="text" name="in" />
			<input type="submit" value="submit"/>
		</form>
	</body></html>
`

/* handle a simple get request */
func SimpleServer(w http.ResponseWriter, request *http.Request) {
	io.WriteString(w, "<h1>hello, world</h1>")
}

func FormServer(w http.ResponseWriter, request *http.Request) {
	w.Header().Set("Content-Type", "text/html")
	switch request.Method {
	case "GET":
		/* display the form to the user */
		io.WriteString(w, form)
	case "POST":
		/* handle the form data, note that ParseForm must
		   be called before we can extract form data */
		//request.ParseForm();
		//io.WriteString(w, request.Form["in"][0])
		io.WriteString(w, request.FormValue("in"))
	}
}

func main() {
	http.HandleFunc("/test1", SimpleServer)
	http.HandleFunc("/test2", FormServer)
	if err := http.ListenAndServe(":8088", nil); err != nil {
		panic(err)
	}
}
```
注：当使用字符串常量表示 html 文本的时候，包含 <html><body>...</body></html> 对于让浏览器将它识别为 html 文档非常重要。

更安全的做法是在处理函数中，在写入返回内容之前将头部的 content-type 设置为 text/html：w.Header().Set("Content-Type", "text/html")。

"Content-Type" 会让浏览器认为它可以使用函数 http.DetectContentType([]byte(form)) 来处理收到的数据。


编写一个网页程序，可以让用户输入一连串的数字。计算出这些数字的均值和中值，并且打印出来

```golang
package main

import (
	"fmt"
	"io"
	"log"
	"net/http"
	"sort"
	"strconv"
	"strings"
)

type opera struct {
	numbers []float64
	mean    float64
	median  float64
}

const form = `
		<html><body>
			<form action="/" method="post">
				<laber for="numbers">Numbers (use comma to separate):</label>
				<input type="text" name="numbers" />
				<input type="submit" value="submit" />
			</form>
		</body></html>
`
const error = `<p>Error: %s</p>`

var pageTop = ""
var pageBottom = ""

func main() {
	http.HandleFunc("/", SimpleOperServer)
	if err := http.ListenAndServe(":9999", nil); err != nil {
		log.Fatal(err)
	}
}
func SimpleOperServer(w http.ResponseWriter, request *http.Request) {

	w.Header().Set("Content-Type", "text/html")
	err := request.ParseForm()
	io.WriteString(w, "Statistics\n")
	fmt.Fprint(w, pageTop, form)
	if err != nil {
		fmt.Fprintf(w, error, err)
	} else {
		if numbers, message, ok := processRequest(request); ok {
			stats := getStats(numbers)
			fmt.Fprintf(w, formatOpera(stats))
		} else if message != "" {
			fmt.Fprintf(w, error, message)
		}
	}
	fmt.Fprintf(w, pageBottom)

}
func processRequest(request *http.Request) ([]float64, string, bool) {
	var numbers []float64
	var text string
	if slice, found := request.Form["numbers"]; found && len(slice) > 0 {
		if strings.Contains(slice[0], "，") {
			text = strings.Replace(slice[0], "，", " ", -1)
		} else {
			text = strings.Replace(slice[0], ",", " ", -1)
		}
		for _, field := range strings.Fields(text) {
			if x, err := strconv.ParseFloat(field, 64); err != nil {
				return numbers, "'" + field + "' is invalid", false
			} else {
				numbers = append(numbers, x)
			}
		}
	}
	if len(numbers) == 0 {
		return numbers, "", false
	}
	return numbers, "", true
}
func getStats(numbers []float64) (operation opera) {
	operation.numbers = numbers
	sort.Float64s(operation.numbers)
	operation.mean = sum(operation.numbers)
	operation.median = median(numbers)
	return
}
func sum(numbers []float64) (mean float64) {
	for _, x := range numbers {
		mean = mean + x
	}
	return
}
func median(numbers []float64) (median float64) {
	middle_pos := len(numbers) / 2
	if len(numbers)%2 == 0 {
		median = (numbers[middle_pos] + numbers[middle_pos+1]) / 2
		return
	}
	median = numbers[middle_pos]
	return
}
func formatOpera(oper opera) string {
	return fmt.Sprintf(`<table border="1">
	<tr><th colspan="2">Results</th></tr>
	<tr><td>Numbers</td><td>%v</td></tr>
	<tr><td>Count</td><td>%d</td></tr>
	<tr><td>Mean</td><td>%f</td></tr>
	<tr><td>Median</td><td>%f</td></tr>
	</table>`, oper.numbers, len(oper.numbers), oper.mean, oper.median)
}


```
