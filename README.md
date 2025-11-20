# Домашнее задание к занятию `«Отказоустойчивость в облаке»` - `Евгений Головенко`

---

### Чеклист готовности к домашнему заданию

1. Создан аккаунт на AWS - [ V ] 
2. На хосте с Win10+VirtualBox+Ubuntu установлено программное обеспечение Terraform - [ V ]

### Задание 1

Возьмите за основу [решение к заданию 1 из занятия «Подъём инфраструктуры в облаке»](https://github.com/netology-code/sdvps-homeworks/blob/main/7-03.md#задание-1).

1. Теперь вместо одной виртуальной машины сделайте terraform playbook, который:

- создаст 2 идентичные виртуальные машины. Используйте аргумент [count](https://www.terraform.io/docs/language/meta-arguments/count.html) для создания таких ресурсов;
- создаст таргет-группу. Поместите в неё созданные на шаге 1 виртуальные машины;
- создаст [сетевой балансировщик нагрузки](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb), который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.

2. Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.

3. Перейдите в веб-консоль AWS и убедитесь, что: 

- созданный балансировщик находится в статусе Active,
- обе виртуальные машины в целевой группе находятся в состоянии healthy.

4. Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.

*В качестве результата пришлите:*

*1. Terraform Playbook.*

*2. Скриншот статуса балансировщика и целевой группы.*

*3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.*

---

### Решение 1

*1. Terraform Playbook.*

```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "eu-central-1"
}

# Preparing network (Default VPC)

# Get default VPC info
data "aws_vpc" "default" {
  default = true
}

# Get subnetwork list for this VPC
data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

# Get image Amazon Linux
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}

# Creating SSH 
resource "tls_private_key" "pk" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "kp" {
  key_name   = "my-key"       # Имя ключа в AWS
  public_key = tls_private_key.pk.public_key_openssh
}

resource "local_file" "ssh_key" {
  filename = "${path.module}/my-key.pem"
  content  = tls_private_key.pk.private_key_pem
  file_permission = "0400"
}

# Security Group
resource "aws_security_group" "web_sg" {
  name        = "allow_http"
  description = "Allow HTTP inbound traffic"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    description = "HTTP from VPC"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow incoming trafic
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Creating 2 VM (EC2)
resource "aws_instance" "web_server" {
  count         = 2 
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  vpc_security_group_ids = [aws_security_group.web_sg.id]

  key_name = aws_key_pair.kp.key_name 
 
# User Data script setup nginx for checking port 80 (Health Check)
    user_data = <<-EOF
              #!/bin/bash
              dnf update -y
              dnf install -y nginx
              systemctl start nginx
              systemctl enable nginx
              EOF

  tags = {
    Name = "WebServer-${count.index + 1}"
  }
}

# Creating target-group and addition hosts

resource "aws_lb_target_group" "nlb_tg" {
  name     = "tf-example-nlb-tg"
  port     = 80
  protocol = "TCP" 
  vpc_id   = data.aws_vpc.default.id
  
    health_check {
    enabled  = true
    protocol = "HTTP"
    port     = "traffic-port"
    path     = "/"
  }
}

# Attaching of created instances to target-group
resource "aws_lb_target_group_attachment" "tg_attachment" {
  count            = 2
  target_group_arn = aws_lb_target_group.nlb_tg.arn
  target_id        = aws_instance.web_server[count.index].id
  port             = 80
}

# Network Load Balancer (NLB)

resource "aws_lb" "network_lb" {
  name               = "tf-example-nlb"
  internal           = false
  load_balancer_type = "network"
  subnets            = data.aws_subnets.default.ids
}

# Listener for NLB
resource "aws_lb_listener" "front_end" {
  load_balancer_arn = aws_lb.network_lb.arn
  port              = "80"
  protocol          = "TCP" # NLB works on the L4 (TCP/UDP)

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.nlb_tg.arn
  }
}

# Useful output
output "nlb_dns_name" {
  value       = aws_lb.network_lb.dns_name
  description = "DNS name NLB. Open it in your browser."
}
```

*2. Скриншот статуса балансировщика и целевой группы.*

<img width="1904" height="921" alt="Capture2" src="https://github.com/user-attachments/assets/76ddc0ee-863b-4899-84bc-330f412f02d2" />

<img width="1909" height="919" alt="Capture3" src="https://github.com/user-attachments/assets/4fd50fc8-e6de-4456-977d-32529c3209f1" />

<img width="1908" height="919" alt="Capture4" src="https://github.com/user-attachments/assets/401df52e-0ee2-4740-b7bc-353ede553741" />

*3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.*

<img width="869" height="718" alt="Capture1" src="https://github.com/user-attachments/assets/c80f1356-ed45-46d7-bcfd-8a5b6f7a8ef4" />

<img width="991" height="357" alt="Capture1-1" src="https://github.com/user-attachments/assets/27ea1d11-3b42-4b52-87d0-ceb0ce973fe0" />



