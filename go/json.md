encodeing/json



```go
//example
func main() {
   type name struct {
      Age int
   }
   j := []byte(`{"age":10}`)
   n := new(name)
   json.Unmarshal(j, n)
   fmt.Println(n)
}
//output
&{10}
```





```go
// Unmarshal解析json编码的数据并存储再结果指针里，如果v是nil或者不是pointer返回InvalidUnmarshalError
// Unmarshal解码数据，根据需要分配map，slices，和指针是必须的
// 根据下面的规则
// unmarshal json数据到point,Unmarshal首先处理json语义null，再这个例子中，Unmarshal设置指针为nil
// 否则指针指向申请的一个新值

// unmarshal json到一个value实现Unmarshaler interface.md,
// Unmarshal调用value的UnmarshalJson方法。包括当输入的json是null
// 否则如果value实现encoding.TextUnmarshaler并且输入的是带引号的字符串，Unmarshal调用value对象的UnsharmalText方法
// 参数是不带引号的字符串

// unmarshal json到一个struct,Unmarshal匹配输入的对象key，首选完全匹配，但也接受不区分大小写的匹配
// 默认情况下没有对象struct字段的忽略

// unmarshal json到一个interface value,Unmarshal存储在接口值中
// bool, for JSON booleans
// float64, for JSON numbers
// string, for JSON strings
// []interface.md{}, for JSON arrays
// map[string]interface.md{}, for JSON objects
// nil for JSON null

// unmarshal json数组到一个slice， Unmarshal重置slice的长度到0然后append每个元素到slice
// 作为特列，unmarshal一个空的json数组到slice，Unmarshal用一个新的slice替换
// unmarshal json数组到go数组，Unmarhsal决定json数组的元素符合go数组的元素
//如果go数组比JSON数组小，多余的json数组元素会被丢弃，如果json数组比go数组小，多余的go数组元素空值
//
// unmarshal josn对象到map，Unmarshal首先建立一个map使用，如果map为nil，Unmarshal分配一个新的map
// 否则复用存在的map，保存原有的内容，Unmarshal存储key value对到map，map的key类型必须是字符串，数字，或者实现json.Unmarshaler
// encoding.TextUnmarshaler.
//
// 如果json的值和给定目标的类型不相符合，或者一个json number对目标类型溢出，Unmarshal 跳过这个字段并尽最大可能完成反编码
// 如果没有更多的严重错误发生, Unmarshal返回一个UnmarshalTypeError描述先前发生的错误
// 在任何例子 它都不能保证剩下的字段遵守有问题的将会呗解析到目标对象

//  json null值解析到interface,map,point,或者slice为nil，因为null在json中意味着不存在，
// unmarshing json null到其他任何go类型没有任何作用

// 当unmarshaling引号字符串，无效的UTF-8或者无效的UTF-16，不会被认为错误
// 会被替换成Unicode字符U+FFFD
func Unmarshal(data []byte, v interface{}) error {
	// 校验json格式 防止结构填充到一半发生错误
   var d decodeState
   err := checkValid(data, &d.scan)
   if err != nil {
      return err
   }
   d.init(data)
   return d.unmarshal(v)
}

func (d *decodeState) init(data []byte) *decodeState {
    d.data = data
    d.off = 0
    d.savedError = nil
    d.errorContext.Struct = nil

    // Reuse the allocated space for the FieldStack slice.
    d.errorContext.FieldStack = d.errorContext.FieldStack[:0]
    return d
}

func (d *decodeState) unmarshal(v interface{}) error {
    rv := reflect.ValueOf(v)
    //如果不是至真或者是nil
    if rv.Kind() != reflect.Ptr || rv.IsNil() {
        return &InvalidUnmarshalError{reflect.TypeOf(v)}
    }
    d.scan.reset()
    //重新开始扫描 找到第一个不是空格的操作符{|[
    d.scanWhile(scanSkipSpace)
    // We decode rv not rv.Elem because the Unmarshaler interface.md
    // test must be applied at the top level of the value.
    err := d.value(rv)
    if err != nil {
        return d.addErrorContext(err)
    }
    return d.savedError
}

// value消费一个json从d.data[d.off-1:],如果v是无效的，value将被丢弃，第一个value的byte已经被读取
func (d *decodeState) value(v reflect.Value) error {
    switch d.opcode {
        default:
        	panic(phasePanicMsg)
        //处理数组
        case scanBeginArray:
            if v.IsValid() {
                if err := d.array(v); err != nil {
                    return err
                }
            } else {
                d.skip()
            }
            d.scanNext()
        //处理对象
        case scanBeginObject:
            if v.IsValid() {
            	//解析json
                if err := d.object(v); err != nil {
                    return err
                }
            } else {
                d.skip()
            }
            d.scanNext()

            case scanBeginLiteral:
            // All bytes inside literal return scanContinue op code.
            start := d.readIndex()
            d.rescanLiteral()

            if v.IsValid() {
                if err := d.literalStore(d.data[start:d.readIndex()], v, false); err != nil {
                    return err
                }
            }
        }
    return nil
}





```

