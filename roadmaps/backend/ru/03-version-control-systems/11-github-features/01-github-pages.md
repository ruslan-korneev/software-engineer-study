# GitHub Pages

## Что такое GitHub Pages?

**GitHub Pages** - это бесплатный хостинг для статических сайтов, интегрированный непосредственно с репозиториями GitHub. Он позволяет публиковать веб-страницы прямо из репозитория без необходимости настраивать собственный сервер.

### Основные возможности

- Бесплатный хостинг статических сайтов
- Автоматическое развертывание при push в репозиторий
- Поддержка Jekyll для генерации сайтов
- Возможность использования кастомных доменов
- Автоматический HTTPS

## Типы сайтов GitHub Pages

### 1. User/Organization Site (Сайт пользователя/организации)

Персональный или организационный сайт, доступный по адресу `username.github.io` или `orgname.github.io`.

**Особенности:**
- Репозиторий должен называться `username.github.io`
- Только один сайт на пользователя/организацию
- Публикуется из ветки `main` или `master`
- URL: `https://username.github.io`

```bash
# Создание репозитория для персонального сайта
git init username.github.io
cd username.github.io
echo "<h1>Hello, World!</h1>" > index.html
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/username/username.github.io.git
git push -u origin main
```

### 2. Project Site (Сайт проекта)

Сайт для конкретного проекта/репозитория.

**Особенности:**
- Может быть создан для любого репозитория
- URL: `https://username.github.io/repository-name`
- Публикуется из выбранной ветки или папки `/docs`

## Настройка GitHub Pages

### Через Settings репозитория

1. Откройте репозиторий на GitHub
2. Перейдите в **Settings** > **Pages**
3. В разделе **Source** выберите:
   - **Branch**: ветка для публикации (например, `main`, `gh-pages`)
   - **Folder**: корневая папка `/` или `/docs`
4. Нажмите **Save**

### Источники публикации

| Источник | Описание |
|----------|----------|
| `main` branch | Публикация из корня основной ветки |
| `gh-pages` branch | Отдельная ветка для сайта |
| `/docs` folder | Папка docs в основной ветке |
| GitHub Actions | Кастомный workflow для сборки |

### Пример структуры проекта с /docs

```
my-project/
├── src/
│   └── app.py
├── docs/
│   ├── index.html
│   ├── styles.css
│   └── about.html
└── README.md
```

## Jekyll и статические генераторы

### Jekyll

**Jekyll** - это генератор статических сайтов, встроенный в GitHub Pages. Он автоматически преобразует Markdown-файлы в HTML.

#### Базовая структура Jekyll-сайта

```
my-jekyll-site/
├── _config.yml          # Конфигурация сайта
├── _posts/              # Блог-посты
│   └── 2024-01-15-my-first-post.md
├── _layouts/            # Шаблоны страниц
│   ├── default.html
│   └── post.html
├── _includes/           # Переиспользуемые компоненты
│   ├── header.html
│   └── footer.html
├── assets/              # Статические ресурсы
│   ├── css/
│   └── images/
└── index.md             # Главная страница
```

#### Файл конфигурации _config.yml

```yaml
title: My Awesome Site
description: A personal blog about programming
author: John Doe
email: john@example.com

# Тема
theme: minima

# Плагины
plugins:
  - jekyll-feed
  - jekyll-seo-tag

# Настройки сборки
markdown: kramdown
highlighter: rouge

# Исключения из сборки
exclude:
  - Gemfile
  - Gemfile.lock
  - node_modules
```

#### Пример поста

```markdown
---
layout: post
title: "Мой первый пост"
date: 2024-01-15 10:00:00 +0300
categories: blog
tags: [начало, программирование]
---

# Добро пожаловать!

Это мой первый пост в блоге на GitHub Pages.

## Код

```python
def hello():
    print("Hello, World!")
```
```

### Темы Jekyll

GitHub Pages поддерживает несколько встроенных тем:

- **minima** (по умолчанию)
- **architect**
- **cayman**
- **dinky**
- **hacker**
- **leap-day**
- **merlot**
- **midnight**
- **minimal**
- **modernist**
- **slate**
- **tactile**
- **time-machine**

```yaml
# В _config.yml
theme: cayman
```

### Другие статические генераторы

Через GitHub Actions можно использовать любой генератор:

| Генератор | Язык | Особенности |
|-----------|------|-------------|
| Hugo | Go | Очень быстрый |
| Gatsby | React | SPA-приложения |
| Next.js | React | SSG + SSR |
| VuePress | Vue | Документация |
| MkDocs | Python | Техническая документация |
| Docusaurus | React | Документация |

#### Пример workflow для Hugo

```yaml
# .github/workflows/hugo.yml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build
        run: hugo --minify

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v2
        with:
          path: ./public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v3
```

## Кастомные домены

### Настройка кастомного домена

1. В **Settings** > **Pages** введите ваш домен в поле **Custom domain**
2. Настройте DNS-записи у вашего регистратора

### DNS-записи для apex-домена (example.com)

Создайте A-записи, указывающие на IP-адреса GitHub:

```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

### DNS-записи для субдомена (www.example.com)

Создайте CNAME-запись:

```
www.example.com -> username.github.io
```

### Файл CNAME

GitHub автоматически создает файл `CNAME` в корне репозитория:

```
example.com
```

### Проверка DNS

```bash
# Проверка A-записей
dig example.com +noall +answer

# Проверка CNAME
dig www.example.com +noall +answer
```

## HTTPS

### Автоматический HTTPS

GitHub Pages автоматически предоставляет HTTPS-сертификаты через Let's Encrypt.

**Для включения:**
1. Перейдите в **Settings** > **Pages**
2. Отметьте **Enforce HTTPS**

### Особенности HTTPS

- Бесплатные SSL-сертификаты
- Автоматическое продление
- Поддержка кастомных доменов
- Принудительное перенаправление с HTTP на HTTPS

### Время активации

- Для `*.github.io`: мгновенно
- Для кастомных доменов: до 24 часов (после настройки DNS)

## Ограничения GitHub Pages

| Ограничение | Значение |
|-------------|----------|
| Размер репозитория | Рекомендуется < 1 GB |
| Размер сайта | < 1 GB |
| Трафик в месяц | ~100 GB |
| Сборок в час | 10 |

## Примеры использования

### Персональное портфолио

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Мое портфолио</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <header>
        <h1>Иван Иванов</h1>
        <p>Full-Stack Developer</p>
    </header>
    <main>
        <section id="projects">
            <h2>Проекты</h2>
            <!-- Список проектов -->
        </section>
        <section id="contact">
            <h2>Контакты</h2>
            <!-- Контактная информация -->
        </section>
    </main>
</body>
</html>
```

### Документация проекта

```markdown
---
layout: default
title: Документация API
---

# API Documentation

## Endpoints

### GET /api/users

Получение списка пользователей.

**Параметры:**
- `limit` (optional): количество записей
- `offset` (optional): смещение

**Пример ответа:**
```json
{
  "users": [
    {"id": 1, "name": "John"}
  ]
}
```
```

## Полезные советы

1. **Используйте ветку `gh-pages`** для разделения кода и сайта
2. **Настройте `.nojekyll`** файл, если не используете Jekyll
3. **Оптимизируйте изображения** для быстрой загрузки
4. **Используйте относительные пути** для ссылок
5. **Тестируйте локально** с помощью `jekyll serve`

```bash
# Локальный запуск Jekyll
gem install bundler jekyll
bundle exec jekyll serve

# Сайт доступен по адресу http://localhost:4000
```

## Полезные ссылки

- [Официальная документация GitHub Pages](https://docs.github.com/en/pages)
- [Jekyll документация](https://jekyllrb.com/docs/)
- [Supported themes](https://pages.github.com/themes/)
