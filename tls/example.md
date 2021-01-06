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
    //ASN.1 编码
   data, _ := ioutil.ReadFile("D:\\gocode\\test\\tls\\a.der")
   rs, err := x509.ParseCertificate(data)
   if err != nil {
      log.Fatal(err)
   }
   log.Println(rs.DNSNames)
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