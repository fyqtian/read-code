### Terraform

https://www.terraform.io/docs/index.html



### 中文教程

https://lonegunmanb.github.io/introduction-terraform

https://shazi7804.github.io/terraform-manage-guide/aws-use-case/web-instance.html

### tx教程

https://cloud.tencent.com/developer/inventory/2539



### 安装

https://learn.hashicorp.com/tutorials/terraform/install-cli?in=terraform/aws-get-started

```
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform
```



### 配置文件

https://www.terraform.io/cli/config/config-file

```bash
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"
```





### import已有的

https://spacelift.io/blog/importing-exisiting-infrastructure-into-terraform



### 多个provider

```
terraform {
  required_version = ">=0.13.5"
  required_providers {
    ucloudbj  = {
      source  = "ucloud/ucloud"
      version = ">=1.24.1"
    }
    ucloudsh  = {
      source  = "ucloud/ucloud"
      version = ">=1.24.1"
    }
  }
}

provider "ucloudbj" {
  public_key  = "your_public_key"
  private_key = "your_private_key"
  project_id  = "your_project_id"
  region      = "cn-bj2"
}

provider "ucloudsh" {
  public_key  = "your_public_key"
  private_key = "your_private_key"
  project_id  = "your_project_id"
  region      = "cn-sh2"
}

data "ucloud_security_groups" "default" {
  provider = ucloudbj
  type     = "recommend_web"
}

data "ucloud_images" "default" {
  provider          = ucloudsh
  availability_zone = "cn-sh2-01"
  name_regex        = "^CentOS 6.5 64"
  image_type        = "base"
}
```

