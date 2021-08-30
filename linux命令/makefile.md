Makefile

[GNU make](https://www.gnu.org/software/make/manual/make.html)

[Makefile (github.com)](https://gist.github.com/isaacs/62a2d1825d04437c6f08)

```bash
$ make -f rules.txt
# 或者
$ make --file=rules.txt
```









```
target ... : prerequisites ...
    command
    ...
    ...
    
    

<target> : <prerequisites> 
[tab]  <commands>
```

**target**

可以是一个object file（目标文件），也可以是一个执行文件，还可以是一个标签（label）。对于标签这种特性，在后续的“伪目标”章节中会有叙述。

**prerequisites**

生成该target所依赖的文件和/或target

```
prerequisites中如果有一个以上的文件比target文件要新的话，command所定义的命令就会被执行。
```

**command**

该target要执行的命令（任意的shell命令）





一个目标**（target）**就构成一条规则。**目标通常是文件名**，指明Make命令所要构建的对象，比如上文的 a.txt 。目标可以是一个文件名，也可以是多个文件名，**之间用空格分隔。**

除了文件名，目标还可以是某个操作的名字，这称为"伪目标"（phony target）。

```
.PHONY: clean
clean:
        rm *.o temp
```



需要注意的是，每行命令在一个单独的shell中执行。这些Shell之间没有继承关系。

```
错误
var-lost:
    export foo=bar
    echo "foo=[$$foo]"
    
   
var-kept:
	export foo=bar; echo "foo=[$$foo]"
	
.ONESHELL:
var-kept:
    export foo=bar; 
    echo "foo=[$$foo]"
```



关闭回声 @



变量

```
txt = Hello World
v1 = $(txt) "abc"
test:
	echo ${txt}
	echo ${v1}
```



变量扩展

```bash
VARIABLE = value
# 在执行时扩展，允许递归扩展。

VARIABLE := value
# 在定义时扩展。

VARIABLE ?= value
# 只有在该变量为空时才设置值。

VARIABLE += value
# 将值追加到变量的尾端。
```



自动变量

&@

```bash
a.txt b.txt: 
    touch $@
    
等同于
a.txt:
    touch a.txt
b.txt:
    touch b.txt
```



### 判断和循环

```bash
a = 10
isTrue:
ifeq (${a},10)
	echo "yes"
endif


a = 101
isTrue:
ifeq (${a},10)
	echo "yes"
else
	echo "no"
endif



LIST = one two three
all:
	for i in $(LIST); do \
		echo $$i; \
	done
```





### 函数

```bash
srcfiles := $(shell echo src/{00..99}.txt)
```





### 调用外部参数

```
make FOO=bar  out
out:
	echo ${FOO}
```





```makefile
VARIABLE = value
# 在执行时扩展，允许递归扩展。

VARIABLE := value
# 在定义时扩展。

VARIABLE ?= value
# 只有在该变量为空时才设置值。

VARIABLE += value
# 将值追加到变量的尾端。
```





```
call函数是唯一一个可以用来创建新的参数化的函数。你可以写一个非常复杂的表达式，这个表达式中，你可以定义许多参数，然后你可以用call函数来向这个表达式传递参数。其语法是：

$(call <expression>;,<parm1>;,<parm2>;,<parm3>;...)
当make执行这个函数时，<expression>;参数中的变量，如$(1)，$(2)，$(3)等，会被参数< parm1>;，<parm2>;，<parm3>;依次取代。而<expression>;的返回值就是 call函数的返回值。例如：

reverse =  $(1) $(2)

foo = $(call reverse,a,b)
那么，foo的值就是“a b”。当然，参数的次序是可以自定义的，不一定是顺序的，如：

reverse =  $(2) $(1)

foo = $(call reverse,a,b)
此时的foo的值就是“b a”。
```

