# GitHub Packages

## Что такое GitHub Packages?

**GitHub Packages** - это платформа для хостинга пакетов, интегрированная с GitHub. Она позволяет публиковать, хранить и устанавливать пакеты рядом с исходным кодом, используя единую систему аутентификации и прав доступа.

### Основные преимущества

- Единое место для кода и пакетов
- Интеграция с правами доступа репозитория
- Поддержка множества форматов пакетов
- Интеграция с GitHub Actions
- Приватные и публичные пакеты
- Webhooks для автоматизации

## Поддерживаемые форматы

| Реестр | Формат | Клиент |
|--------|--------|--------|
| Container registry | Docker images | docker, podman |
| npm registry | JavaScript | npm, yarn |
| Maven registry | Java | maven, gradle |
| NuGet registry | .NET | nuget, dotnet |
| RubyGems registry | Ruby | gem, bundler |

### URL реестров

```
# Container Registry
ghcr.io

# npm
npm.pkg.github.com

# Maven
maven.pkg.github.com

# NuGet
nuget.pkg.github.com

# RubyGems
rubygems.pkg.github.com
```

## Аутентификация

### Personal Access Token (PAT)

Для работы с GitHub Packages нужен токен с правами:
- `read:packages` - загрузка пакетов
- `write:packages` - публикация пакетов
- `delete:packages` - удаление пакетов

### Создание токена

1. GitHub Settings > Developer settings > Personal access tokens
2. Generate new token (classic)
3. Выберите нужные права (scopes)
4. Сохраните токен

### Аутентификация в разных клиентах

#### Docker

```bash
# Логин в GitHub Container Registry
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
```

#### npm

```bash
# В файле ~/.npmrc
//npm.pkg.github.com/:_authToken=YOUR_TOKEN
@OWNER:registry=https://npm.pkg.github.com
```

#### Maven

```xml
<!-- В ~/.m2/settings.xml -->
<settings>
  <servers>
    <server>
      <id>github</id>
      <username>USERNAME</username>
      <password>TOKEN</password>
    </server>
  </servers>
</settings>
```

## Docker / Container Registry

### Публикация Docker-образа

```bash
# 1. Сборка образа с тегом для ghcr.io
docker build -t ghcr.io/OWNER/IMAGE_NAME:TAG .

# 2. Логин в registry
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# 3. Push образа
docker push ghcr.io/OWNER/IMAGE_NAME:TAG
```

### Dockerfile пример

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "app.py"]
```

### Pull образа

```bash
docker pull ghcr.io/OWNER/IMAGE_NAME:TAG
```

### Multi-platform builds

```bash
# Создание buildx builder
docker buildx create --name mybuilder --use

# Сборка для нескольких платформ
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/OWNER/IMAGE_NAME:TAG \
  --push .
```

## npm Packages

### Настройка проекта

#### package.json

```json
{
  "name": "@OWNER/package-name",
  "version": "1.0.0",
  "description": "My awesome package",
  "main": "index.js",
  "repository": {
    "type": "git",
    "url": "https://github.com/OWNER/REPO.git"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  }
}
```

### Публикация

```bash
# Аутентификация
npm login --registry=https://npm.pkg.github.com

# Публикация
npm publish
```

### Установка пакета

```bash
# Добавить в .npmrc
echo "@OWNER:registry=https://npm.pkg.github.com" >> .npmrc

# Установка
npm install @OWNER/package-name
```

### Пример .npmrc для проекта

```ini
# .npmrc
@myorg:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

## Maven Packages

### Настройка pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project>
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>my-package</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <distributionManagement>
        <repository>
            <id>github</id>
            <name>GitHub Packages</name>
            <url>https://maven.pkg.github.com/OWNER/REPO</url>
        </repository>
    </distributionManagement>

    <repositories>
        <repository>
            <id>github</id>
            <url>https://maven.pkg.github.com/OWNER/REPO</url>
        </repository>
    </repositories>
</project>
```

### Публикация Maven-пакета

```bash
mvn deploy
```

### Использование пакета

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>my-package</artifactId>
    <version>1.0.0</version>
</dependency>
```

## NuGet Packages

### Настройка проекта

```xml
<!-- .csproj файл -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net7.0</TargetFramework>
    <PackageId>OWNER.PackageName</PackageId>
    <Version>1.0.0</Version>
    <Authors>Your Name</Authors>
    <Company>Your Company</Company>
    <RepositoryUrl>https://github.com/OWNER/REPO</RepositoryUrl>
  </PropertyGroup>
</Project>
```

### nuget.config

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="github" value="https://nuget.pkg.github.com/OWNER/index.json" />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <github>
      <add key="Username" value="USERNAME" />
      <add key="ClearTextPassword" value="TOKEN" />
    </github>
  </packageSourceCredentials>
</configuration>
```

### Публикация

```bash
# Создание пакета
dotnet pack --configuration Release

# Публикация
dotnet nuget push "bin/Release/*.nupkg" \
  --source "github" \
  --api-key YOUR_TOKEN
```

## RubyGems Packages

### Настройка gemspec

```ruby
# my_gem.gemspec
Gem::Specification.new do |spec|
  spec.name          = "my_gem"
  spec.version       = "1.0.0"
  spec.authors       = ["Your Name"]
  spec.email         = ["your@email.com"]
  spec.summary       = "A short summary"
  spec.homepage      = "https://github.com/OWNER/REPO"
  spec.license       = "MIT"

  spec.files         = Dir["lib/**/*"]
  spec.require_paths = ["lib"]

  spec.metadata["github_repo"] = "ssh://github.com/OWNER/REPO"
end
```

### Публикация

```bash
# Настройка credentials
echo ":github: Bearer TOKEN" >> ~/.gem/credentials
chmod 0600 ~/.gem/credentials

# Сборка и публикация
gem build my_gem.gemspec
gem push --key github \
  --host https://rubygems.pkg.github.com/OWNER \
  my_gem-1.0.0.gem
```

### Установка

```ruby
# Gemfile
source "https://rubygems.pkg.github.com/OWNER" do
  gem "my_gem", "1.0.0"
end
```

## Интеграция с GitHub Actions

### Публикация Docker-образа

```yaml
name: Publish Docker Image

on:
  release:
    types: [published]
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

### Публикация npm-пакета

```yaml
name: Publish npm Package

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://npm.pkg.github.com'
          scope: '@OWNER'

      - name: Install dependencies
        run: npm ci

      - name: Publish
        run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Публикация Maven-пакета

```yaml
name: Publish Maven Package

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Publish to GitHub Packages
        run: mvn deploy
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Управление пакетами

### Просмотр пакетов

Пакеты доступны:
1. В репозитории: вкладка **Packages**
2. В профиле: раздел **Packages**
3. По прямой ссылке: `github.com/OWNER/REPO/packages`

### Версионирование

Рекомендуется использовать **Semantic Versioning**:

```
MAJOR.MINOR.PATCH

1.0.0 - первый релиз
1.1.0 - новая функциональность (backward compatible)
1.1.1 - исправление бага
2.0.0 - breaking changes
```

### Удаление пакетов

```bash
# Через GitHub CLI (для container images)
gh api \
  --method DELETE \
  -H "Accept: application/vnd.github+json" \
  /user/packages/container/PACKAGE_NAME/versions/VERSION_ID
```

Или через веб-интерфейс: Package Settings > Delete package

### Видимость пакетов

| Тип | Описание |
|-----|----------|
| Private | Только для членов организации/владельца |
| Internal | Для членов enterprise |
| Public | Доступен всем |

## Лимиты и цены

### Бесплатный план

| Ресурс | Лимит |
|--------|-------|
| Хранилище | 500 MB |
| Трафик | 1 GB/месяц |
| Container Registry | Бесплатно для публичных образов |

### Pro/Team/Enterprise

- Увеличенные лимиты хранилища
- Больше трафика
- Приоритетная поддержка

## Полезные практики

### 1. Используйте теги для версий

```bash
# Плохо
docker push ghcr.io/owner/app:latest

# Хорошо
docker push ghcr.io/owner/app:1.2.3
docker push ghcr.io/owner/app:1.2
docker push ghcr.io/owner/app:1
```

### 2. Автоматизируйте через Actions

```yaml
# Автоматическая публикация при создании тега
on:
  push:
    tags:
      - 'v*'
```

### 3. Используйте GITHUB_TOKEN

В GitHub Actions используйте встроенный `GITHUB_TOKEN` вместо PAT:

```yaml
env:
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 4. Настройте Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    registries:
      - ghcr

registries:
  ghcr:
    type: docker-registry
    url: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

### 5. Сканируйте уязвимости

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'ghcr.io/${{ github.repository }}:${{ github.sha }}'
    format: 'sarif'
    output: 'trivy-results.sarif'

- name: Upload Trivy scan results
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: 'trivy-results.sarif'
```

## Полезные ссылки

- [GitHub Packages Documentation](https://docs.github.com/en/packages)
- [Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [npm Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-npm-registry)
- [Working with Actions](https://docs.github.com/en/packages/managing-github-packages-using-github-actions-workflows)
