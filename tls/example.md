```go
package main

import (
   "crypto/x509"
   "encoding/pem"
   "io/ioutil"
   "log"
)

func main() {
   decodeDer()
   decodePem()
}


func decodeDer() {
	data, _ := ioutil.ReadFile("D:\\gocode\\test\\tls\\a.der")
	rs, err := x509.ParseCertificate(data)
	if err != nil {
		log.Fatal(err)
	}
	log.Println(rs.DNSNames)
	log.Println(rs.SerialNumber.Text(16))
	log.Println(rs.Signature)
	log.Println(rs.Subject)
	log.Println(rs.PublicKey.(*rsa.PublicKey).N.Text(16))
	log.Println(rs.IssuingCertificateURL)
}
func decodePem() {
    //base64 
   data, _ := ioutil.ReadFile("D:\\gocode\\test\\tls\\b.pem")
   rs, _ := pem.Decode(data)
   cert, err := x509.ParseCertificate(rs.Bytes)
   if err != nil {
      log.Fatal(err)
   }
   log.Println(cert.DNSNames)
}
```



```go
var pemStart = []byte("\n-----BEGIN ")
var pemEnd = []byte("\n-----END ")
var pemEndOfLine = []byte("-----")


//Decode将会找到PEM格式得区块（证书，私钥等）它将会返回区块和剩下得输入，如果没有pem数据没找到，返回p=nil和rest=data
func Decode(data []byte) (p *Block, rest []byte) {
   rest = data
    //判断第一行是不是pemStart开头
   if bytes.HasPrefix(data, pemStart[1:]) {
      rest = rest[len(pemStart)-1 : len(data)]
   } else if i := bytes.Index(data, pemStart); i >= 0 {
      rest = rest[i+len(pemStart) : len(data)]
   } else {
      return nil, data
   }
	//第一行内容，剩下得内容
   typeLine, rest := getLine(rest)
    //判断第一行后续内容
   if !bytes.HasSuffix(typeLine, pemEndOfLine) {
      return decodeError(data, rest)
   }
   typeLine = typeLine[0 : len(typeLine)-len(pemEndOfLine)]

   p = &Block{
      Headers: make(map[string]string),
      Type:    string(typeLine),
   }

   for {
      // This loop terminates because getLine's second result is
      // always smaller than its argument.
      if len(rest) == 0 {
         return nil, data
      }
       //正文第一行
      line, next := getLine(rest)
       //如果包含:跳出
      i := bytes.IndexByte(line, ':')
      if i == -1 {
         break
      }

      // TODO(agl): need to cope with values that spread across lines.
      key, val := line[:i], line[i+1:]
      key = bytes.TrimSpace(key)
      val = bytes.TrimSpace(val)
      p.Headers[string(key)] = string(val)
      rest = next
   }

   var endIndex, endTrailerIndex int

 //找到结尾
   if len(p.Headers) == 0 && bytes.HasPrefix(rest, pemEnd[1:]) {
      endIndex = 0
      endTrailerIndex = len(pemEnd) - 1
   } else {
      endIndex = bytes.Index(rest, pemEnd)
      endTrailerIndex = endIndex + len(pemEnd)
   }

   if endIndex < 0 {
      return decodeError(data, rest)
   }


    //剩下得证书内容
   endTrailer := rest[endTrailerIndex:]
   endTrailerLen := len(typeLine) + len(pemEndOfLine)
   if len(endTrailer) < endTrailerLen {
      return decodeError(data, rest)
   }

   restOfEndLine := endTrailer[endTrailerLen:]
   endTrailer = endTrailer[:endTrailerLen]
   if !bytes.HasPrefix(endTrailer, typeLine) ||
      !bytes.HasSuffix(endTrailer, pemEndOfLine) {
      return decodeError(data, rest)
   }

   // The line must end with only whitespace.
   if s, _ := getLine(restOfEndLine); len(s) != 0 {
      return decodeError(data, rest)
   }
	//base64 decode
   base64Data := removeSpacesAndTabs(rest[:endIndex])
   p.Bytes = make([]byte, base64.StdEncoding.DecodedLen(len(base64Data)))
   n, err := base64.StdEncoding.Decode(p.Bytes, base64Data)
   if err != nil {
      return decodeError(data, rest)
   }
   p.Bytes = p.Bytes[:n]
   

   // the -1 is because we might have only matched pemEnd without the
   // leading newline if the PEM block was empty.
   _, rest = getLine(rest[endIndex+len(pemEnd)-1:])

   return
}
```