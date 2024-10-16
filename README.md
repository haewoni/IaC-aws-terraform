# <p align="center"> IaC-AWS-Terraform

### Terraform 을 사용하여 AWS S3 버킷과 EC2 인스턴스 생성하기

## 사용기술
- Terraform
- AWS S3, EC2
<br>

## 프로젝트 목적 🌷
인프라 자동화(IaC)를 통해 클라우드 자원을 효율적으로 관리하고, 반복 가능한 인프라 배포 프로세스 구축

<br>

## 실습 개요 :star:

- step 01 : AWS CLI 사용하여 Terrafrom 설치하기
- step 02 : AWS S3 버킷 생성하기
- step 03 : S3에 html 파일 업로드하고 endpoint로 확인하기
- step 04 : EC2 생성하기

<br>

## 실습 과정 :mag_right:

### ☑️ step 01 : AWS CLI 사용하여 Terrafrom 설치하기
```
# root 계정접속
$ sudo su -

# wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg


# echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# apt-get update && apt-get install terraform -y

# terraform -version
```

### ☑️ step 02 : AWS S3 버킷 생성하기

#### 2-1. 권한 부여 설정
```
# IAM 역할 생성
resource "aws_iam_role" "s3_create_bucket_role" {
  name = "[IAM Role Name]" 
  
  assume_role_policy = jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Action": "sts:AssumeRole",
        "Effect": "Allow",
        "Principal": {
          "Service": "ec2.amazonaws.com"
        }
      }
    ]
  })
}

# IAM 정책 정의 (S3에 대한 모든 권한 부여)
resource "aws_iam_policy" "s3_full_access_policy" {
  name        = "[Policy Name]"  
  description = "Full access to S3 resources"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:*"  # 모든 S3 액세스 허용
        ]
        Resource = [
          "*"  # 모든 S3 리소스에 대한 권한
        ]
      }
    ]
  })
}

# IAM 역할에 정책 연결
resource "aws_iam_role_policy_attachment" "attach_s3_policy" {
  role       = aws_iam_role.s3_create_bucket_role.name
  policy_arn = aws_iam_policy.s3_full_access_policy.arn
}

```

#### 2-2. 버킷 생성
```
# S3 버킷 생성
resource "aws_s3_bucket" [bucket명] {
  bucket = [bucket명]  # 생성하고자 하는 S3 버킷 이름
}

# S3 버킷의 public access block 설정
resource "aws_s3_bucket_public_access_block" [access block 명] {
  bucket = aws_s3_bucket.bucket2.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}
```

#### 2-3. index.html 업로드

#### 2-4. apply 하기

```
terraform init
terraform apply
```



### ☑️ step 03 : S3에 html 파일 업로드하고 endpoint로 확인하기

#### 3-1. index.html 업로드

#### 3-2. index.html 수정

- 내용 수정 후 apply
- 
```
# 이미 존재하는 S3 버킷에 index.html 파일을 업로드
resource "aws_s3_object" "index_updated" {
  bucket        = "[S3 Bucket Name]"  
  key           = "index2.html"
  source        = "index2.html"
  content_type  = "text/html"

  # Optional
  etag   = filemd5("index2.html")  # 파일 변경 시 etag도 갱신
}

# S3 버킷의 웹사이트 호스팅 설정
resource "aws_s3_bucket_website_configuration" "xweb_bucket_website_updated" {
  bucket = "[S3 Bucket Name]" 

  index_document {
    suffix = "index2.html"
  }
}

```



### ☑️ step 04 : EC2 생성하기

```
# AWS 제공자 설정
provider "aws" {
  region = "ap-northeast-2"  # 서울 리전
}

# VPC 생성
resource "aws_vpc" [vpc명] {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = [vpc명]
  }
}

# 서브넷 생성
resource "aws_subnet" [subnet명] {
  vpc_id            = aws_vpc.[vpc명].id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "ap-northeast-2a"  # 서울 리전의 가용영역
  tags = {
    Name = [subnet명]
  }
}

# 인터넷 게이트웨이 생성
resource "aws_internet_gateway" [igw명] {
  vpc_id = aws_vpc.[vpc명].id
  tags = {
    Name = [igw명]
  }
}

# 라우팅 테이블 생성
resource "aws_route_table" [routing table명] {
  vpc_id = aws_vpc.[vpc명].id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.[igw명] .id
  }
  tags = {
    Name = [routing table명] 
  }
}

# 라우팅 테이블과 서브넷 연결
resource "aws_route_table_association" "route_table_assoc" {
  subnet_id      = aws_subnet.[subnet명] .id
  route_table_id = aws_route_table.[routing table명] .id
}

# 보안 그룹 생성 (SSH 허용)
resource "aws_security_group" [security group명]  {
  vpc_id = aws_vpc.[vpc명] .id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # SSH 접속 허용
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]  # 모든 아웃바운드 트래픽 허용
  }

  tags = {
    Name = "[security group명] "
  }
}

# EC2 인스턴스 생성
resource "aws_instance" [ec2명] {
  ami           = "ami-0a10b2721688ce9d2"  # Amazon Linux 2 AMI (서울 리전)
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.[subnet명].id  # 수동으로 생성한 서브넷 사용
  vpc_security_group_ids = [aws_security_group.security_group.id]  # 보안 그룹 ID 사용

  key_name = [키페어id]  # 사전에 생성한 키 페어

  tags = {
    Name = "ce17-ec2"
  }
}

```


## 트러블슈팅 📝
- 기존 생성된 html 파일 변화 감지 안되는 현상
```
# resource Optional tag 추가 
resource "aws_s3_object" [object name] {
  bucket        = aws_s3_bucket.[bucket name].id  # 생성된 S3 버킷 이름 사용
  key           = "index.html"
  source        = "index.html"
  content_type  = "text/html"

  # Optional
  etag   = filemd5("index.html")  # 파일 변경 시 etag도 갱신
}
```

- EC2 설정 누락 -> VPC, 서브넷, 게이트웨이, 라우팅 테이블 리소스 생성 코드 추가  

- 리소스 이름 중복 시 에러 발생 -> 모든 리소스 이름은 고유하게 설정 후 적용

```
Error: Duplicate resource "aws_s3_object" configuration
│ 
│   on uploadfile.tf line 2:
│    2: resource "aws_s3_object" "index" {
│ 
│ A aws_s3_object resource named "index" was already declared at resource.tf:19,1-33. Resource names must
│ be unique per type in each module.
```
- aws 콘솔창에 확인 입력 없이 명령어만으로 접속 url 확인

```
output "website_endpoint" {
  value = aws_s3_bucket.bucket1.website_endpoint
  description = "The endpoint for the S3 bucket website."
}
```

<br>

