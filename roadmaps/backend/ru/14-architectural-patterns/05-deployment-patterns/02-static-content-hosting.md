# Static Content Hosting (Хостинг статического контента)

## Определение

**Static Content Hosting** — это паттерн развёртывания, при котором статические ресурсы (HTML, CSS, JavaScript, изображения, шрифты, видео) размещаются отдельно от серверов приложений на специализированных системах, оптимизированных для быстрой доставки контента.

Этот паттерн является фундаментом современных веб-архитектур, позволяя разгрузить серверы приложений и значительно улучшить производительность за счёт использования CDN, кэширования и географического распределения.

```
┌─────────────────────────────────────────────────────────────────┐
│              STATIC CONTENT HOSTING ARCHITECTURE                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                      ┌─────────┐                                │
│                      │ Browser │                                │
│                      └────┬────┘                                │
│                           │                                     │
│           ┌───────────────┼───────────────┐                    │
│           │               │               │                     │
│           ▼               ▼               ▼                     │
│    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐             │
│    │   Static    │ │     CDN     │ │   API       │             │
│    │   Host      │ │   (Edge)    │ │   Server    │             │
│    │             │ │             │ │             │             │
│    │  HTML/CSS   │ │ Images/JS   │ │  JSON/Data  │             │
│    │  favicon    │ │ Large files │ │  Business   │             │
│    └─────────────┘ └─────────────┘ └─────────────┘             │
│          │               │               │                      │
│          │               │               │                      │
│          ▼               ▼               ▼                      │
│    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐             │
│    │   Object    │ │   Origin    │ │  Database   │             │
│    │   Storage   │ │   Storage   │ │             │             │
│    │   (S3)      │ │             │ │             │             │
│    └─────────────┘ └─────────────┘ └─────────────┘             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Ключевые характеристики

### 1. Типы статического контента

```
┌─────────────────────────────────────────────────────────────────┐
│                   ТИПЫ СТАТИЧЕСКОГО КОНТЕНТА                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ IMMUTABLE (Неизменяемый)                                 │  │
│  │                                                          │  │
│  │ • Версионированные бандлы: app.a1b2c3.js               │  │
│  │ • Хэшированные ассеты: styles.d4e5f6.css               │  │
│  │ • Сжатые изображения: hero.webp                         │  │
│  │                                                          │  │
│  │ Cache-Control: public, max-age=31536000, immutable       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ MUTABLE (Изменяемый)                                     │  │
│  │                                                          │  │
│  │ • HTML документы: index.html                             │  │
│  │ • Манифесты: manifest.json, sitemap.xml                  │  │
│  │ • Конфигурации: config.json                              │  │
│  │                                                          │  │
│  │ Cache-Control: public, max-age=0, must-revalidate        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ USER-GENERATED (Пользовательский)                        │  │
│  │                                                          │  │
│  │ • Аватары пользователей                                  │  │
│  │ • Загруженные документы                                  │  │
│  │ • Медиа-контент                                          │  │
│  │                                                          │  │
│  │ Cache-Control: public, max-age=86400                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Модели хостинга

| Модель | Описание | Примеры |
|--------|----------|---------|
| **Object Storage** | Бессерверное хранилище файлов | AWS S3, GCS, Azure Blob |
| **CDN** | Глобальная сеть доставки | CloudFlare, Fastly, Akamai |
| **Static Site Hosting** | Специализированные платформы | Netlify, Vercel, GitHub Pages |
| **Container-based** | Nginx/Apache в контейнере | Docker + Nginx |
| **Edge Functions** | Код на границе сети | CloudFlare Workers, Vercel Edge |

### 3. Стратегии кэширования

```python
# Примеры Cache-Control заголовков для разных типов контента

CACHE_STRATEGIES = {
    # Статические ассеты с хэшем в имени - кэшировать навсегда
    "immutable_assets": {
        "pattern": r"\.(js|css|woff2?|ttf|eot)$",
        "headers": {
            "Cache-Control": "public, max-age=31536000, immutable",
            "Vary": "Accept-Encoding"
        }
    },

    # Изображения - долгий кэш, но с revalidation
    "images": {
        "pattern": r"\.(png|jpg|jpeg|gif|webp|svg|ico)$",
        "headers": {
            "Cache-Control": "public, max-age=2592000",  # 30 дней
            "Vary": "Accept"
        }
    },

    # HTML - короткий кэш или no-cache
    "html": {
        "pattern": r"\.html$",
        "headers": {
            "Cache-Control": "public, max-age=0, must-revalidate",
            "Vary": "Accept-Encoding"
        }
    },

    # API responses cached at edge
    "api_cacheable": {
        "pattern": r"/api/v1/public/",
        "headers": {
            "Cache-Control": "public, s-maxage=60, stale-while-revalidate=30",
            "Vary": "Accept, Accept-Encoding"
        }
    }
}
```

## Когда использовать

### Идеальные сценарии

1. **Single Page Applications (SPA)**
   - React, Vue, Angular приложения
   - Статический бандл + динамический API

2. **Статические сайты**
   - Блоги, документация, лендинги
   - Генераторы: Next.js SSG, Gatsby, Hugo

3. **Высоконагруженные проекты**
   - Миллионы запросов в день
   - Глобальная аудитория

4. **Медиа-контент**
   - Изображения, видео, аудио
   - Стриминговые платформы

5. **E-commerce**
   - Каталоги товаров
   - Изображения продуктов

### Когда НЕ подходит

```
┌─────────────────────────────────────────────────────────────────┐
│                      НЕПОДХОДЯЩИЕ СЛУЧАИ                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ Персонализированный контент                                │
│     → Используйте Edge Functions или SSR                       │
│                                                                 │
│  ❌ Real-time данные                                           │
│     → WebSockets, Server-Sent Events                           │
│                                                                 │
│  ❌ Приватный контент без авторизации                          │
│     → Signed URLs, token-based access                          │
│                                                                 │
│  ❌ Часто меняющийся контент                                   │
│     → ISR (Incremental Static Regeneration)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|-------------|----------|
| **Высокая производительность** | Контент ближе к пользователю, быстрая доставка |
| **Масштабируемость** | CDN автоматически масштабируется |
| **Низкая стоимость** | Дешевле, чем вычислительные ресурсы |
| **Надёжность** | Множество edge-серверов, отказоустойчивость |
| **Безопасность** | Изоляция от backend, DDoS защита |
| **Простота** | Нет серверной логики для статики |
| **SEO** | Быстрая загрузка улучшает ранжирование |

### Недостатки

| Недостаток | Описание |
|-----------|----------|
| **Cache invalidation** | Сложность инвалидации кэша |
| **Stale content** | Риск показа устаревшего контента |
| **Vendor lock-in** | Зависимость от провайдера |
| **Настройка CORS** | Сложности с cross-origin запросами |
| **Стоимость egress** | Плата за исходящий трафик |
| **Отладка** | Сложнее отлаживать проблемы кэширования |

## Примеры реализации

### Пример 1: AWS S3 + CloudFront

```python
# infrastructure/static_hosting.py
# Terraform-like конфигурация на Python (Pulumi)

import pulumi
import pulumi_aws as aws
from pulumi_aws import s3, cloudfront, route53

class StaticHosting:
    """
    Инфраструктура для хостинга статического контента
    на AWS S3 с CloudFront CDN
    """

    def __init__(self, domain: str, environment: str):
        self.domain = domain
        self.environment = environment
        self.bucket_name = f"{domain.replace('.', '-')}-{environment}"

    def create_s3_bucket(self):
        """Создание S3 bucket для статических файлов"""

        # Основной bucket
        self.bucket = s3.Bucket(
            f"{self.bucket_name}-bucket",
            bucket=self.bucket_name,
            acl="private",  # Публичный доступ через CloudFront
            website=s3.BucketWebsiteArgs(
                index_document="index.html",
                error_document="404.html"
            ),
            versioning=s3.BucketVersioningArgs(
                enabled=True
            ),
            # CORS для API запросов
            cors_rules=[
                s3.BucketCorsRuleArgs(
                    allowed_headers=["*"],
                    allowed_methods=["GET", "HEAD"],
                    allowed_origins=[f"https://{self.domain}"],
                    expose_headers=["ETag"],
                    max_age_seconds=3600
                )
            ],
            # Жизненный цикл для старых версий
            lifecycle_rules=[
                s3.BucketLifecycleRuleArgs(
                    enabled=True,
                    noncurrent_version_expiration=s3.BucketLifecycleRuleNoncurrentVersionExpirationArgs(
                        days=30
                    )
                )
            ],
            tags={
                "Environment": self.environment,
                "Purpose": "static-hosting"
            }
        )

        # Политика для CloudFront
        self.bucket_policy = s3.BucketPolicy(
            f"{self.bucket_name}-policy",
            bucket=self.bucket.id,
            policy=self.bucket.arn.apply(
                lambda arn: self._create_bucket_policy(arn)
            )
        )

        return self.bucket

    def _create_bucket_policy(self, bucket_arn: str) -> str:
        """Политика доступа только для CloudFront"""
        return f"""{{
            "Version": "2012-10-17",
            "Statement": [
                {{
                    "Sid": "AllowCloudFrontAccess",
                    "Effect": "Allow",
                    "Principal": {{
                        "Service": "cloudfront.amazonaws.com"
                    }},
                    "Action": "s3:GetObject",
                    "Resource": "{bucket_arn}/*",
                    "Condition": {{
                        "StringEquals": {{
                            "AWS:SourceArn": "{self.distribution_arn}"
                        }}
                    }}
                }}
            ]
        }}"""

    def create_cloudfront_distribution(self):
        """Создание CloudFront distribution"""

        # Origin Access Control для безопасного доступа к S3
        oac = cloudfront.OriginAccessControl(
            f"{self.bucket_name}-oac",
            name=f"{self.bucket_name}-oac",
            origin_access_control_origin_type="s3",
            signing_behavior="always",
            signing_protocol="sigv4"
        )

        # CloudFront Distribution
        self.distribution = cloudfront.Distribution(
            f"{self.bucket_name}-cdn",
            enabled=True,
            is_ipv6_enabled=True,
            default_root_object="index.html",
            price_class="PriceClass_100",  # Только США и Европа

            # Origins
            origins=[
                cloudfront.DistributionOriginArgs(
                    domain_name=self.bucket.bucket_regional_domain_name,
                    origin_id="S3Origin",
                    origin_access_control_id=oac.id
                )
            ],

            # Поведение по умолчанию
            default_cache_behavior=cloudfront.DistributionDefaultCacheBehaviorArgs(
                allowed_methods=["GET", "HEAD", "OPTIONS"],
                cached_methods=["GET", "HEAD"],
                target_origin_id="S3Origin",
                viewer_protocol_policy="redirect-to-https",
                compress=True,

                # Кэширование
                cache_policy_id=self._get_cache_policy_id(),
                origin_request_policy_id=self._get_origin_request_policy_id(),

                # Функции
                function_associations=[
                    cloudfront.DistributionDefaultCacheBehaviorFunctionAssociationArgs(
                        event_type="viewer-response",
                        function_arn=self.security_headers_function.arn
                    )
                ]
            ),

            # Кастомные поведения для разных типов контента
            ordered_cache_behaviors=[
                # Immutable assets - долгий кэш
                cloudfront.DistributionOrderedCacheBehaviorArgs(
                    path_pattern="/static/*",
                    allowed_methods=["GET", "HEAD"],
                    cached_methods=["GET", "HEAD"],
                    target_origin_id="S3Origin",
                    viewer_protocol_policy="redirect-to-https",
                    compress=True,
                    min_ttl=31536000,
                    max_ttl=31536000,
                    default_ttl=31536000
                ),
                # API responses - короткий кэш
                cloudfront.DistributionOrderedCacheBehaviorArgs(
                    path_pattern="/api/*",
                    allowed_methods=["GET", "HEAD", "OPTIONS"],
                    cached_methods=["GET", "HEAD"],
                    target_origin_id="APIOrigin",
                    viewer_protocol_policy="https-only",
                    compress=True,
                    min_ttl=0,
                    max_ttl=60,
                    default_ttl=0
                )
            ],

            # Custom error responses (SPA routing)
            custom_error_responses=[
                cloudfront.DistributionCustomErrorResponseArgs(
                    error_code=404,
                    response_code=200,
                    response_page_path="/index.html",
                    error_caching_min_ttl=0
                ),
                cloudfront.DistributionCustomErrorResponseArgs(
                    error_code=403,
                    response_code=200,
                    response_page_path="/index.html",
                    error_caching_min_ttl=0
                )
            ],

            # SSL Certificate
            viewer_certificate=cloudfront.DistributionViewerCertificateArgs(
                acm_certificate_arn=self.certificate.arn,
                ssl_support_method="sni-only",
                minimum_protocol_version="TLSv1.2_2021"
            ),

            aliases=[self.domain, f"www.{self.domain}"],

            restrictions=cloudfront.DistributionRestrictionsArgs(
                geo_restriction=cloudfront.DistributionRestrictionsGeoRestrictionArgs(
                    restriction_type="none"
                )
            ),

            tags={
                "Environment": self.environment
            }
        )

        return self.distribution

    def create_security_headers_function(self):
        """CloudFront Function для security headers"""

        function_code = """
        function handler(event) {
            var response = event.response;
            var headers = response.headers;

            // Security headers
            headers['strict-transport-security'] = {
                value: 'max-age=31536000; includeSubDomains; preload'
            };
            headers['x-content-type-options'] = { value: 'nosniff' };
            headers['x-frame-options'] = { value: 'DENY' };
            headers['x-xss-protection'] = { value: '1; mode=block' };
            headers['referrer-policy'] = { value: 'strict-origin-when-cross-origin' };
            headers['permissions-policy'] = {
                value: 'camera=(), microphone=(), geolocation=()'
            };

            // CSP для SPA
            headers['content-security-policy'] = {
                value: "default-src 'self'; " +
                       "script-src 'self' 'unsafe-inline'; " +
                       "style-src 'self' 'unsafe-inline'; " +
                       "img-src 'self' data: https:; " +
                       "font-src 'self'; " +
                       "connect-src 'self' https://api.example.com;"
            };

            return response;
        }
        """

        self.security_headers_function = cloudfront.Function(
            f"{self.bucket_name}-security-headers",
            name=f"{self.bucket_name}-security-headers",
            runtime="cloudfront-js-1.0",
            code=function_code
        )

        return self.security_headers_function
```

### Пример 2: Nginx конфигурация для статики

```nginx
# /etc/nginx/conf.d/static-hosting.conf

# Upstream для API (если нужен)
upstream api_backend {
    server api:8000;
    keepalive 32;
}

# Кэш для статических файлов
proxy_cache_path /var/cache/nginx/static
    levels=1:2
    keys_zone=static_cache:100m
    max_size=10g
    inactive=7d
    use_temp_path=off;

server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    # Редирект на HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # Modern SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Root directory
    root /var/www/static;
    index index.html;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json
               application/javascript application/rss+xml
               application/atom+xml image/svg+xml;

    # Brotli compression (if module installed)
    brotli on;
    brotli_comp_level 6;
    brotli_types text/plain text/css text/xml application/json
                 application/javascript application/rss+xml
                 application/atom+xml image/svg+xml;

    # ===============================================
    # Статические ассеты с хэшем - кэшировать навсегда
    # ===============================================
    location ~* \.(js|css)$ {
        expires 1y;
        add_header Cache-Control "public, max-age=31536000, immutable";
        add_header X-Content-Type-Options "nosniff";

        # Версионированные файлы
        try_files $uri =404;

        # Логирование
        access_log off;
    }

    # ===============================================
    # Изображения и медиа
    # ===============================================
    location ~* \.(jpg|jpeg|png|gif|webp|svg|ico|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, max-age=2592000";
        add_header X-Content-Type-Options "nosniff";

        # WebP fallback
        location ~* \.(jpg|jpeg|png)$ {
            add_header Vary "Accept";
            try_files $uri$webp_suffix $uri =404;
        }

        access_log off;
    }

    # ===============================================
    # HTML файлы - не кэшировать
    # ===============================================
    location ~* \.html$ {
        expires -1;
        add_header Cache-Control "no-cache, no-store, must-revalidate";
        add_header Pragma "no-cache";

        # Security headers
        add_header X-Frame-Options "DENY";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Content-Type-Options "nosniff";
        add_header Referrer-Policy "strict-origin-when-cross-origin";
    }

    # ===============================================
    # SPA routing - все неизвестные пути на index.html
    # ===============================================
    location / {
        try_files $uri $uri/ /index.html;

        # Security headers для index.html
        add_header X-Frame-Options "DENY";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Content-Type-Options "nosniff";
        add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self' https://api.example.com;";
    }

    # ===============================================
    # API Proxy
    # ===============================================
    location /api/ {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # CORS headers
        add_header Access-Control-Allow-Origin "https://example.com" always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Authorization, Content-Type" always;

        if ($request_method = 'OPTIONS') {
            add_header Access-Control-Max-Age 1728000;
            add_header Content-Type 'text/plain charset=UTF-8';
            add_header Content-Length 0;
            return 204;
        }

        # Кэширование GET запросов
        proxy_cache static_cache;
        proxy_cache_methods GET HEAD;
        proxy_cache_valid 200 1m;
        proxy_cache_valid 404 1m;
        proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;
        add_header X-Cache-Status $upstream_cache_status;
    }

    # ===============================================
    # Health check
    # ===============================================
    location /health {
        access_log off;
        return 200 "OK\n";
        add_header Content-Type text/plain;
    }

    # Deny access to hidden files
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

### Пример 3: CI/CD для деплоя статики

```yaml
# .github/workflows/deploy-static.yml

name: Deploy Static Content

on:
  push:
    branches: [main]
    paths:
      - 'frontend/**'
      - '.github/workflows/deploy-static.yml'

env:
  AWS_REGION: eu-west-1
  S3_BUCKET: example-com-static-prod
  CLOUDFRONT_DISTRIBUTION_ID: E1234567890ABC

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        working-directory: frontend
        run: npm ci

      - name: Build
        working-directory: frontend
        run: npm run build
        env:
          VITE_API_URL: https://api.example.com
          VITE_ENV: production

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: static-build
          path: frontend/dist
          retention-days: 7

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: static-build
          path: dist

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Sync immutable assets to S3
        run: |
          # JS, CSS, fonts - с долгим кэшем
          aws s3 sync dist/assets s3://${{ env.S3_BUCKET }}/assets \
            --cache-control "public, max-age=31536000, immutable" \
            --delete

      - name: Sync HTML and other files to S3
        run: |
          # HTML файлы - без кэша
          aws s3 sync dist s3://${{ env.S3_BUCKET }} \
            --exclude "assets/*" \
            --cache-control "public, max-age=0, must-revalidate" \
            --delete

      - name: Invalidate CloudFront cache
        run: |
          # Инвалидируем только index.html и корневые файлы
          aws cloudfront create-invalidation \
            --distribution-id ${{ env.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/index.html" "/manifest.json" "/robots.txt" "/"

      - name: Verify deployment
        run: |
          # Проверяем, что сайт доступен
          sleep 30
          curl -sSf https://example.com > /dev/null
          echo "Deployment verified successfully"

      - name: Notify on success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Static content deployed successfully",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Static Content Deployed* :rocket:\nCommit: `${{ github.sha }}`\nBranch: `${{ github.ref_name }}`"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Пример 4: Версионирование и rollback

```python
# deploy/static_versioner.py

import os
import json
import boto3
from datetime import datetime
from typing import Optional
import hashlib

class StaticVersioner:
    """
    Управление версиями статического контента
    с поддержкой rollback
    """

    def __init__(self, bucket_name: str, distribution_id: str):
        self.s3 = boto3.client('s3')
        self.cloudfront = boto3.client('cloudfront')
        self.bucket_name = bucket_name
        self.distribution_id = distribution_id
        self.manifest_key = '_deployments/manifest.json'

    def deploy(self, build_dir: str, version: Optional[str] = None) -> dict:
        """
        Деплой новой версии статического контента
        """
        if version is None:
            version = self._generate_version()

        deployment = {
            'version': version,
            'timestamp': datetime.utcnow().isoformat(),
            'files': []
        }

        # Загрузка файлов с версионированием
        for root, dirs, files in os.walk(build_dir):
            for file in files:
                local_path = os.path.join(root, file)
                relative_path = os.path.relpath(local_path, build_dir)

                # Определяем S3 ключ
                if self._is_immutable_asset(relative_path):
                    # Ассеты идут в папку версии
                    s3_key = f"_versions/{version}/{relative_path}"
                else:
                    # HTML и манифесты - в корень
                    s3_key = relative_path

                # Загружаем файл
                content_type = self._get_content_type(file)
                cache_control = self._get_cache_control(relative_path)

                with open(local_path, 'rb') as f:
                    content = f.read()
                    file_hash = hashlib.md5(content).hexdigest()

                self.s3.put_object(
                    Bucket=self.bucket_name,
                    Key=s3_key,
                    Body=content,
                    ContentType=content_type,
                    CacheControl=cache_control,
                    Metadata={
                        'version': version,
                        'hash': file_hash
                    }
                )

                deployment['files'].append({
                    'path': relative_path,
                    's3_key': s3_key,
                    'hash': file_hash
                })

        # Сохраняем манифест версии
        self._save_version_manifest(deployment)

        # Обновляем текущую версию
        self._set_current_version(version)

        # Инвалидируем кэш
        self._invalidate_cache()

        return deployment

    def rollback(self, target_version: str) -> dict:
        """
        Откат на предыдущую версию
        """
        # Проверяем, существует ли версия
        version_manifest = self._get_version_manifest(target_version)
        if not version_manifest:
            raise ValueError(f"Version {target_version} not found")

        # Обновляем ссылки в корне на старую версию
        for file_info in version_manifest['files']:
            if not self._is_immutable_asset(file_info['path']):
                # Копируем файл из версии в корень
                self.s3.copy_object(
                    Bucket=self.bucket_name,
                    CopySource={
                        'Bucket': self.bucket_name,
                        'Key': file_info['s3_key']
                    },
                    Key=file_info['path'],
                    CacheControl=self._get_cache_control(file_info['path'])
                )

        # Обновляем текущую версию
        self._set_current_version(target_version)

        # Инвалидируем кэш
        self._invalidate_cache()

        return {
            'action': 'rollback',
            'version': target_version,
            'timestamp': datetime.utcnow().isoformat()
        }

    def list_versions(self, limit: int = 10) -> list:
        """Получить список доступных версий"""
        manifest = self._get_deployment_manifest()
        return manifest.get('versions', [])[-limit:]

    def get_current_version(self) -> str:
        """Получить текущую активную версию"""
        manifest = self._get_deployment_manifest()
        return manifest.get('current_version', 'unknown')

    def cleanup_old_versions(self, keep_count: int = 5):
        """Удалить старые версии, оставив последние N"""
        manifest = self._get_deployment_manifest()
        versions = manifest.get('versions', [])

        if len(versions) <= keep_count:
            return {'deleted': 0}

        versions_to_delete = versions[:-keep_count]
        deleted_count = 0

        for version_info in versions_to_delete:
            version = version_info['version']
            # Удаляем файлы версии
            self._delete_version_files(version)
            deleted_count += 1

        # Обновляем манифест
        manifest['versions'] = versions[-keep_count:]
        self._save_deployment_manifest(manifest)

        return {'deleted': deleted_count}

    def _generate_version(self) -> str:
        """Генерация уникального идентификатора версии"""
        timestamp = datetime.utcnow().strftime('%Y%m%d%H%M%S')
        return f"v{timestamp}"

    def _is_immutable_asset(self, path: str) -> bool:
        """Проверка, является ли файл иммутабельным ассетом"""
        immutable_extensions = {'.js', '.css', '.woff', '.woff2', '.ttf', '.eot'}
        return any(path.endswith(ext) for ext in immutable_extensions)

    def _get_cache_control(self, path: str) -> str:
        """Определение Cache-Control для файла"""
        if self._is_immutable_asset(path):
            return "public, max-age=31536000, immutable"
        elif path.endswith('.html'):
            return "public, max-age=0, must-revalidate"
        else:
            return "public, max-age=86400"

    def _get_content_type(self, filename: str) -> str:
        """Определение Content-Type по расширению"""
        content_types = {
            '.html': 'text/html',
            '.css': 'text/css',
            '.js': 'application/javascript',
            '.json': 'application/json',
            '.png': 'image/png',
            '.jpg': 'image/jpeg',
            '.svg': 'image/svg+xml',
            '.woff': 'font/woff',
            '.woff2': 'font/woff2',
        }
        ext = os.path.splitext(filename)[1].lower()
        return content_types.get(ext, 'application/octet-stream')

    def _invalidate_cache(self):
        """Инвалидация CloudFront кэша"""
        self.cloudfront.create_invalidation(
            DistributionId=self.distribution_id,
            InvalidationBatch={
                'Paths': {
                    'Quantity': 3,
                    'Items': ['/index.html', '/manifest.json', '/']
                },
                'CallerReference': str(datetime.utcnow().timestamp())
            }
        )
```

## Best practices и антипаттерны

### Best Practices

```
┌─────────────────────────────────────────────────────────────────┐
│                      BEST PRACTICES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ Используйте content hashing                                │
│     Добавляйте хэш контента в имена файлов                     │
│     app.a1b2c3d4.js вместо app.js                              │
│                                                                 │
│  ✅ Разделяйте mutable и immutable контент                     │
│     Разные стратегии кэширования для разных типов              │
│                                                                 │
│  ✅ Сжимайте контент                                           │
│     Gzip/Brotli для текстовых файлов                           │
│     WebP/AVIF для изображений                                  │
│                                                                 │
│  ✅ Настройте правильные заголовки                             │
│     Cache-Control, ETag, Content-Type                          │
│                                                                 │
│  ✅ Используйте CDN                                            │
│     Географическое распределение для низкой латентности        │
│                                                                 │
│  ✅ Версионируйте деплои                                       │
│     Возможность быстрого rollback                              │
│                                                                 │
│  ✅ Мониторьте производительность                              │
│     Core Web Vitals, cache hit ratio                           │
│                                                                 │
│  ✅ Добавьте security headers                                  │
│     CSP, HSTS, X-Frame-Options                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Антипаттерны

| Антипаттерн | Проблема | Решение |
|------------|----------|---------|
| **Долгий кэш для HTML** | Пользователи видят старую версию | max-age=0, must-revalidate |
| **Без версионирования ассетов** | Проблемы при обновлении | Content hashing в именах |
| **Один Cache-Control для всего** | Неоптимальное кэширование | Разные политики по типам |
| **Игнорирование CORS** | Ошибки при API запросах | Правильная настройка headers |
| **Инвалидация всего кэша** | Дорого и медленно | Точечная инвалидация |
| **Отсутствие сжатия** | Медленная загрузка | Gzip/Brotli на сервере |
| **Большие бандлы** | Долгая первая загрузка | Code splitting |

## Связанные паттерны

```
┌─────────────────────────────────────────────────────────────────┐
│                   СВЯЗАННЫЕ ПАТТЕРНЫ                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │ CDN             │     │ Edge Computing  │                   │
│  └────────┬────────┘     └────────┬────────┘                   │
│           │                       │                             │
│           ▼                       ▼                             │
│  Глобальная доставка      Вычисления на                        │
│  контента                 границе сети                         │
│                                                                 │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │ Jamstack        │     │ Backend for     │                   │
│  │                 │     │ Frontend (BFF)  │                   │
│  └────────┬────────┘     └────────┬────────┘                   │
│           │                       │                             │
│           ▼                       ▼                             │
│  JavaScript, APIs,        API Gateway для                      │
│  Markup architecture      каждого клиента                      │
│                                                                 │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │ Blue-Green      │     │ Immutable       │                   │
│  │ Deployment      │     │ Infrastructure  │                   │
│  └────────┬────────┘     └────────┬────────┘                   │
│           │                       │                             │
│           ▼                       ▼                             │
│  Два окружения для        Инфраструктура как                   │
│  zero-downtime            неизменяемый артефакт                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Ресурсы для изучения

### Документация
- [AWS S3 Static Website Hosting](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html)
- [CloudFront Developer Guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/)
- [Netlify Docs](https://docs.netlify.com/)
- [Vercel Documentation](https://vercel.com/docs)

### Инструменты оптимизации
- **Lighthouse** - аудит производительности
- **WebPageTest** - детальный анализ загрузки
- **Bundlephobia** - размер npm пакетов
- **Squoosh** - сжатие изображений

### Статьи
- [The Cost of JavaScript](https://v8.dev/blog/cost-of-javascript-2019) - V8 Blog
- [HTTP Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching) - MDN
- [Jamstack](https://jamstack.org/) - архитектура современных сайтов

### Курсы
- [Web.dev Learn Performance](https://web.dev/learn/#performance)
- [Frontend Masters - Web Performance](https://frontendmasters.com/courses/web-performance/)
