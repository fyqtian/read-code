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

// unmarshal json到一个value实现Unmarshaler interface,
// Unmarshal调用value的UnmarshalJson方法。包括当输入的json是null
// 否则如果value实现encoding.TextUnmarshaler并且输入的是带引号的字符串，Unmarshal调用value对象的UnsharmalText方法
// 参数是不带引号的字符串

// unmarshal json到一个struct,Unmarshal匹配输入的对象key，首选完全匹配，但也接受不区分大小写的匹配
// 默认情况下没有对象struct字段的忽略
//
/
// To unmarshal JSON into an interface value,
// Unmarshal stores one of these in the interface value:
//
// bool, for JSON booleans
// float64, for JSON numbers
// string, for JSON strings
// []interface{}, for JSON arrays
// map[string]interface{}, for JSON objects
// nil for JSON null
//
// To unmarshal a JSON array into a slice, Unmarshal resets the slice length
// to zero and then appends each element to the slice.
// As a special case, to unmarshal an empty JSON array into a slice,
// Unmarshal replaces the slice with a new empty slice.
//
// To unmarshal a JSON array into a Go array, Unmarshal decodes
// JSON array elements into corresponding Go array elements.
// If the Go array is smaller than the JSON array,
// the additional JSON array elements are discarded.
// If the JSON array is smaller than the Go array,
// the additional Go array elements are set to zero values.
//
// To unmarshal a JSON object into a map, Unmarshal first establishes a map to
// use. If the map is nil, Unmarshal allocates a new map. Otherwise Unmarshal
// reuses the existing map, keeping existing entries. Unmarshal then stores
// key-value pairs from the JSON object into the map. The map's key type must
// either be any string type, an integer, implement json.Unmarshaler, or
// implement encoding.TextUnmarshaler.
//
// If a JSON value is not appropriate for a given target type,
// or if a JSON number overflows the target type, Unmarshal
// skips that field and completes the unmarshaling as best it can.
// If no more serious errors are encountered, Unmarshal returns
// an UnmarshalTypeError describing the earliest such error. In any
// case, it's not guaranteed that all the remaining fields following
// the problematic one will be unmarshaled into the target object.
//
// The JSON null value unmarshals into an interface, map, pointer, or slice
// by setting that Go value to nil. Because null is often used in JSON to mean
// ``not present,'' unmarshaling a JSON null into any other Go type has no effect
// on the value and produces no error.
//
// When unmarshaling quoted strings, invalid UTF-8 or
// invalid UTF-16 surrogate pairs are not treated as an error.
// Instead, they are replaced by the Unicode replacement
// character U+FFFD.
//
func Unmarshal(data []byte, v interface{}) error {
   // Check for well-formedness.
   // Avoids filling out half a data structure
   // before discovering a JSON syntax error.
   var d decodeState
   err := checkValid(data, &d.scan)
   if err != nil {
      return err
   }

   d.init(data)
   return d.unmarshal(v)
}
```

