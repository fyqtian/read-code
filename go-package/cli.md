### cli

仓库地址https://github.com/urfave/cli

```go
//example
package main

import (
   "fmt"
   "github.com/urfave/cli/v2"
   "os"
   "sort"
)

func main() {
   cmd := &cli.App{
      Name: "test",
      Flags: []cli.Flag{
         &cli.StringFlag{
            Name:  "lang",
            Value: "english",
            Usage: "language for the greeting",
         },
      },
      Commands: []*cli.Command{
         {
            Name:    "complete",
            Aliases: []string{"c"},
            Usage:   "complete a task on the list",
            Action: func(c *cli.Context) error {
               return nil
            },
         },
         {
            Name:    "add",
            Aliases: []string{"a"},
            Usage:   "add a task to the list",
            Action: func(c *cli.Context) error {
               return nil
            },
         },
      },
      Usage: "make an explosive entrance",
      Action: func(c *cli.Context) error {
         fmt.Println("boom! I say!")
         return nil
      },
   }
   sort.Sort(cli.FlagsByName(cmd.Flags))
   sort.Sort(cli.CommandsByName(cmd.Commands))
   cmd.Run(os.Args)
}
```





```go
func (a *App) Run(arguments []string) (err error) {
   return a.RunContext(context.Background(), arguments)
}


//RunContext和Run一样除了会传递Context到子命令，通过这样你可以传递超时或者取消请求
func (a *App) RunContext(ctx context.Context, arguments []string) (err error) {
	
    //Setup初始化数据结构
    a.Setup()

	
	shellComplete, arguments := checkShellCompleteFlag(a, arguments)
	//*flag.FlagSet
	set, err := a.newFlagSet()
	if err != nil {
		return err
	}
	//解析flag 会对-it这种类型 进行拆解 编程-i -t类型
    //设置UseShortOptionHandling=true
	err = parseIter(set, a, arguments[1:], shellComplete)
	nerr := normalizeFlags(a.Flags, set)
	context := NewContext(a, set, &Context{Context: ctx})
	if nerr != nil {
		_, _ = fmt.Fprintln(a.Writer, nerr)
		_ = ShowAppHelp(context)
		return nerr
	}
	context.shellComplete = shellComplete
	
	if checkCompletions(context) {
		return nil
	}
	//处理error
	if err != nil {
		if a.OnUsageError != nil {
			err := a.OnUsageError(context, err, false)
			a.handleExitCoder(context, err)
			return err
		}
		_, _ = fmt.Fprintf(a.Writer, "%s %s\n\n", "Incorrect Usage.", err.Error())
		_ = ShowAppHelp(context)
		return err
	}
	//flag设置help打印
	if !a.HideHelp && checkHelp(context) {
		_ = ShowAppHelp(context)
		return nil
	}
	//设置版本号打印
	if !a.HideVersion && checkVersion(context) {
		ShowVersion(context)
		return nil
	}
	//校验required字段
	cerr := checkRequiredFlags(a.Flags, context)
	if cerr != nil {
		_ = ShowAppHelp(context)
		return cerr
	}
	//收尾调用
	if a.After != nil {
		defer func() {
			if afterErr := a.After(context); afterErr != nil {
				if err != nil {
					err = newMultiError(err, afterErr)
				} else {
					err = afterErr
				}
			}
		}()
	}
	//调用前
	if a.Before != nil {
		beforeErr := a.Before(context)
		if beforeErr != nil {
			a.handleExitCoder(context, beforeErr)
			err = beforeErr
			return err
		}
	}

	args := context.Args()
    //命令行剩下的参数
	if args.Present() {
		name := args.First()
    	//调用command
		c := a.Command(name)
        //存在Command调用方法
		if c != nil {
            //子命令处理
			return c.Run(context)
		}
	}
	
	if a.Action == nil {
		a.Action = helpCommand.Action
	}

	// Run default Action
	err = a.Action(context)

	a.handleExitCoder(context, err)
	return err
}

```