# INFRASTRUCTURE SETUP - DOCKER, NGINX, VPS

## 1. Mục tiêu triển khai

Kiến trúc production tối thiểu:

```text
Internet :443/:80
        |
      Nginx TLS
        |
  ASP.NET API + SignalR
    |        |        |
 MySQL   RabbitMQ  Worker
```

Chỉ Nginx mở public. MySQL, RabbitMQ và internal API nằm trong private Docker network.

## 2. Thành phần Docker Compose

Các service đề nghị:

- `api`: ASP.NET Core 8 Web API.
- `worker`: publisher/consumer outbox RabbitMQ.
- `mysql`: MySQL 8 volume persistent.
- `rabbitmq`: RabbitMQ management chỉ bind localhost hoặc private admin network.
- `nginx`: reverse proxy và TLS.
- `backup`: cronjob/container hoặc host cron chạy backup.
- `dozzle`: chỉ mở qua VPN/basic auth, không public mặc định.

Không build production bằng image `latest`; pin version và publish digest khi có thể.

## 3. Dockerfile API mẫu

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
EXPOSE 8080
USER $APP_UID
COPY publish/ .
ENTRYPOINT ["dotnet", "Pos.Api.dll"]
```

Build ngoài pipeline:

```powershell
dotnet restore
dotnet test --configuration Release
 dotnet publish -c Release -o publish --no-restore
 docker build -t registry.example.com/pos-api:<git-sha> .
 docker push registry.example.com/pos-api:<git-sha>
```

Không nhúng connection string/secret vào image.

## 4. Compose production skeleton

```yaml
services:
  api:
    image: registry.example.com/pos-api:${IMAGE_TAG}
    restart: unless-stopped
    environment:
      ASPNETCORE_URLS: http://+:8080
      ConnectionStrings__Default: ${MYSQL_CONNECTION_STRING}
      RabbitMq__Host: rabbitmq
    depends_on:
      mysql: { condition: service_healthy }
      rabbitmq: { condition: service_healthy }
    networks: [backend]

  worker:
    image: registry.example.com/pos-worker:${IMAGE_TAG}
    restart: unless-stopped
    environment:
      ConnectionStrings__Default: ${MYSQL_CONNECTION_STRING}
      RabbitMq__Host: rabbitmq
    networks: [backend]

  mysql:
    image: mysql:8.0
    restart: unless-stopped
    command: --default-authentication-plugin=caching_sha2_password
    environment:
      MYSQL_DATABASE: pos
      MYSQL_USER: pos_app
      MYSQL_PASSWORD_FILE: /run/secrets/mysql_password
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root_password
    volumes: [mysql_data:/var/lib/mysql]
    networks: [backend]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]

  rabbitmq:
    image: rabbitmq:3.13-management
    restart: unless-stopped
    volumes: [rabbitmq_data:/var/lib/rabbitmq]
    networks: [backend]

  nginx:
    image: nginx:1.27-alpine
    restart: unless-stopped
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certbot/www:/var/www/certbot:ro
      - ./certbot/conf:/etc/letsencrypt:ro
    depends_on: [api]
    networks: [edge, backend]

networks:
  edge: {}
  backend:
    internal: true

volumes:
  mysql_data:
  rabbitmq_data:
```

Tên secret và image phải đổi theo môi trường thật. Không commit file `.env` production.

## 5. Nginx reverse proxy

```nginx
server {
    listen 80;
    server_name api.example.com;
    location /.well-known/acme-challenge/ { root /var/www/certbot; }
    location / { return 301 https://$host$request_uri; }
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    client_max_body_size 2m;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options DENY always;
    add_header Referrer-Policy no-referrer always;

    location / {
        proxy_pass http://api:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }

    location /hubs/ {
        proxy_pass http://api:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 120s;
    }
}
```

Cần cấu hình `ForwardedHeaders` trong ASP.NET để app biết request gốc là HTTPS. Rate limit và CORS nên cấu hình thêm ở API/Nginx.

## 6. VPS hardening

1. Ubuntu LTS cập nhật tự động security patches.
2. SSH key-only, tắt password login và root login.
3. Firewall chỉ mở `22` từ IP quản trị, `80/443` public.
4. Không mở `3306`, `5672`, `15672`, `8080` ra Internet.
5. Docker daemon không public TCP socket.
6. Tạo user deploy không dùng root; quyền thư mục secret tối thiểu.
7. Bật disk monitoring, memory/swap alert và log rotation.
8. Dùng fail2ban hoặc security group cho SSH.
9. Đồng bộ NTP để JWT, audit và event timestamp chính xác.

## 7. MySQL production

- Volume persistent trên disk mã hóa nếu nhà cung cấp hỗ trợ.
- User app chỉ có quyền database cần thiết; migration dùng user riêng.
- Backup logical `mysqldump` hoặc physical theo kích thước.
- Backup mã hóa, upload storage khác VPS, retention nhiều ngày.
- Cronjob hằng ngày và restore drill định kỳ.
- Theo dõi connection pool, slow query, deadlock, disk usage.
- Không chạy migration destructive tự động mà không review.

Backup mẫu:

```bash
mysqldump --single-transaction --routines --triggers \
  -h mysql -u backup_user -p pos | gzip > /backup/pos-$(date +%F).sql.gz
```

Password không truyền trên command line trong production; dùng secret file hoặc credential helper.

## 8. RabbitMQ production

- Durable quorum queue cho event quan trọng.
- Publisher confirms và manual ack.
- Retry queue + DLQ.
- Management UI chỉ qua VPN/SSH tunnel.
- User publisher và consumer riêng.
- Alert queue depth, unacked messages, disk free và consumer lag.

## 9. CI/CD GitHub Actions

Pipeline đề nghị:

1. Checkout và restore dependency.
2. Lint/build.
3. Unit/integration/API contract test.
4. Dependency/container/security scan.
5. Build image với `git-sha`, push registry.
6. SSH/VPS deploy immutable tag.
7. `docker compose pull`.
8. Chạy migration có kiểm soát.
9. `docker compose up -d`.
10. Health check `/health/live`, `/health/ready`, login và sync smoke test.
11. Nếu fail, rollback image tag trước; migration rollback phải có kế hoạch riêng.

Không deploy `latest`, không ghi secret vào Actions log.

## 10. Health check và observability

API cần:

- `/health/live`: process còn sống, không kiểm tra dependency nặng.
- `/health/ready`: MySQL/RabbitMQ sẵn sàng.
- Correlation ID trong log.
- Metrics request latency, status, sync success/failure, outbox age, RabbitMQ lag.
- Alert 5xx, 401 spike, DLQ > 0, backup fail, disk gần đầy và certificate sắp hết hạn.

Log production dạng structured JSON, redaction PII/token/PIN.

## 11. TLS và certificate

Dùng Let’s Encrypt/Certbot hoặc certificate manager. Tự động renew trước hạn và test renew staging. Mobile phải dùng HTTPS; certificate pinning chỉ thêm khi có quy trình rotation rõ ràng.

## 12. Disaster recovery

- RPO/RTO phải được chốt theo gói SaaS.
- Có bản backup ngoài VPS.
- Có runbook mất VPS, mất DB, RabbitMQ corruption, lộ secret và certificate expiry.
- Test restore sang VPS/staging định kỳ.
- Sau restore kiểm tra tenant count, invoice count, outbox consistency và sync smoke test.
