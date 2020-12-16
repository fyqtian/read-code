### ansi

https://github.com/mgutz/ansi

windows shell支持颜色

```go
package main

import "github.com/mgutz/ansi"
import "fmt"

func main() {
   // colorize a string, SLOW
   msg := ansi.Color("foo", "red+b:white")

   // create a FAST closure function to avoid computation of ANSI code
   fmt.Println([]byte(msg))
   fmt.Println(msg, "Bring back the 80s!")
}
```

```go
var (
	plain = false
	// Colors maps common color names to their ANSI color code.
	Colors = map[string]int{
		"black":   black,
		"red":     red,
		"green":   green,
		"yellow":  yellow,
		"blue":    blue,
		"magenta": magenta,
		"cyan":    cyan,
		"white":   white,
		"default": defaultt,
	}
)

// ColorCode returns the ANSI color color code for style.
func ColorCode(style string) string {
	return colorCode(style).String()
}

func Color(s, style string) string {
    //plain default == false
   if plain || len(style) < 1 {
      return s
   }
   buf := colorCode(style)
   buf.WriteString(s)
   //换码符
   buf.WriteString(Reset)
   return buf.String()
}

// Gets the ANSI color code for a style.
//example style = "red+b:white"
func colorCode(style string) *bytes.Buffer {
	buf := bytes.NewBufferString("")
	if plain || style == "" {
		return buf
	}
    // Reset is the ANSI reset escape sequence
	//Reset = "\033[0m"
	if style == "reset" {
		buf.WriteString(Reset)
		return buf
	} else if style == "off" {
		return buf
	}
	
	foregroundBackground := strings.Split(style, ":")
    //["red","b"]
	foreground := strings.Split(foregroundBackground[0], "+")
    //red
	fgKey := foreground[0]
    //1
	fg := Colors[fgKey]
    //b
	fgStyle := ""
	if len(foreground) > 1 {
		fgStyle = foreground[1]
	}
	//  white
	bg, bgStyle := "", ""

	if len(foregroundBackground) > 1 {
		background := strings.Split(foregroundBackground[1], "+")
		bg = background[0]
		if len(background) > 1 {
			bgStyle = background[1]
		}
	}
	//"\033["
	buf.WriteString(start)
    //30
	base := normalIntensityFG
    //0;
	buf.WriteString(normal) // reset any previous style
	if len(fgStyle) > 0 {
		if strings.Contains(fgStyle, "b") {
			buf.WriteString(bold)
		}
		if strings.Contains(fgStyle, "d") {
			buf.WriteString(dim)
		}
		if strings.Contains(fgStyle, "B") {
			buf.WriteString(blink)
		}
		if strings.Contains(fgStyle, "u") {
			buf.WriteString(underline)
		}
		if strings.Contains(fgStyle, "i") {
			buf.WriteString(inverse)
		}
		if strings.Contains(fgStyle, "s") {
			buf.WriteString(strikethrough)
		}
		if strings.Contains(fgStyle, "h") {
			base = highIntensityFG
		}
	}

	// if 256-color
	n, err := strconv.Atoi(fgKey)
	if err == nil {
		fmt.Fprintf(buf, "38;5;%d;", n)
	} else {
		fmt.Fprintf(buf, "%d;", base+fg)
	}

	base = normalIntensityBG
	if len(bg) > 0 {
		if strings.Contains(bgStyle, "h") {
			base = highIntensityBG
		}
		// if 256-color
		n, err := strconv.Atoi(bg)
		if err == nil {
			fmt.Fprintf(buf, "48;5;%d;", n)
		} else {
			fmt.Fprintf(buf, "%d;", base+Colors[bg])
		}
	}

	// remove last ";"
	buf.Truncate(buf.Len() - 1)
	buf.WriteRune('m')
	return buf
}

```