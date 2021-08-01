### Terraform

https://www.terraform.io/docs/index.html



### 安装

https://learn.hashicorp.com/tutorials/terraform/install-cli?in=terraform/aws-get-started

```
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform
```

