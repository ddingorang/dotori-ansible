# 도토리 - EC2 배포 Ansible

Ubuntu 기반 EC2 인스턴스 하나에 Docker를 설치하고, PostgreSQL·Redis·RabbitMQ·Spring Boot
백엔드를 하나의 docker-compose 스택으로 배포하며, Nginx를 리버스 프록시로 앞단에 두는
Ansible 플레이북입니다. Jenkins도 같은 인스턴스에 설치해 CI/CD 파이프라인(빌드 → ECR
push → Ansible 배포)을 구성할 수 있습니다.

## 아키텍처

```
Internet
  │
  ▼
Nginx (80/443, 리버스 프록시 + Let's Encrypt HTTPS)
  │
  ├── /            → Spring Boot 앱 컨테이너 (8080)
  │                     ├── PostgreSQL 컨테이너 (5432)
  │                     ├── Redis 컨테이너 (6379)
  │                     └── RabbitMQ 컨테이너 (5672 / 관리 UI 15672)
  │
  └── /finance/    → Finance Sandbox 컨테이너 (8090) - finance-sandbox jar를 서버에서 로컬 빌드

별도: Jenkins (9090) - 앱 이미지를 빌드해서 ECR에 push
```

앱 이미지는 Jenkins가 빌드해서 ECR에 push한 것을 그대로 pull해서 실행합니다(jar를 서버로
직접 복사하는 방식은 사용하지 않습니다). 다만 Finance Sandbox는 별도 CI/ECR 파이프라인이
없으므로, 저장소 루트에 준비해둔 jar 파일을 Ansible이 서버로 복사한 뒤 그 자리에서
`docker compose build`로 이미지를 빌드해 컨테이너로 띄웁니다(`files/finance-sandbox/Dockerfile`).
PostgreSQL/Redis/RabbitMQ 데이터는 Docker 볼륨에 저장되어 컨테이너를 재기동해도 유지됩니다.

## 사전 준비

1. **EC2 인스턴스**: Ubuntu 22.04 이상
   - 인바운드 보안그룹: 22번(SSH), 80번(HTTP), 9090번(Jenkins), 15672번(RabbitMQ 관리 UI) 오픈
   - HTTPS를 쓸 경우 443번도 추가로 오픈
   - 8080(앱), 5432(PostgreSQL), 6379(Redis), 5672(RabbitMQ AMQP)는 컨테이너 간 내부
     통신에만 쓰이므로 외부 오픈은 선택 사항입니다
2. **로컬/CI 환경**: Ansible 설치 (`pip install ansible` 또는 `brew install ansible`)
3. **SSH 키**: EC2 접속용 pem 키 파일 준비 (저장소에는 포함되어 있지 않습니다 — 직접 준비)
4. **Finance Sandbox jar**: 저장소 루트에 `finance-sandbox-0.0.1-SNAPSHOT.jar` 파일을 준비
   (`.gitignore`의 `*.jar` 규칙 때문에 커밋되지 않으므로, 배포 전마다 최신 jar를 직접 이
   위치에 두어야 합니다)

```bash
git clone <이 저장소>
cd dotori-ansible

# 필요한 Ansible 컬렉션 설치
ansible-galaxy collection install -r requirements.yml
```

## 설정 파일 준비

이 저장소는 비밀번호/도메인/서버 IP 같은 환경별 값을 커밋하지 않습니다. `.example`로 끝나는
템플릿 파일을 복사해서 실제 값을 채운 뒤 사용하세요 (실제 파일들은 `.gitignore`에 등록되어
있어 실수로 커밋되지 않습니다).

| 템플릿 | 실제로 만들 파일 | 용도 |
|---|---|---|
| `inventory/hosts.ini.example` | `inventory/hosts.ini` | EC2 접속 정보 |
| `group_vars/all.yml.example` | `group_vars/all.yml` | 배포 설정값 |
| `group_vars/vault.yml.example` | `group_vars/vault.yml` | 비밀번호/키 (암호화 필수) |

### 1. 인벤토리 설정

```bash
cp inventory/hosts.ini.example inventory/hosts.ini
```

`inventory/hosts.ini`를 열어 EC2 퍼블릭 IP와 키 경로를 실제 값으로 수정합니다.

```ini
[dotori_server]
ec2-target ansible_host=13.xxx.xxx.xxx

[dotori_server:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/your-ec2-key.pem
```

### 2. 배포 설정값 준비

```bash
cp group_vars/all.yml.example group_vars/all.yml
```

`group_vars/all.yml`을 열어 최소한 아래 값들을 환경에 맞게 채웁니다.

- `aws_account_id`, `aws_region` — ECR 이미지가 있는 AWS 계정/리전
- `domain_name`, `certbot_email` — HTTPS를 쓸 경우 (선택, 아래 "HTTPS" 절 참고)

### 3. 민감정보(Vault) 설정

비밀번호를 평문으로 두지 않도록 `ansible-vault`로 암호화합니다.

```bash
cp group_vars/vault.yml.example group_vars/vault.yml
# vault.yml 파일을 열어 실제 비밀번호로 수정한 뒤
ansible-vault encrypt group_vars/vault.yml
```

채워야 할 값:

```yaml
vault_postgres_password: ""
vault_redis_password: ""
vault_rabbitmq_password: ""
vault_jwt_secret: ""
vault_finance_api_key: ""

# ECR에서 dotori-backend 이미지를 pull할 때 사용하는 IAM 자격증명
vault_aws_access_key_id: ""
vault_aws_secret_access_key: ""
```

`group_vars/all.yml`은 이 `vault_*` 변수들을 참조하도록 이미 작성되어 있습니다.

## HTTPS (Let's Encrypt) 설정 (선택)

도메인이 있다면 `group_vars/all.yml`의 `domain_name`, `certbot_email`을 채워서 HTTPS를
자동으로 적용할 수 있습니다.

```yaml
domain_name: dotori.example.com   # 이 서버의 퍼블릭 IP를 가리키는 A 레코드가 미리 설정되어 있어야 함
certbot_email: you@example.com    # Let's Encrypt 만료 알림용 이메일
```

`domain_name`을 채우면 `enable_https`가 자동으로 `true`가 되어, 플레이북이 다음을 수행합니다.

1. `certbot` 설치, ACME 챌린지용 webroot(`/var/www/certbot`) 준비
2. Nginx를 우선 HTTP(80)만으로 배포 (ACME 챌린지 + `https://`로 리다이렉트)
3. 인증서가 없으면 `certbot certonly --webroot`로 최초 발급
4. 발급된 인증서를 포함해 Nginx 설정을 재배포 → 443에서 HTTPS 서비스
5. 인증서 갱신(`certbot.timer`가 자동 실행) 시 Nginx가 자동으로 reload되도록 훅 등록
6. 보안그룹/UFW에 443 포트 오픈

`domain_name`을 비워두면(기본값) 위 과정은 전부 건너뛰고 기존처럼 80번 HTTP만 사용합니다.

> EC2 보안그룹에서도 443번 포트 인바운드를 미리 열어두세요(UFW는 플레이북이 자동으로 엽니다).

## 배포 실행

```bash
ansible-playbook playbook.yml --ask-vault-pass
```

- `--ask-vault-pass`: `vault.yml` 암호를 입력받아 복호화
- 최초 실행 시 Docker/Jenkins 설치까지 포함되어 몇 분 정도 소요될 수 있습니다.
- 이미 배포된 상태에서 다시 실행하면(코드 변경, 이미지 새 버전 등) 변경된 부분만 반영되고
  컨테이너가 재기동됩니다.

## 배포 확인

```bash
ssh -i ~/.ssh/your-ec2-key.pem ubuntu@<EC2_IP>
docker compose -f /opt/dotori/docker-compose.yml ps
docker compose -f /opt/dotori/docker-compose.yml logs -f app
sudo nginx -t
sudo systemctl status nginx
```

애플리케이션은 `http://<EC2_IP>` (또는 HTTPS 설정 시 `https://<도메인>`)로 접근 가능합니다.
(8080번 포트를 보안그룹에서 열어둔 경우 `http://<EC2_IP>:8080`으로 직접 접근도 가능합니다.)

## 디렉토리 구조

```
dotori-ansible/
├── ansible.cfg
├── requirements.yml
├── playbook.yml
├── .gitignore
├── inventory/
│   └── hosts.ini.example      # cp → hosts.ini 로 실제 값 채우기
├── group_vars/
│   ├── all.yml.example        # cp → all.yml 로 실제 값 채우기
│   └── vault.yml.example      # cp → vault.yml 로 실제 값 채우고 ansible-vault encrypt
├── files/
│   └── finance-sandbox/
│       └── Dockerfile         # finance-sandbox jar를 서버에서 빌드하기 위한 Dockerfile
├── finance-sandbox-0.0.1-SNAPSHOT.jar   # 직접 준비 (커밋되지 않음, .gitignore 참고)
└── templates/
    ├── docker-compose.yml.j2
    ├── env.j2
    └── nginx-dotori.conf.j2
```

## 참고 사항

- PostgreSQL/Redis/RabbitMQ 데이터는 Docker 볼륨에 저장되어, 컨테이너를 재기동해도
  데이터가 유지됩니다. 완전히 초기화하려면 서버에서 `docker compose down -v` 실행.
- CI/CD(Jenkins, GitHub Actions 등)와 연동한다면, 이미지 빌드 → ECR push → Ansible 실행
  단계를 파이프라인에 그대로 편입시킬 수 있습니다.
- 지금 구조는 서버 1대에 앱과 DB를 함께 두는 구성입니다. 트래픽이 늘어나면 DB를 별도
  인스턴스(RDS, ElastiCache 등)로 분리하는 것을 고려하세요.
- Finance Sandbox(8090) 컨테이너는 포트를 직접 외부에 열지 않고, Nginx가 `/finance/`
  경로로 리버스 프록시합니다 (`http(s)://<도메인 또는 EC2 IP>/finance/...`). 8090 포트
  자체는 PostgreSQL/Redis/RabbitMQ와 마찬가지로 UFW에서 열려 있지 않습니다.
- `domain_name`을 설정하지 않으면 Nginx는 80번 포트로 평문 HTTP만 프록시합니다. 이 경우
  `cookie_secure`도 `false`로 바꿔주는 것을 권장합니다(기본값은 `true`이며 HTTPS 없이
  `true`로 두면 브라우저가 쿠키를 저장하지 않습니다).
- `domain_name`을 설정하면 위의 HTTPS 절 설명대로 certbot이 자동으로 인증서를 발급/갱신하고
  443/HTTPS를 적용하며, 이때는 `cookie_secure: true` 그대로 사용하면 됩니다.
- `inventory/hosts.ini`, `group_vars/all.yml`, `group_vars/vault.yml`, `*.pem`은
  `.gitignore`에 등록되어 있어 커밋되지 않습니다. 실수로 `git add -f` 등으로 강제
  추가하지 않도록 주의하세요.
