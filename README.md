# Terraform을 사용한 AWS 인프라 자동화

## 개요

이 프로젝트는 Terraform을 사용하여 AWS 인프라를 자동으로 구축하는 예제입니다. 이 예제에서는 AWS의 S3 버킷과 EC2 인스턴스를 설정하여, 정적 웹사이트 호스팅과 인스턴스 배포를 자동화하는 방법을 다룹니다. 모든 리소스는 Terraform 코드로 선언되어 있으며, 쉽게 재사용 가능하고 인프라를 코드로 관리(IaC)할 수 있도록 설계되었습니다.

## 주요 기능

- **S3 버킷 생성 및 정적 웹사이트 호스팅**
- **EC2 인스턴스 생성**
- **보안 그룹 설정**
- **인프라 상태 출력(Output) 설정**

이 README 파일은 이 코드를 클론하고 실행하는 방법을 단계별로 안내하며, 각 구성 요소의 역할을 명확하게 설명합니다.

## 사전 준비 사항

Terraform을 사용하여 이 프로젝트를 배포하기 전에, 다음의 요구 사항을 충족해야 합니다:

- **Terraform**: 최신 버전의 Terraform을 설치하세요. [Terraform 설치하기](https://www.terraform.io/downloads)
- **AWS CLI**: AWS 자격 증명을 설정하기 위해 AWS CLI를 설치 및 구성하세요. [AWS CLI 설치하기](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- **AWS 자격 증명**: AWS 계정에 적절한 권한이 있는 자격 증명이 필요합니다. Terraform이 리소스를 생성할 수 있도록 자격 증명을 설정합니다.
- **SSH 키 페어**: EC2 인스턴스에 접근할 수 있도록 키 페어를 생성합니다.

## 사용 방법

### 1. 프로젝트 클론

```
apt-get update && apt-get install terraform -y
```

### 2. Terraform 초기화 및 실행

1. **Terraform 초기화**
Terraform 모듈과 플러그인을 다운로드하고 초기화합니다.
    
    ```
    terraform init
    ```
    
2. **Terraform 플랜 확인**
생성될 인프라를 미리 검토합니다.
    
    ```
    terraform plan
    ```
    
3. **Terraform 적용**
인프라를 실제로 배포합니다. 이 과정에서 사용자에게 확인을 요청할 수 있습니다.
    
    ```
    terraform apply
    ```
    
4. **출력 확인**
배포가 완료되면 웹사이트 엔드포인트와 EC2 인스턴스 정보가 출력됩니다.

## 리소스 설명

### 1. S3 버킷 생성

```
resource "aws_s3_bucket" "bucket1" {
  bucket = "{버킷이름}"
}
```

이 블록은 **S3 버킷**을 생성합니다. 여기서 버킷 이름은 `ce20`입니다.

### 2. S3 퍼블릭 액세스 설정

```
resource "aws_s3_bucket_public_access_block" "bucket1_public_access_block" {
  bucket = aws_s3_bucket.bucket1.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = true
  restrict_public_buckets = false
}
```

퍼블릭 액세스 설정은 S3 버킷에 대한 접근을 제어합니다. 이 설정에서는 퍼블릭 정책과 ACL을 허용하도록 구성했습니다.

### 3. S3 웹사이트 호스팅 설정

```
resource "aws_s3_bucket_website_configuration" "xweb_bucket_website" {
  bucket = aws_s3_bucket.bucket1.id
  index_document {
    suffix = "index.html"
  }
}
```

이 블록은 S3 버킷을 **정적 웹사이트로 사용**하기 위한 구성을 정의합니다. `index.html`이 메인 페이지가 됩니다.

### 4. 웹사이트 파일 업로드

```
resource "aws_s3_object" "index" {
  bucket       = aws_s3_bucket.bucket1.id
  key          = "index.html"
  source       = "index.html"
  content_type = "text/html"
}
```
![image](https://github.com/user-attachments/assets/b9dd3b56-8468-4b35-acd3-53e762973c27)<br>
![image](https://github.com/user-attachments/assets/385c5484-d7c1-4ccc-86b7-bb50aef98ce2)

로컬에서 `index.html` 파일을 S3 버킷에 업로드하여 웹사이트가 정상 작동하도록 합니다.

### 5. S3 퍼블릭 읽기 정책 설정

```
resource "aws_s3_bucket_policy" "public_read_access" {
  bucket = aws_s3_bucket.bucket1.id

  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": [ "s3:GetObject" ],
      "Resource": [
        "arn:aws:s3:::{버킷이름}/*"
      ]
    }
  ]
}
EOF
}
```

이 정책은 S3 버킷의 객체를 **모든 사용자**가 읽을 수 있도록 허용하여, 정적 웹사이트 파일에 퍼블릭 액세스를 제공합니다.

### 6. EC2 인스턴스 생성

```
resource "aws_instance" "ce20" {
  ami           = "ami-02c329a4b4aba6a48"
  instance_type = "t2.micro"
  key_name      = "ce20-key"
  tags = {
    Name = "{인스턴스 이름}"
  }
}
```

EC2 인스턴스를 생성하며, AMI ID와 인스턴스 타입, 키 페어 이름을 지정합니다.

### 7. 보안 그룹 설정

```
resource "aws_security_group" "example_sg" {
  name_prefix = "terraform-example-sg"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

이 보안 그룹은 EC2 인스턴스에 대한 **SSH 접근(포트 22)**을 허용합니다.

### 8. Output 설정

```
output "website_endpoint" {
  value       = aws_s3_bucket.bucket1.website_endpoint
  description = "The endpoint for the S3 bucket website."
}

output "main_html_url" {
  value = "https://${aws_s3_bucket_object.index.bucket}.s3.amazonaws.com/${aws_s3_bucket_object.index.key}"
}
```

Terraform 배포가 완료된 후 **웹사이트 엔드포인트**와 파일 URL을 출력하여 사용자가 쉽게 접근할 수 있도록 합니다.

