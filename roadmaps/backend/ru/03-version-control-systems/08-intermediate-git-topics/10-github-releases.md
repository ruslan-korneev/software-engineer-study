# GitHub Releases

## Что такое релизы

**GitHub Releases** — это механизм для публикации версий вашего программного обеспечения. Релизы позволяют:

- Пометить определённую точку в истории как стабильную версию
- Предоставить пользователям готовые к использованию артефакты (бинарные файлы, архивы)
- Документировать изменения между версиями (release notes)
- Автоматизировать процесс выпуска через CI/CD

```
┌─────────────────────────────────────────────────────────────────┐
│                    GitHub Releases Flow                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Git Tag          GitHub Release         Downloads             │
│      │                   │                    │                  │
│      ▼                   ▼                    ▼                  │
│  ┌───────┐          ┌─────────┐         ┌─────────┐            │
│  │v1.0.0 │─────────►│ Release │─────────►│ Assets  │            │
│  └───────┘          │  Page   │         │ .zip    │            │
│                     │         │         │ .tar.gz │            │
│   Точка в          │ - Notes │         │ binary  │            │
│   истории          │ - Assets│         └─────────┘            │
│                     └─────────┘                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Release vs Tag

### Git Tag

**Tag** — это просто метка (указатель) на конкретный коммит в Git.

```bash
# Создание lightweight tag
git tag v1.0.0

# Создание annotated tag (рекомендуется)
git tag -a v1.0.0 -m "Version 1.0.0"

# Отправка тега на remote
git push origin v1.0.0
# или все теги
git push origin --tags
```

### GitHub Release

**Release** — это GitHub-специфичная сущность, которая:
- Создаётся поверх Git tag
- Хранится на GitHub (не в репозитории)
- Может содержать release notes и прикреплённые файлы

```
┌─────────────────────────────────────────────────────────────────┐
│                   Tag vs Release                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   GIT TAG (в репозитории)       GITHUB RELEASE (на GitHub)     │
│                                                                  │
│   ┌───────────────────┐         ┌───────────────────────────┐  │
│   │ v1.0.0            │         │ Release v1.0.0            │  │
│   │                   │────────►│                           │  │
│   │ Просто указатель  │         │ + Release Notes           │  │
│   │ на коммит         │         │ + Прикреплённые файлы     │  │
│   │                   │         │ + Статус (latest/pre)     │  │
│   └───────────────────┘         │ + Auto-generated notes    │  │
│                                  └───────────────────────────┘  │
│                                                                  │
│   Можно иметь tag               Для release НУЖЕН tag          │
│   без release                                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Ключевые отличия

| Аспект | Git Tag | GitHub Release |
|--------|---------|----------------|
| Где хранится | В репозитории Git | На GitHub |
| Синхронизация | git push/pull | Только через GitHub |
| Файлы | Нет | Можно прикрепить |
| Release Notes | Только в annotated | Полноценный markdown |
| Уведомления | Нет | Можно подписаться |

## Создание релиза через UI

### Способ 1: Из существующего тега

```bash
# Сначала создаём тег локально
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0
```

Затем на GitHub:
1. Перейти в репозиторий
2. Нажать "Releases" в правой панели
3. Нажать "Draft a new release"
4. Выбрать существующий тег из списка
5. Заполнить release notes
6. Прикрепить файлы (опционально)
7. Нажать "Publish release"

### Способ 2: Создание тега и релиза одновременно

1. Перейти в "Releases" → "Draft a new release"
2. В поле "Tag" ввести новое имя тега (например, v1.0.0)
3. Выбрать target branch (обычно main)
4. GitHub создаст тег автоматически при публикации

### Опции при создании релиза

```
┌─────────────────────────────────────────────────────────────────┐
│                Draft a new release                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Tag version: [v1.0.0    ▼]  @ Target: [main ▼]               │
│                                                                  │
│   ☐ Create new tag: v1.0.0 on publish                          │
│                                                                  │
│   Release title: [Version 1.0.0                    ]           │
│                                                                  │
│   Describe this release:                                        │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │ ## What's Changed                                       │  │
│   │ * New feature X by @user                                │  │
│   │ * Bug fix Y by @user                                    │  │
│   │                                                         │  │
│   │ **Full Changelog**: link                                │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   [Generate release notes]                                      │
│                                                                  │
│   Attach binaries: [Drop files here or click to upload]        │
│                                                                  │
│   ☐ Set as a pre-release                                       │
│   ☐ Set as the latest release                                  │
│                                                                  │
│   [Save draft]  [Publish release]                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Автоматические release notes

GitHub может автоматически генерировать release notes на основе:
- Pull requests, вошедших в релиз
- Commit messages
- Labels на PR

### Включение автогенерации

1. Нажать "Generate release notes" при создании релиза
2. Или настроить шаблон в `.github/release.yml`

### Конфигурация .github/release.yml

```yaml
# .github/release.yml
changelog:
  exclude:
    labels:
      - ignore-for-release
      - dependencies
    authors:
      - dependabot
      - octocat

  categories:
    - title: "🚀 Features"
      labels:
        - enhancement
        - feature

    - title: "🐛 Bug Fixes"
      labels:
        - bug
        - bugfix
        - fix

    - title: "📚 Documentation"
      labels:
        - documentation
        - docs

    - title: "🔧 Maintenance"
      labels:
        - chore
        - maintenance

    - title: "⚠️ Breaking Changes"
      labels:
        - breaking-change
        - breaking

    - title: "Other Changes"
      labels:
        - "*"
```

### Пример автогенерированных notes

```markdown
## What's Changed

### 🚀 Features
* Add OAuth2 authentication by @developer1 in #123
* Implement caching layer by @developer2 in #145

### 🐛 Bug Fixes
* Fix memory leak in parser by @developer1 in #134
* Resolve race condition by @developer3 in #156

### 📚 Documentation
* Update API documentation by @developer2 in #167

## New Contributors
* @developer3 made their first contribution in #156

**Full Changelog**: https://github.com/org/repo/compare/v0.9.0...v1.0.0
```

## Прикрепление артефактов (binaries)

К релизу можно прикрепить файлы для скачивания:

### Типы файлов

```
┌─────────────────────────────────────────────────────────────────┐
│                    Release Assets Examples                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   myapp-v1.0.0-linux-amd64.tar.gz     (Linux binary)           │
│   myapp-v1.0.0-darwin-amd64.tar.gz    (macOS binary)           │
│   myapp-v1.0.0-windows-amd64.zip      (Windows binary)         │
│   myapp-v1.0.0.deb                    (Debian package)         │
│   myapp-v1.0.0.rpm                    (RPM package)            │
│   checksums.txt                        (SHA256 checksums)       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Загрузка через UI

1. При создании/редактировании релиза
2. Перетащить файлы в область "Attach binaries"
3. Или нажать для выбора файлов

### Ограничения

- Максимальный размер файла: 2 GB
- Максимальное количество файлов: неограничено
- Максимальный общий размер: определяется планом GitHub

### Автоматическая загрузка через GitHub CLI

```bash
# Загрузка артефактов к существующему релизу
gh release upload v1.0.0 ./dist/myapp-linux-amd64.tar.gz

# Загрузка нескольких файлов
gh release upload v1.0.0 \
    ./dist/myapp-linux-amd64.tar.gz \
    ./dist/myapp-darwin-amd64.tar.gz \
    ./dist/myapp-windows-amd64.zip \
    ./dist/checksums.txt

# Перезаписать существующие файлы
gh release upload v1.0.0 ./dist/myapp.tar.gz --clobber
```

## API для релизов

GitHub предоставляет REST API для работы с релизами.

### Получение списка релизов

```bash
# Через curl
curl -s https://api.github.com/repos/owner/repo/releases | jq '.[].tag_name'

# Через GitHub CLI
gh release list
```

### Создание релиза через API

```bash
# Через curl
curl -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/owner/repo/releases \
  -d '{
    "tag_name": "v1.0.0",
    "target_commitish": "main",
    "name": "Release v1.0.0",
    "body": "## Changes\n- Feature X\n- Bug fix Y",
    "draft": false,
    "prerelease": false
  }'

# Через GitHub CLI (рекомендуется)
gh release create v1.0.0 \
    --title "Release v1.0.0" \
    --notes "## Changes
- Feature X
- Bug fix Y"
```

### Загрузка артефактов через API

```bash
# Получаем upload_url из ответа create release
# Затем загружаем файл
curl -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Content-Type: application/gzip" \
  --data-binary @myapp.tar.gz \
  "https://uploads.github.com/repos/owner/repo/releases/12345/assets?name=myapp.tar.gz"
```

### Удаление релиза

```bash
# Через API
curl -X DELETE \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/owner/repo/releases/12345

# Через GitHub CLI
gh release delete v1.0.0 --yes

# Удалить релиз и тег
gh release delete v1.0.0 --yes --cleanup-tag
```

### Получение конкретного релиза

```bash
# Последний релиз
curl -s https://api.github.com/repos/owner/repo/releases/latest | jq '.tag_name'

# По тегу
curl -s https://api.github.com/repos/owner/repo/releases/tags/v1.0.0

# Через gh
gh release view v1.0.0
```

## Автоматизация релизов (GitHub Actions)

### Базовый workflow для создания релиза

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'  # Триггер на теги типа v1.0.0

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Полный workflow с билдом и артефактами

```yaml
# .github/workflows/release.yml
name: Build and Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    strategy:
      matrix:
        include:
          - os: ubuntu-latest
            target: linux-amd64
          - os: macos-latest
            target: darwin-amd64
          - os: windows-latest
            target: windows-amd64

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'

      - name: Build
        run: |
          go build -o myapp-${{ matrix.target }} ./cmd/myapp

      - name: Archive (Linux/macOS)
        if: runner.os != 'Windows'
        run: |
          tar -czvf myapp-${{ github.ref_name }}-${{ matrix.target }}.tar.gz myapp-${{ matrix.target }}

      - name: Archive (Windows)
        if: runner.os == 'Windows'
        run: |
          Compress-Archive -Path myapp-${{ matrix.target }} -DestinationPath myapp-${{ github.ref_name }}-${{ matrix.target }}.zip

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.target }}
          path: myapp-*

  release:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4

      - name: Download all artifacts
        uses: actions/download-artifact@v4
        with:
          path: ./artifacts
          merge-multiple: true

      - name: Generate checksums
        run: |
          cd artifacts
          sha256sum * > checksums.txt

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            artifacts/*
          generate_release_notes: true
          draft: false
          prerelease: ${{ contains(github.ref, 'alpha') || contains(github.ref, 'beta') || contains(github.ref, 'rc') }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Автоматическое определение версии

```yaml
# .github/workflows/release.yml
name: Semantic Release

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install semantic-release
        run: npm install -g semantic-release @semantic-release/git @semantic-release/changelog

      - name: Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: npx semantic-release
```

### Конфигурация semantic-release

```json
// .releaserc.json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    "@semantic-release/github",
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md"],
      "message": "chore(release): ${nextRelease.version} [skip ci]"
    }]
  ]
}
```

## Работа с релизами через GitHub CLI

```bash
# Список релизов
gh release list

# Создание релиза
gh release create v1.0.0 --title "Release 1.0.0" --notes "Release notes here"

# Создание с автогенерацией notes
gh release create v1.0.0 --generate-notes

# Создание из файла с notes
gh release create v1.0.0 --notes-file RELEASE_NOTES.md

# Создание draft релиза
gh release create v1.0.0 --draft

# Создание pre-release
gh release create v1.0.0-beta.1 --prerelease

# Загрузка артефактов
gh release upload v1.0.0 ./dist/*.tar.gz

# Создание с артефактами сразу
gh release create v1.0.0 ./dist/*.tar.gz --generate-notes

# Просмотр релиза
gh release view v1.0.0

# Скачивание артефактов
gh release download v1.0.0

# Удаление релиза
gh release delete v1.0.0 --yes

# Редактирование релиза
gh release edit v1.0.0 --notes "Updated notes"
```

## Best Practices

### 1. Используйте Semantic Versioning

```
MAJOR.MINOR.PATCH

v1.0.0 → v1.0.1  (patch: bug fixes)
v1.0.0 → v1.1.0  (minor: new features, backwards compatible)
v1.0.0 → v2.0.0  (major: breaking changes)

Pre-release: v1.0.0-alpha.1, v1.0.0-beta.1, v1.0.0-rc.1
```

### 2. Пишите информативные release notes

```markdown
## v1.2.0 (2024-01-15)

### ✨ New Features
- Add dark mode support (#123)
- Implement OAuth2 login (#145)

### 🐛 Bug Fixes
- Fix memory leak in cache module (#167)
- Resolve race condition in worker pool (#178)

### 💥 Breaking Changes
- Remove deprecated `oldMethod()` - use `newMethod()` instead
- Config file format changed from YAML to TOML

### 📦 Dependencies
- Upgrade lodash from 4.17.20 to 4.17.21

### 🔧 Internal
- Migrate CI from Travis to GitHub Actions
- Add integration tests
```

### 3. Автоматизируйте процесс

```yaml
# Не делайте релизы вручную!
# Используйте CI/CD
on:
  push:
    tags:
      - 'v*'
```

### 4. Подписывайте релизы

```bash
# Создание подписанного тега
git tag -s v1.0.0 -m "Signed release v1.0.0"

# Проверка подписи
git tag -v v1.0.0
```

### 5. Включайте checksums для бинарных файлов

```bash
# Генерация
sha256sum myapp-*.tar.gz > checksums.txt

# Проверка
sha256sum -c checksums.txt
```

## Полезные ссылки

- [GitHub Releases Documentation](https://docs.github.com/en/repositories/releasing-projects-on-github)
- [Release API](https://docs.github.com/en/rest/releases)
- [GitHub CLI Release Commands](https://cli.github.com/manual/gh_release)
- [Semantic Versioning](https://semver.org/)
- [Conventional Commits](https://www.conventionalcommits.org/)
