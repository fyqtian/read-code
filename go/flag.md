### flag

```go
//example 
type userFlag struct {
	arr []string
}

func (u *userFlag) String() string {
	return strings.Join(u.arr, "-")
}

func (u *userFlag) Set(s string) error {
	u.arr = strings.Split(s, ",")
	return nil
}

func main() {
	fmt.Println(os.Args)
	f := flag.NewFlagSet(os.Args[0], flag.ContinueOnError)
	first := f.String("first", "", "input string")
	u := new(userFlag)
	f.Var(u, "slice", "string split by comma")
	err := f.Parse(os.Args[1:])
	fmt.Println(*first, err, u.arr)
}

//output
go run main.go  -first=dd -slice=a,b,c
Usage of C:\Users\ADMINI~1\AppData\Local\Temp\go-build355297882\b001\exe\main.exe:
dd <nil> [a b c]

```



```go
//CommandLine是一个默认得命令行集合,从os.Args解析
//快捷方法BoolVar, Arg等是对CommandLine得包装
var CommandLine = NewFlagSet(os.Args[0], ExitOnError)

// NewFlagSet返回一个空得flag集合带有指定得名字和错误处理属性
//如果名字不为空，默认得帮助信息和错误信息将会打印
func NewFlagSet(name string, errorHandling ErrorHandling) *FlagSet {
	f := &FlagSet{
		name:          name,
		errorHandling: errorHandling,
	}
	f.Usage = f.defaultUsage
	return f
}

//一个FlagSet代表定义得flags集合
//FlagSet中名字必须唯一，如果重复定义名字将会造成panic
type FlagSet struct {
    //Usage当解析flags发出错误时会调用。
    //字段是一个函数不是方法，将会被修改为指向一个用户定义得error方法
    //ErrorHandling设置决定了调用usage后得行为
    //对于CommandLine默认得ExitOnError，调用usage后退出
    
	Usage func()

	name          string
	parsed        bool
	actual        map[string]*Flag
	formal        map[string]*Flag
	args          []string // arguments after flags
	errorHandling ErrorHandling
	output        io.Writer // nil means stderr; use Output() accessor
}


func (f *FlagSet) StringVar(p *string, name string, value string, usage string) {
	f.Var(newStringValue(value, p), name, usage)
}

//StringVar定义了一个named的字符串flag，默认值，和使用内容
//参数p指向存储flag值一个字符串变量
func StringVar(p *string, name string, value string, usage string) {
	CommandLine.Var(newStringValue(value, p), name, usage)
}

//返回一个字符串指针指向flag的值
func (f *FlagSet) String(name string, value string, usage string) *string {
	p := new(string)
	f.StringVar(p, name, value, usage)
	return p
}

//Var定义了一个flag指定的名字和使用内容，
//flag类型和值由Value字段表示，同胞包含用户定义实现的value
//例如调用者创建一个Flag，将逗号分隔的字符串转换成一个切片
//通过给片赋值的方法来表示字符串。
//特别是Set将会分解逗号把字符串切割到切片
func (f *FlagSet) Var(value Value, name string, usage string) {
	//默认的value是一个字符串 不会改变
	flag := &Flag{name, usage, value, value.String()}
	_, alreadythere := f.formal[name]
    //判断是否已经定义过
	if alreadythere {
		var msg string
		if f.name == "" {
			msg = fmt.Sprintf("flag redefined: %s", name)
		} else {
			msg = fmt.Sprintf("%s flag redefined: %s", f.name, name)
		}
		fmt.Fprintln(f.Output(), msg)
		panic(msg) // Happens only if flags are declared with identical names
	}
	if f.formal == nil {
		f.formal = make(map[string]*Flag)
	}
	f.formal[name] = flag
}

func Parse() {
	// Ignore errors; CommandLine is set for ExitOnError.
	CommandLine.Parse(os.Args[1:])
}


// Parse parses flag definitions from the argument list, which should not
// include the command name. Must be called after all flags in the FlagSet
// are defined and before flags are accessed by the program.
// The return value will be ErrHelp if -help or -h were set but not defined.
//Parse通过arguments解析flag，argument不该包含command name,必须在定义所有的flag后在程序调用flag之前
//如果设置了-help或-h但未定义，则返回值将为ErrHelp。
func (f *FlagSet) Parse(arguments []string) error {
    //设置已经解析
	f.parsed = true
	f.args = arguments
	for {
		seen, err := f.parseOne()
		if seen {
			continue
		}
		if err == nil {
			break
		}
		switch f.errorHandling {
		case ContinueOnError:
			return err
		case ExitOnError:
			if err == ErrHelp {
				os.Exit(0)
			}
			os.Exit(2)
		case PanicOnError:
			panic(err)
		}
	}
	return nil
}

// parseOne parses one flag. It reports whether a flag was seen.
func (f *FlagSet) parseOne() (bool, error) {
	if len(f.args) == 0 {
		return false, nil
	}
	s := f.args[0]
    //如果长度为1 或者不是-开头 返回
	if len(s) < 2 || s[0] != '-' {
		return false, nil
	}
	numMinuses := 1
    //如果第二位还是-
	if s[1] == '-' {
		numMinuses++
        //s == --认为无效
		if len(s) == 2 { // "--" terminates the flags
			f.args = f.args[1:]
			return false, nil
		}
	}
    //截出剩余字符串
	name := s[numMinuses:]
    //如果长度为0 或者是---key 或者是--=value
	if len(name) == 0 || name[0] == '-' || name[0] == '=' {
		return false, f.failf("bad flag syntax: %s", s)
	}

	//认为这是一个flag
	f.args = f.args[1:]
	hasValue := false
	value := ""
    //找到=号在第几个下标
	for i := 1; i < len(name); i++ { // equals cannot be first
		if name[i] == '=' {
            //截出value
			value = name[i+1:]
			hasValue = true
            //截出key
			name = name[0:i]
			break
		}
	}
	m := f.formal
	flag, alreadythere := m[name] // BUG
    //如果没找到key
	if !alreadythere {
        //如果是help或者是h打印使用信息
		if name == "help" || name == "h" { // special case for nice help message.
			f.usage()
			return false, ErrHelp
		}
        //未定义panic
		return false, f.failf("flag provided but not defined: -%s", name)
	}
	//判断是不是bool类型
	if fv, ok := flag.Value.(boolFlag); ok && fv.IsBoolFlag() { // special case: doesn't need an arg
		if hasValue {
			if err := fv.Set(value); err != nil {
				return false, f.failf("invalid boolean value %q for -%s: %v", value, name, err)
			}
		} else {
            //没有设值 默认为true
			if err := fv.Set("true"); err != nil {
				return false, f.failf("invalid boolean flag %s: %v", name, err)
			}
		}
	} else {
		// It must have a value, which might be the next argument.
        //如果没有值并且剩余的arg>0 拿下一个arg顶上
		if !hasValue && len(f.args) > 0 {
			// value is the next arg
			hasValue = true
			value, f.args = f.args[0], f.args[1:]
		}
        //如果没有下一个flag打印错误信息
		if !hasValue {
			return false, f.failf("flag needs an argument: -%s", name)
		}
        //设置value
		if err := flag.Value.Set(value); err != nil {
			return false, f.failf("invalid value %q for flag -%s: %v", value, name, err)
		}
	}
	if f.actual == nil {
		f.actual = make(map[string]*Flag)
	}
    //解析成功的flag写入
	f.actual[name] = flag
	return true, nil
}
```

