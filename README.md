# argparse

argparser inspired by [python argparse](https://docs.python.org/3.9/library/argparse.html)

provide not just simple parse args, but :

- [x] sub command
- [x] argument groups
- [x] positional arguments
- [x] optimizable parse method
- [x] optimizable validate checker
- [x] argument choice support
- [ ] ......

## Installation

```bash
go get github.com/hellflame/argparse
```

> no dependence needed

## Usage

just go:

```go
package main

import (
    "fmt"
    "github.com/hellflame/argparse"
)

func main() {
    parser := argparse.NewParser("basic", "this is a basic program", nil)
  
    name := parser.String("n", "name", nil)
  
    if e := parser.Parse(nil); e != nil {
        fmt.Println(e.Error())
      	return
    }
    fmt.Printf("hello %s\n", *name)
}
```

[example](examples/basic)

checkout output:

```bash
=> go run main.go
usage: basic [-h] [-n NAME]

this is a basic program

optional arguments:
  -h, --help            show this help message
  -n NAME, --name NAME

=> go run main.go -n hellflame
hello hellflame
```

a few point:

about object __parser__ :

1. `NewParser` first argument is the name of your program, but it's ok __not to fill__ , when it's empty string, program name will be `os.Args[0]` , which can be wired when using `go run`, but it will be you executable file's name when you release the code. It can be convinient where the release name is uncertain
2. `help` function is auto injected, but you can disable it when `NewParser`, with `&ParserConfig{DisableHelp: true}`. then you can use any way to define the `help` function, or whether to `help` 
3. when `help` showed up, the program will default __exit with code 1__ , this is stoppable by setting `ParserConfig.ContinueOnHelp`  to by `true`, or just use your own help function instead

about __parse__ action:

1. the argument `name` is only usable __after__  `parser.Parse` , or there might be errors happening
2. when passing `parser.Parse` a `nil` as argument, `os.Args[1:]` is used as parse source, so notice to __only pass arguments__ to `parser.Parse` , or the program name may be `unrecognized`
3. the short name of your agument can be __more than one character__

---

based on these points, the code can be like this:

```go
func main() {
    parser := argparse.NewParser("", "this is a basic program", &argparse.ParserConfig{
        DisableHelp:true,
        DisableDefaultShowHelp: true})
  
    name := parser.String("n", "name", nil)
    help := parser.Flag("help", "help-me", nil)
  
    if e := parser.Parse(os.Args[1:]); e != nil {
        fmt.Println(e.Error())
        return
    }
    if *help {
        parser.PrintHelp()
        return
    }
    if *name != "" {
        fmt.Printf("hello %s\n", *name)
    }
}
```

[example](examples/basic-bias)

check output:

```bash
=> go run main.go

=> go run main.go -h
unrecognized arguments: -h

# the real help entry is -help / --help-me
=> go run main.go -help
usage: /var/folq1pddT/go-build42601/exe/main [-n NAME] [-help]

this is a basic program

optional arguments:
  -n NAME, --name NAME
  -help, --help-me

# still functional
=> go run main.go --name hellflame
hello hellflame
```

a few points:

1. `DisableHelp` only avoid `-h/--help` flag to register to parser, but the `help` is still fully functional
2. if keep `DisableDefaultShowHelp` to be false, where there is no argument, the `help` function will still show up
3. after the manual call of `parser.PrintHelp()` , program goes on
4. notice the order of usage array, it's mostly the order of your arguments

## Config

### 1. parser config

relative struct: 

```go
type ParserConfig struct {
	Usage                  string // manual usage display
	EpiLog                 string // message after help
	DisableHelp            bool   // disable help entry register [-h/--help]
	ContinueOnHelp         bool   // set true to: continue program after default help is printed
	DisableDefaultShowHelp bool   // set false to: default show help when there is no args to parse (default action)
}
```

eg:

```go
func main() {
	parser := argparse.NewParser("basic", "this is a basic program",
		&argparse.ParserConfig{
			Usage:                  "basic xxx",
			EpiLog:                 "more detail please visit https://github.com/hellflame/argparse",
			DisableHelp:            true,
			ContinueOnHelp:         true,
			DisableDefaultShowHelp: true,
		})
  
	name := parser.String("n", "name", nil)
	help := parser.Flag("help", "help-me", nil)
  
	if e := parser.Parse(nil); e != nil {
		fmt.Println(e.Error())
		return
	}
	if *help {
		parser.PrintHelp()
		return
	}
	if *name != "" {
		fmt.Printf("hello %s\n", *name)
	}
}
```

[example](examples/parser-config)

output:

```bash
=> go run main.go
# there will be no help message
# affected by DisableDefaultShowHelp

=> go run main.go --help-me
usage: basic xxx   # <=== Usage

this is a basic program

optional arguments: # no [-h/--help] flag is registerd, which is affected by DisableHelp
  -n NAME, --name NAME
  -help, --help-me 

more detail please visit https://github.com/hellflame/argparse  # <=== EpiLog
```

except the comment above, `ContinueOnHelp` is only affective on your program process, which give you possibility to do something when default `help` is shown

### 2. argument options

related struct:

```go
type Option struct {
	Meta       string // meta value for help/usage generate
	multi      bool   // take more than one argument
	Default    string // default argument value if not given
	isFlag     bool   // use as flag
	Required   bool   // require to be set
	Positional bool   // is positional argument
	Help       string // help message
	Group      string // argument group info, default to be no group
	Choices    []interface{}  // input argument must be one/some of the choice
	Validate   func(arg string) error  // customize function to check argument validation
	Formatter  func(arg string) (interface{}, error) // format input arguments by the given method
}
```

## How it works

```
  ┌──────────────────────┐ ┌──────────────────────┐
  │                      │ │                      │
  │     OptionArgsMap    │ │  PositionalArgsList  │
  │                      │ │                      │
  │      -h ───► helpArg │ │                      │
  │                      │ │[  posArg1  posArg2  ]│
  │      -n ──┐          │ │                      │
  │           │► nameArg │ │                      │
  │  --name ──┘          │ │                      │
  │                      │ │                      │
  └──────────────────────┘ └──────────────────────┘
             ▲ yes                  no ▲
             │                         │
             │ match?──────────────────┘
             │
             │
           ┌─┴──┐                   match helpArg:
    args:  │ -h │-n  hellflame           ▼
           └────┘                  ┌──isflag?───┐
                                   ▼            ▼
                                  done ┌──MultiValue?───┐
                                       ▼                ▼
                                   ┌──parse     consume untill
                                   ▼            NextOptionArgs
                                 done
```



## Examples

