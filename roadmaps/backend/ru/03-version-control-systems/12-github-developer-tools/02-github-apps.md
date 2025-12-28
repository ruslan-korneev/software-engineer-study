# GitHub Apps

[prev: 01-github-api](./01-github-api.md) | [next: 03-webhooks](./03-webhooks.md)
---

GitHub Apps - это официальный и рекомендуемый способ создания интеграций с GitHub. Они предоставляют более гибкую систему разрешений, лучшую безопасность и могут работать как автономные сервисы.

## Что такое GitHub Apps

GitHub App - это приложение, которое интегрируется с GitHub и может:
- Автоматизировать рабочие процессы (CI/CD, code review)
- Расширять функциональность GitHub
- Реагировать на события в репозиториях
- Выполнять действия от своего имени или от имени пользователя

### Примеры популярных GitHub Apps

- **Dependabot** - автоматическое обновление зависимостей
- **Codecov** - отчеты о покрытии кода
- **Renovate** - управление зависимостями
- **Stale** - автоматическое закрытие неактивных issues
- **WIP** - блокировка PR с "Work in Progress" в заголовке

## Отличие от OAuth Apps

| Характеристика | GitHub Apps | OAuth Apps |
|---------------|-------------|------------|
| Установка | На репозиторий/организацию | На аккаунт пользователя |
| Разрешения | Гранулярные, только нужные | Широкие scopes |
| Rate limits | Выше (5000-15000 req/hr) | Стандартные (5000 req/hr) |
| Webhooks | Встроенные | Требуют отдельной настройки |
| Аутентификация | JWT + Installation token | OAuth token |
| Действия от имени | Приложения или пользователя | Только пользователя |
| Bot аккаунт | Автоматический (`app[bot]`) | Нет |

### Когда использовать GitHub Apps

- Интеграции с CI/CD
- Боты для автоматизации
- Инструменты code review
- Автоматизация репозиториев
- Сервисы, работающие с несколькими репозиториями

### Когда использовать OAuth Apps

- Приложениям нужен только доступ к ресурсам пользователя
- Простые интеграции без webhooks
- Доступ к данным, недоступным для GitHub Apps

## Создание GitHub App

### Шаг 1: Регистрация приложения

1. Перейдите в Settings -> Developer settings -> GitHub Apps
2. Нажмите "New GitHub App"
3. Заполните форму:

```
GitHub App name: My Awesome Bot
Description: Automated code review assistant
Homepage URL: https://myapp.example.com

Callback URL: https://myapp.example.com/callback
(для user-to-server токенов)

Setup URL: https://myapp.example.com/setup
(опционально, после установки)

Webhook URL: https://myapp.example.com/webhook
Webhook secret: <random-secret-string>
```

### Шаг 2: Настройка permissions

Выберите минимально необходимые разрешения:

**Repository permissions:**
- Contents: Read & write (для работы с файлами)
- Issues: Read & write (для создания/изменения issues)
- Pull requests: Read & write (для работы с PR)
- Checks: Read & write (для CI/CD статусов)
- Metadata: Read-only (базовая информация)

**Organization permissions:**
- Members: Read-only (список участников)
- Administration: Read & write (управление org)

**Account permissions:**
- Email addresses: Read-only (email пользователя)

### Шаг 3: Выбор событий (Events)

Подпишитесь на нужные webhook события:

```
[x] Push
[x] Pull request
[x] Issues
[x] Issue comment
[x] Pull request review
[x] Check run
[x] Check suite
```

### Шаг 4: Генерация ключей

После создания приложения:

1. **App ID** - уникальный идентификатор (например, 123456)
2. **Client ID** - для OAuth flow
3. **Client secret** - генерируется один раз
4. **Private key** - скачайте `.pem` файл

```bash
# Сохраните private key безопасно
mv ~/Downloads/my-app.2024-01-15.private-key.pem ./private-key.pem
chmod 600 ./private-key.pem
```

## Аутентификация GitHub Apps

GitHub Apps используют двухуровневую систему аутентификации.

### 1. Аутентификация как приложение (JWT)

Используется для:
- Получения информации о приложении
- Получения installation access token
- Управления установками

```javascript
import jwt from 'jsonwebtoken';
import fs from 'fs';

const privateKey = fs.readFileSync('./private-key.pem', 'utf8');
const appId = process.env.GITHUB_APP_ID;

function generateJWT() {
  const payload = {
    iat: Math.floor(Date.now() / 1000) - 60,  // Issued at (60 сек назад)
    exp: Math.floor(Date.now() / 1000) + 600, // Expires in 10 minutes
    iss: appId                                 // Issuer (App ID)
  };

  return jwt.sign(payload, privateKey, { algorithm: 'RS256' });
}

// Использование
const token = generateJWT();

const response = await fetch('https://api.github.com/app', {
  headers: {
    'Authorization': `Bearer ${token}`,
    'Accept': 'application/vnd.github+json'
  }
});

const appInfo = await response.json();
console.log(appInfo.name); // My Awesome Bot
```

### 2. Аутентификация как installation

Для работы с конкретными репозиториями/организациями:

```javascript
async function getInstallationToken(installationId) {
  const jwtToken = generateJWT();

  const response = await fetch(
    `https://api.github.com/app/installations/${installationId}/access_tokens`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${jwtToken}`,
        'Accept': 'application/vnd.github+json'
      }
    }
  );

  const { token, expires_at } = await response.json();
  return token;
}

// Получить installation ID
async function getInstallationId(owner, repo) {
  const jwtToken = generateJWT();

  const response = await fetch(
    `https://api.github.com/repos/${owner}/${repo}/installation`,
    {
      headers: {
        'Authorization': `Bearer ${jwtToken}`,
        'Accept': 'application/vnd.github+json'
      }
    }
  );

  const { id } = await response.json();
  return id;
}

// Использование
const installationId = await getInstallationId('owner', 'repo');
const installationToken = await getInstallationToken(installationId);

// Теперь можем работать с репозиторием
const response = await fetch('https://api.github.com/repos/owner/repo/issues', {
  headers: {
    'Authorization': `Bearer ${installationToken}`,
    'Accept': 'application/vnd.github+json'
  }
});
```

### 3. User-to-server токены

Для действий от имени пользователя (OAuth flow):

```javascript
// 1. Redirect пользователя
const authUrl = `https://github.com/login/oauth/authorize?client_id=${CLIENT_ID}`;

// 2. Обмен code на token
const response = await fetch('https://github.com/login/oauth/access_token', {
  method: 'POST',
  headers: {
    'Accept': 'application/json',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    client_id: CLIENT_ID,
    client_secret: CLIENT_SECRET,
    code: authorizationCode
  })
});

const { access_token } = await response.json();
```

## Permissions и Events

### Repository Permissions

```yaml
# Чтение/запись содержимого репозитория
contents: write

# Управление issues
issues: write

# Работа с pull requests
pull_requests: write

# CI/CD статусы
checks: write
statuses: write

# Управление workflows
actions: write

# Чтение метаданных (всегда включено)
metadata: read
```

### Subscribe to Events

```yaml
# События для подписки
events:
  - push                    # Новые коммиты
  - pull_request            # Создание/обновление PR
  - pull_request_review     # Review на PR
  - issues                  # Создание/обновление issues
  - issue_comment           # Комментарии к issues/PR
  - check_run               # Запуск/завершение checks
  - check_suite             # Suite checks
  - create                  # Создание веток/тегов
  - delete                  # Удаление веток/тегов
  - release                 # Публикация релизов
  - workflow_run            # Завершение workflow
```

## Установка на репозитории

### Публичная установка

Если приложение публичное, пользователи могут установить его через:
1. GitHub Marketplace
2. Прямая ссылка: `https://github.com/apps/my-app`

### Программная установка

```javascript
// Получить все установки приложения
async function getAllInstallations() {
  const jwtToken = generateJWT();

  const response = await fetch('https://api.github.com/app/installations', {
    headers: {
      'Authorization': `Bearer ${jwtToken}`,
      'Accept': 'application/vnd.github+json'
    }
  });

  return response.json();
}

// Получить репозитории конкретной установки
async function getInstallationRepos(installationId) {
  const token = await getInstallationToken(installationId);

  const response = await fetch(
    'https://api.github.com/installation/repositories',
    {
      headers: {
        'Authorization': `Bearer ${token}`,
        'Accept': 'application/vnd.github+json'
      }
    }
  );

  const { repositories } = await response.json();
  return repositories;
}
```

### Manifest Flow

Для автоматического создания приложения:

```html
<form action="https://github.com/settings/apps/new" method="post">
  <input type="hidden" name="manifest" value='
    {
      "name": "My App",
      "url": "https://myapp.example.com",
      "hook_attributes": {
        "url": "https://myapp.example.com/webhook"
      },
      "redirect_url": "https://myapp.example.com/callback",
      "default_permissions": {
        "issues": "write",
        "contents": "read"
      },
      "default_events": ["issues", "push"]
    }
  '>
  <button type="submit">Create GitHub App</button>
</form>
```

## Probot Framework

Probot - это фреймворк для создания GitHub Apps на Node.js. Он значительно упрощает разработку.

### Установка

```bash
npx create-probot-app my-first-app
cd my-first-app
npm install
```

### Структура проекта

```
my-first-app/
├── src/
│   └── index.ts        # Основная логика
├── test/
│   └── index.test.ts   # Тесты
├── .env                # Конфигурация
├── app.yml             # Manifest приложения
└── package.json
```

### Базовый пример

```typescript
// src/index.ts
import { Probot } from 'probot';

export default (app: Probot) => {
  // Реакция на создание issue
  app.on('issues.opened', async (context) => {
    const issueComment = context.issue({
      body: 'Thanks for opening this issue! We will look into it soon.',
    });
    await context.octokit.issues.createComment(issueComment);
  });

  // Реакция на создание PR
  app.on('pull_request.opened', async (context) => {
    const { pull_request } = context.payload;

    // Проверка заголовка PR
    if (pull_request.title.toLowerCase().startsWith('wip')) {
      await context.octokit.pulls.createReview({
        ...context.pullRequest(),
        event: 'REQUEST_CHANGES',
        body: 'Please remove WIP from the title when ready for review.',
      });
    } else {
      // Добавить label
      await context.octokit.issues.addLabels({
        ...context.issue(),
        labels: ['needs-review'],
      });
    }
  });

  // Реакция на комментарий
  app.on('issue_comment.created', async (context) => {
    const { comment } = context.payload;

    // Команда /assign
    if (comment.body.includes('/assign')) {
      const username = comment.body.split('/assign ')[1]?.trim();
      if (username) {
        await context.octokit.issues.addAssignees({
          ...context.issue(),
          assignees: [username],
        });
      }
    }
  });

  // Логирование всех событий
  app.onAny(async (context) => {
    app.log.info({ event: context.name, action: context.payload.action });
  });
};
```

### Конфигурация (.env)

```bash
# GitHub App credentials
APP_ID=123456
PRIVATE_KEY_PATH=./private-key.pem
# или
PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\n..."

WEBHOOK_SECRET=your-webhook-secret

# Опционально
LOG_LEVEL=debug
PORT=3000
```

### Запуск

```bash
# Development
npm run dev

# Production
npm start
```

### Тестирование с Probot

```typescript
// test/index.test.ts
import nock from 'nock';
import { Probot, ProbotOctokit } from 'probot';
import myApp from '../src';

describe('My Probot App', () => {
  let probot: Probot;

  beforeEach(() => {
    nock.disableNetConnect();
    probot = new Probot({
      appId: 123,
      privateKey: 'test',
      Octokit: ProbotOctokit.defaults({
        retry: { enabled: false },
        throttle: { enabled: false },
      }),
    });
    probot.load(myApp);
  });

  test('creates a comment when an issue is opened', async () => {
    const mock = nock('https://api.github.com')
      .post('/app/installations/2/access_tokens')
      .reply(200, { token: 'test' })
      .post('/repos/owner/repo/issues/1/comments', (body: any) => {
        expect(body.body).toContain('Thanks for opening');
        return true;
      })
      .reply(200);

    await probot.receive({
      name: 'issues',
      payload: {
        action: 'opened',
        issue: { number: 1 },
        repository: { owner: { login: 'owner' }, name: 'repo' },
        installation: { id: 2 },
      },
    });

    expect(mock.isDone()).toBe(true);
  });
});
```

## Примеры использования

### Auto-labeler

Автоматическое добавление labels на основе измененных файлов:

```typescript
app.on('pull_request.opened', async (context) => {
  const { data: files } = await context.octokit.pulls.listFiles({
    ...context.pullRequest(),
  });

  const labels: string[] = [];

  for (const file of files) {
    if (file.filename.startsWith('docs/')) {
      labels.push('documentation');
    }
    if (file.filename.endsWith('.test.ts') || file.filename.endsWith('.spec.ts')) {
      labels.push('tests');
    }
    if (file.filename.includes('package.json')) {
      labels.push('dependencies');
    }
  }

  if (labels.length > 0) {
    await context.octokit.issues.addLabels({
      ...context.issue(),
      labels: [...new Set(labels)],
    });
  }
});
```

### Stale Issues Bot

Автоматическое закрытие неактивных issues:

```typescript
import { Probot } from 'probot';

export default (app: Probot) => {
  // Запуск по расписанию (через GitHub Actions или cron)
  app.on('schedule.repository', async (context) => {
    const staleDate = new Date();
    staleDate.setDate(staleDate.getDate() - 30);

    const { data: issues } = await context.octokit.issues.listForRepo({
      ...context.repo(),
      state: 'open',
      sort: 'updated',
      direction: 'asc',
      per_page: 100,
    });

    for (const issue of issues) {
      const updatedAt = new Date(issue.updated_at);

      if (updatedAt < staleDate) {
        // Добавить label "stale"
        await context.octokit.issues.addLabels({
          ...context.repo(),
          issue_number: issue.number,
          labels: ['stale'],
        });

        // Добавить комментарий
        await context.octokit.issues.createComment({
          ...context.repo(),
          issue_number: issue.number,
          body: 'This issue has been automatically marked as stale due to inactivity. ' +
                'It will be closed in 7 days if no further activity occurs.',
        });
      }
    }
  });
};
```

### PR Size Checker

Проверка размера PR и добавление предупреждения:

```typescript
app.on(['pull_request.opened', 'pull_request.synchronize'], async (context) => {
  const { pull_request } = context.payload;

  const additions = pull_request.additions;
  const deletions = pull_request.deletions;
  const totalChanges = additions + deletions;

  let size: string;
  let message: string;

  if (totalChanges < 50) {
    size = 'XS';
    message = 'Great job keeping this PR small!';
  } else if (totalChanges < 200) {
    size = 'S';
    message = 'Nice, manageable PR size.';
  } else if (totalChanges < 500) {
    size = 'M';
    message = 'Medium-sized PR. Consider splitting if possible.';
  } else if (totalChanges < 1000) {
    size = 'L';
    message = 'Large PR. Please consider breaking it down.';
  } else {
    size = 'XL';
    message = 'Very large PR! This will be hard to review. Please split it.';
  }

  // Добавить label с размером
  await context.octokit.issues.addLabels({
    ...context.issue(),
    labels: [`size/${size}`],
  });

  // Создать check run
  await context.octokit.checks.create({
    ...context.repo(),
    name: 'PR Size',
    head_sha: pull_request.head.sha,
    status: 'completed',
    conclusion: totalChanges > 1000 ? 'failure' : 'success',
    output: {
      title: `PR Size: ${size}`,
      summary: message,
      text: `- Additions: ${additions}\n- Deletions: ${deletions}\n- Total: ${totalChanges}`,
    },
  });
});
```

### Auto-merge Bot

Автоматический merge PR после прохождения всех checks:

```typescript
app.on('check_suite.completed', async (context) => {
  const { check_suite } = context.payload;

  if (check_suite.conclusion !== 'success') {
    return;
  }

  // Найти PR для этого check suite
  const { data: pulls } = await context.octokit.pulls.list({
    ...context.repo(),
    state: 'open',
    head: `${context.repo().owner}:${check_suite.head_branch}`,
  });

  for (const pull of pulls) {
    // Проверить наличие label "auto-merge"
    const hasAutoMergeLabel = pull.labels.some(
      (label) => label.name === 'auto-merge'
    );

    if (!hasAutoMergeLabel) {
      continue;
    }

    // Проверить все статусы
    const { data: status } = await context.octokit.repos.getCombinedStatusForRef({
      ...context.repo(),
      ref: pull.head.sha,
    });

    if (status.state === 'success') {
      await context.octokit.pulls.merge({
        ...context.repo(),
        pull_number: pull.number,
        merge_method: 'squash',
      });

      app.log.info(`Auto-merged PR #${pull.number}`);
    }
  }
});
```

## Развертывание GitHub App

### Варианты хостинга

1. **Vercel** - serverless функции
2. **AWS Lambda** - с API Gateway
3. **Heroku** - классический хостинг
4. **DigitalOcean App Platform**
5. **Self-hosted** - свой сервер

### Пример для Vercel

```typescript
// api/webhook.ts
import { createNodeMiddleware, Probot } from 'probot';
import app from '../src';

const probot = new Probot({
  appId: process.env.APP_ID!,
  privateKey: process.env.PRIVATE_KEY!,
  secret: process.env.WEBHOOK_SECRET!,
});

probot.load(app);

export default createNodeMiddleware(probot.webhooks, {
  path: '/api/webhook',
});
```

### Конфигурация в репозитории

Создайте `.github/config.yml` для настройки приложения:

```yaml
# .github/my-app.yml
labels:
  documentation:
    - 'docs/**'
  tests:
    - '**/*.test.ts'
    - '**/*.spec.ts'
  frontend:
    - 'src/components/**'
    - 'src/pages/**'

autoMerge:
  enabled: true
  requiredApprovals: 2
  mergeMethod: squash

stale:
  daysUntilStale: 30
  daysUntilClose: 7
  exemptLabels:
    - 'pinned'
    - 'security'
```

## Best Practices

### 1. Минимальные permissions

Запрашивайте только те разрешения, которые действительно нужны.

### 2. Обработка ошибок

```typescript
app.on('issues.opened', async (context) => {
  try {
    await context.octokit.issues.createComment({
      ...context.issue(),
      body: 'Welcome!',
    });
  } catch (error) {
    context.log.error(error, 'Failed to create comment');
    // Не пробрасывайте ошибку, чтобы webhook не падал
  }
});
```

### 3. Идемпотентность

Webhook может быть доставлен несколько раз. Убедитесь, что повторная обработка безопасна.

```typescript
app.on('issues.opened', async (context) => {
  const { data: comments } = await context.octokit.issues.listComments({
    ...context.issue(),
  });

  // Проверить, не оставляли ли мы уже комментарий
  const alreadyCommented = comments.some(
    (c) => c.user?.type === 'Bot' && c.body?.includes('Welcome!')
  );

  if (!alreadyCommented) {
    await context.octokit.issues.createComment({
      ...context.issue(),
      body: 'Welcome!',
    });
  }
});
```

### 4. Rate limiting

Используйте встроенные механизмы Octokit:

```typescript
import { Octokit } from '@octokit/rest';
import { throttling } from '@octokit/plugin-throttling';
import { retry } from '@octokit/plugin-retry';

const MyOctokit = Octokit.plugin(throttling, retry);

const octokit = new MyOctokit({
  auth: token,
  throttle: {
    onRateLimit: (retryAfter, options) => {
      console.warn(`Rate limit hit, retrying after ${retryAfter}s`);
      return true;
    },
    onSecondaryRateLimit: (retryAfter, options) => {
      console.warn(`Secondary rate limit hit`);
      return true;
    },
  },
});
```

## Полезные ресурсы

- [GitHub Apps Documentation](https://docs.github.com/en/developers/apps)
- [Probot Documentation](https://probot.github.io/docs/)
- [GitHub App Manifest](https://docs.github.com/en/developers/apps/building-github-apps/creating-a-github-app-from-a-manifest)
- [Octokit Authentication](https://github.com/octokit/authentication-strategies.js)
- [GitHub Marketplace](https://github.com/marketplace)

---
[prev: 01-github-api](./01-github-api.md) | [next: 03-webhooks](./03-webhooks.md)