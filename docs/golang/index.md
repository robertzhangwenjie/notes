# golang

## 打包
- 打包操作以... 开头，如... Type
```golang
func append(slice []Type, elems ...Type) []Type
```
`elems ...Type` 指的是将所有入参变量打包进首个参数之后的 elems 切片中

## 解包
- 解包操作符以... 结束如 slice ...
```golang
var slice1 = []int{1,2,3,4}
var slice2 = []int{5,6}
slice3 := append(slice1, slice2...)
fmt.Println(slice3)
```
> [1 2 3 4 5 6]

