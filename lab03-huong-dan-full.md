# Lab 03 — CI/CD với Jenkins, Docker, Nginx, IaC (VMware thay EC2)

Toàn bộ EC2 trong đề gốc được thay bằng **VM VMware Workstation** (tạo qua Vagrant). Máy host Windows đã có sẵn VMware Workstation 17.5.2, Vagrant 2.4.9 (WSL), vagrant-vmware-utility đang chạy.

Repo dùng cho lab: `https://github.com/KhacThien88/CloudHCMUS_Lab01_FE`

---

## Phần A — CI: Jenkins build & push image

### A1. Tạo VM #1 (Jenkins VM) — thay cho "Tạo 1 EC2"

```bash
mkdir -p ~/WorkSpace/devops/jenkins-vm && cd ~/WorkSpace/devops/jenkins-vm
vagrant init bento/ubuntu-22.04
```

`Vagrantfile`:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-22.04"
  config.vm.hostname = "jenkins-vm"
  config.vm.network "private_network", type: "dhcp"
  config.vm.network "forwarded_port", guest: 8080, host: 8080
  config.vm.network "forwarded_port", guest: 50000, host: 50000

  config.vm.provider "vmware_desktop" do |v|
    v.vmx["memsize"] = "4096"
    v.vmx["numvcpus"] = "2"
  end
end
```

```bash
vagrant up --provider=vmware_desktop
vagrant ssh
```

### A2. Host Jenkins trên VM

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER && newgrp docker

docker volume create jenkins_home
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --restart unless-stopped \
  jenkins/jenkins:lts-jdk17
```

Mở khóa lần đầu:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Trình duyệt Windows → `http://localhost:8080` → nhập password → **Install suggested plugins** → tạo admin.

### A3. Trigger webhook (GitHub → Jenkins)

VM nằm sau NAT nên cần expose public tạm bằng ngrok:

```bash
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update && sudo apt install -y ngrok
ngrok config add-authtoken <TOKEN>
ngrok http 8080
```

GitHub repo → **Settings → Webhooks → Add webhook**:
- Payload URL: `https://xxxx.ngrok-free.app/github-webhook/`
- Content type: `application/json`
- Event: **Just the push event**

Trong Jenkins job → **Build Triggers** → tick **GitHub hook trigger for GITScm polling**.

### A4. Cài Blue Ocean, Stage View và plugin cần thiết

**Manage Jenkins → Plugins → Available** → cài:

| Plugin | Mục đích |
|---|---|
| Blue Ocean | UI trực quan pipeline |
| Pipeline: Stage View | Xem stage dạng bảng |
| GitHub Integration | Nhận webhook |
| Docker Pipeline | Lệnh `docker.build()` / `docker.withRegistry()` trong Jenkinsfile |
| Credentials Binding | Bind credential DockerHub/GitHub vào pipeline |

Restart Jenkins nếu được yêu cầu: `docker restart jenkins`

### A5. Cấu hình credentials

**Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

1. **DockerHub**: kind = "Username with password", ID = `dockerhub-cred`
2. **GitHub**: kind = "Username with password" (hoặc Personal Access Token), ID = `github-cred`

### A6. Jenkinsfile — CI (checkout, build, push image)

Tạo `Jenkinsfile` ở root repo FE (`CloudHCMUS_Lab01_FE`):

```groovy
pipeline {
    agent any

    environment {
        DOCKERHUB_CRED = credentials('dockerhub-cred')
        IMAGE_NAME = "yourdockerhubuser/lab01-fe"
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('Push') {
            steps {
                sh "echo $DOCKERHUB_CRED_PSW | docker login -u $DOCKERHUB_CRED_USR --password-stdin"
                sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                sh "docker push ${IMAGE_NAME}:latest"
            }
        }
    }

    post {
        always {
            sh "docker logout"
        }
    }
}
```

> Repo là HTML/JS cơ bản nên `Dockerfile` chỉ cần base image nginx phục vụ static file, ví dụ:
> ```dockerfile
> FROM nginx:alpine
> COPY . /usr/share/nginx/html
> EXPOSE 80
> ```

### A7. Tạo Pipeline job + test luồng CI

Jenkins → **New Item → Pipeline**
- **Pipeline script from SCM** → Git → URL repo → Branch → Script Path: `Jenkinsfile`

Test: commit + push code lên GitHub → webhook gọi Jenkins → job tự chạy → kiểm tra DockerHub có image mới (tag = build number + `latest`). ✅ Hoàn thành CI.

---

## Phần B — CD: Deploy image lên VM app + Nginx reverse proxy + domain + SSL

### B1. Tạo VM #2 (App VM) — thay cho "Tạo 1 EC2 mới"

```bash
cd ~/WorkSpace/devops
mkdir app-vm && cd app-vm
vagrant init bento/ubuntu-22.04
```

`Vagrantfile` tương tự VM Jenkins nhưng forward port cho app (ví dụ container FE chạy port 8081 trên host):

```ruby
config.vm.network "forwarded_port", guest: 80, host: 8081
```

```bash
vagrant up --provider=vmware_desktop
```

### B2. Pipeline deploy — khi commit code thì app VM tự pull & chạy image mới nhất

Cài Docker trên App VM (giống bước A2). Thêm stage **Deploy** vào Jenkinsfile (chạy sau stage Push), SSH sang App VM để pull + restart container:

```groovy
stage('Deploy') {
    steps {
        sshagent(['app-vm-ssh-cred']) {
            sh """
                ssh -o StrictHostKeyChecking=no vagrant@<APP_VM_IP> '
                    docker pull ${IMAGE_NAME}:latest &&
                    docker rm -f lab01-fe || true &&
                    docker run -d --name lab01-fe -p 80:80 ${IMAGE_NAME}:latest
                '
            """
        }
    }
}
```

- Cần plugin **SSH Agent** trong Jenkins, credential SSH key của App VM add vào Jenkins (`app-vm-ssh-cred`).
- `<APP_VM_IP>` = IP mạng private_network của App VM (`vagrant ssh -c "ip a"` để lấy).

Test: truy cập `http://<APP_VM_IP>:port` (hoặc `http://localhost:8081` nếu forward port) → thấy app FE. Commit code mới → pipeline tự deploy → refresh trang thấy version mới.

### B3. Tạo VM #3 (Nginx reverse proxy VM) — thay cho "Tạo 1 EC2 mới, tải nginx"

```bash
cd ~/WorkSpace/devops
mkdir nginx-vm && cd nginx-vm
vagrant init bento/ubuntu-22.04
```

```ruby
config.vm.network "forwarded_port", guest: 80, host: 8082
config.vm.network "forwarded_port", guest: 443, host: 8443
```

```bash
vagrant up --provider=vmware_desktop
vagrant ssh
sudo apt update && sudo apt install -y nginx
```

### B4. Cấu hình Nginx reverse proxy → App VM

`/etc/nginx/sites-available/app`:

```nginx
server {
    listen 80;
    server_name test-app-hoten.khacthienit.click;

    location / {
        proxy_pass http://<APP_VM_IP>:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### B5. Trỏ domain (A record) vào Nginx VM

Vì VM chạy local (không có public IP thật như EC2), cần 1 trong 2 cách:
- **Public IP thật**: nếu máy Windows có public IP (hoặc port-forward router ra ngoài) → dùng chính IP đó, port-forward 80/443 từ router → VM Nginx.
- **Tunnel** (khuyến nghị cho lab tại nhà): dùng `cloudflared tunnel` hoặc `ngrok` trỏ domain qua tunnel thay vì A record trực tiếp.

> Theo đề gốc: A record `test-app-hoten.khacthienit.click` → liên hệ người quản lý domain (bé trong đề) để trỏ record vào IP của bạn.

### B6. Cài Certbot, xin SSL cert

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo systemctl stop nginx     # tắt nginx trước khi xin cert (đề yêu cầu)
sudo certbot certonly --standalone -d test-app-hoten.khacthienit.click
sudo systemctl start nginx
```

### B7. Cấu hình Nginx dùng cert, reverse proxy qua HTTPS

```nginx
server {
    listen 443 ssl;
    server_name test-app-hoten.khacthienit.click;

    ssl_certificate     /etc/letsencrypt/live/test-app-hoten.khacthienit.click/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/test-app-hoten.khacthienit.click/privkey.pem;

    location / {
        proxy_pass http://<APP_VM_IP>:80;
        proxy_set_header Host $host;
    }
}

server {
    listen 80;
    server_name test-app-hoten.khacthienit.click;
    return 301 https://$host$request_uri;
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Truy cập `https://test-app-hoten.khacthienit.click` → ra app FE. ✅ Hoàn thành phần domain/SSL.

---

## Phần C — CD bằng Infrastructure as Code (Terraform + Ansible)

### C1. Tìm hiểu Terraform & Ansible (tóm tắt)

- **Terraform**: khai báo hạ tầng (VM, network...) dưới dạng code (`.tf`), `terraform apply` sẽ tạo/sửa/xoá hạ tầng đúng với state khai báo. Có provider cho AWS, và **provider `vmware/vmworkstation`** (community) cho VMware Workstation local.
- **Ansible**: công cụ configuration management, chạy playbook YAML qua SSH để cài đặt phần mềm (Docker...) và deploy ứng dụng lên VM đã có sẵn (không tạo VM, chỉ cấu hình bên trong).

### C2. Source IaC mới cho luồng CD

Cấu trúc thư mục đề xuất:

```
iac/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── ansible/
    ├── inventory.ini
    ├── playbook.yml
    └── roles/
        └── docker/
```

**terraform/main.tf** (dùng provider VMware Workstation để tạo App VM mới):

```hcl
terraform {
  required_providers {
    vmworkstation = {
      source  = "elsudano/vmworkstation"
    }
  }
}

provider "vmworkstation" {
  user     = var.vmws_user
  password = var.vmws_password
  url      = "https://127.0.0.1:8697/api"
}

resource "vmworkstation_vm" "app_vm" {
  denomination = "app-vm-cd"
  sourceid     = var.template_vm_id
  path         = var.vm_path
  processors   = 2
  memory       = 4096
}
```

> Nếu provider VMware Workstation community không ổn định trong môi trường lab, phương án thay thế đơn giản hơn: dùng Terraform chỉ để gọi `local-exec` chạy `vagrant up`, giữ đúng tinh thần "Terraform tạo hạ tầng, Ansible cấu hình" mà không phụ thuộc provider bên thứ 3.

**ansible/inventory.ini**:

```ini
[app]
<APP_VM_IP> ansible_user=vagrant ansible_ssh_private_key_file=~/.vagrant.d/insecure_private_key
```

**ansible/playbook.yml** — cài Docker, chạy image mới nhất từ CI:

```yaml
- hosts: app
  become: true
  vars:
    image_name: "yourdockerhubuser/lab01-fe:latest"
  tasks:
    - name: Install Docker
      shell: curl -fsSL https://get.docker.com | sh

    - name: Pull latest image
      docker_image:
        name: "{{ image_name }}"
        source: pull

    - name: Run container
      docker_container:
        name: lab01-fe
        image: "{{ image_name }}"
        state: started
        recreate: true
        ports:
          - "80:80"
```

### C3. Luồng pipeline CD hoàn chỉnh

Thêm stage vào Jenkinsfile (hoặc pipeline CD riêng), chạy sau khi CI push image xong:

```groovy
stage('Terraform Apply') {
    steps {
        dir('iac/terraform') {
            sh 'terraform init'
            sh 'terraform apply -auto-approve'
        }
    }
}

stage('Ansible Deploy') {
    steps {
        dir('iac/ansible') {
            sh 'ansible-playbook -i inventory.ini playbook.yml'
        }
    }
}

stage('Update Reverse Proxy') {
    steps {
        sshagent(['nginx-vm-ssh-cred']) {
            sh """
                ssh -o StrictHostKeyChecking=no vagrant@<NGINX_VM_IP> '
                    sudo sed -i "s/proxy_pass .*/proxy_pass http:\\/\\/<NEW_APP_VM_IP>:80;/" /etc/nginx/sites-available/app &&
                    sudo systemctl reload nginx
                '
            """
        }
    }
}
```

### C4. Kiểm tra kết quả

1. Push commit mới lên repo → CI build & push image DockerHub.
2. CD: Terraform tạo App VM mới → Ansible cài Docker + chạy image mới nhất.
3. Nginx VM được cập nhật `proxy_pass` trỏ về IP App VM mới.
4. Truy cập `https://test-app-hoten.khacthienit.click` → ra app, xác nhận version mới nhất đã chạy trên hạ tầng mới hoàn toàn tạo bởi IaC.

---

## Checklist tổng

- [ ] Jenkins VM chạy, cài xong Blue Ocean + Stage View + plugin cần thiết
- [ ] Jenkinsfile CI: checkout → build → push DockerHub, trigger tự động qua webhook
- [ ] Credentials DockerHub + GitHub cấu hình trong Jenkins
- [ ] App VM chạy container, pipeline CD deploy version mới khi commit
- [ ] Nginx VM reverse proxy hoạt động, domain trỏ đúng, HTTPS bằng certbot
- [ ] Source IaC (Terraform + Ansible) tạo App VM mới + cài Docker + deploy image mới nhất, cập nhật reverse proxy tự động

---

## Ghi chú khác biệt so với EC2 gốc

- **Public IP / domain**: EC2 có public IP thật sẵn; VM VMware chạy local nên cần port-forward router hoặc tunnel (ngrok/cloudflared) để expose ra internet thật cho webhook & domain.
- **Terraform provider**: AWS có provider chính thức mạnh; VMware Workstation chỉ có provider cộng đồng (`elsudano/vmworkstation`), kém ổn định hơn — nếu gặp lỗi, dùng phương án `local-exec` gọi Vagrant như ghi chú ở Phần C2.
- **Network giữa các VM**: dùng `private_network` của Vagrant để 3 VM (Jenkins, App, Nginx) thấy nhau qua IP nội bộ, tương tự VPC/Security Group trong AWS.
