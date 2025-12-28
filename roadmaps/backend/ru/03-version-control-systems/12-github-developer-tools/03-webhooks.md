# Webhooks

[prev: 02-github-apps](./02-github-apps.md) | [next: 13-github-security](../13-github-security.md)
---

Webhooks - это механизм уведомлений, который позволяет GitHub отправлять HTTP-запросы на ваш сервер при возникновении определенных событий. Это основа для создания интеграций, ботов и автоматизации рабочих процессов.

## Что такое Webhooks

Webhook - это HTTP callback (обратный вызов). Когда в репозитории происходит событие (push, создание PR, комментарий), GitHub отправляет POST-запрос на указанный URL с информацией о событии.

### Принцип работы

```
1. Событие в репозитории (push, PR, issue)
           ↓
2. GitHub формирует JSON payload
           ↓
3. POST запрос на Webhook URL
           ↓
4. Ваш сервер обрабатывает запрос
           ↓
5. Ответ 200 OK (или 2xx)
```

### Преимущества webhooks

- **Real-time уведомления** - мгновенная реакция на события
- **Экономия ресурсов** - не нужно постоянно опрашивать API (polling)
- **Гибкость** - выбор нужных событий
- **Масштабируемость** - один webhook может обрабатывать тысячи событий

## Настройка webhook в репозитории

### Через веб-интерфейс

1. Перейдите в репозиторий
2. Settings -> Webhooks -> Add webhook
3. Заполните форму:

```
Payload URL: https://your-server.com/webhook
Content type: application/json
Secret: your-secret-token (опционально, но рекомендуется)

Which events would you like to trigger this webhook?
○ Just the push event
○ Send me everything
● Let me select individual events
  [x] Push
  [x] Pull requests
  [x] Issues
  [x] Issue comments
```

### Через GitHub API

```bash
curl -X POST \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/owner/repo/hooks \
  -d '{
    "name": "web",
    "active": true,
    "events": ["push", "pull_request", "issues"],
    "config": {
      "url": "https://your-server.com/webhook",
      "content_type": "json",
      "secret": "your-secret-token",
      "insecure_ssl": "0"
    }
  }'
```

### Через Octokit

```javascript
const { Octokit } = require('@octokit/rest');

const octokit = new Octokit({ auth: TOKEN });

async function createWebhook(owner, repo) {
  const { data } = await octokit.repos.createWebhook({
    owner,
    repo,
    config: {
      url: 'https://your-server.com/webhook',
      content_type: 'json',
      secret: process.env.WEBHOOK_SECRET,
    },
    events: ['push', 'pull_request', 'issues'],
    active: true,
  });

  console.log(`Webhook created: ${data.id}`);
  return data;
}
```

## События (Events)

GitHub поддерживает множество событий. Вот основные категории:

### Repository events

| Событие | Описание |
|---------|----------|
| `push` | Push коммитов в репозиторий |
| `create` | Создание ветки или тега |
| `delete` | Удаление ветки или тега |
| `fork` | Fork репозитория |
| `release` | Публикация релиза |
| `repository` | Создание/удаление/изменение репозитория |

### Pull request events

| Событие | Описание |
|---------|----------|
| `pull_request` | Создание/обновление/закрытие PR |
| `pull_request_review` | Review на PR |
| `pull_request_review_comment` | Комментарий в review |
| `pull_request_review_thread` | Thread в review |

### Issue events

| Событие | Описание |
|---------|----------|
| `issues` | Создание/обновление/закрытие issue |
| `issue_comment` | Комментарий к issue или PR |
| `label` | Создание/удаление label |
| `milestone` | Создание/обновление milestone |

### CI/CD events

| Событие | Описание |
|---------|----------|
| `check_run` | Запуск/завершение check |
| `check_suite` | Suite checks |
| `status` | Изменение статуса коммита |
| `workflow_run` | Запуск/завершение workflow |
| `workflow_job` | Запуск/завершение job |
| `deployment` | Создание deployment |
| `deployment_status` | Статус deployment |

### Collaboration events

| Событие | Описание |
|---------|----------|
| `member` | Добавление/удаление collaborator |
| `team_add` | Добавление репозитория в team |
| `organization` | События организации |
| `membership` | Изменение членства в team |

### Actions для событий

Многие события имеют action, указывающий тип действия:

```json
{
  "action": "opened",     // opened, closed, reopened
  "action": "created",    // created, edited, deleted
  "action": "synchronize" // для PR при новых коммитах
}
```

## Payload структура

Каждый webhook содержит JSON payload с информацией о событии.

### Общие поля

```json
{
  "action": "opened",
  "sender": {
    "login": "octocat",
    "id": 1,
    "type": "User"
  },
  "repository": {
    "id": 123456,
    "name": "repo-name",
    "full_name": "owner/repo-name",
    "owner": {
      "login": "owner"
    },
    "private": false,
    "html_url": "https://github.com/owner/repo-name"
  },
  "organization": {
    "login": "org-name"
  },
  "installation": {
    "id": 789
  }
}
```

### Push event

```json
{
  "ref": "refs/heads/main",
  "before": "abc123...",
  "after": "def456...",
  "created": false,
  "deleted": false,
  "forced": false,
  "base_ref": null,
  "compare": "https://github.com/owner/repo/compare/abc123...def456",
  "commits": [
    {
      "id": "def456...",
      "message": "Update README",
      "timestamp": "2024-01-15T10:30:00Z",
      "author": {
        "name": "John Doe",
        "email": "john@example.com",
        "username": "johndoe"
      },
      "added": ["new-file.txt"],
      "removed": [],
      "modified": ["README.md"]
    }
  ],
  "head_commit": {
    "id": "def456...",
    "message": "Update README"
  },
  "pusher": {
    "name": "johndoe",
    "email": "john@example.com"
  }
}
```

### Pull request event

```json
{
  "action": "opened",
  "number": 42,
  "pull_request": {
    "id": 123,
    "number": 42,
    "state": "open",
    "title": "Add new feature",
    "body": "Description of changes",
    "user": {
      "login": "contributor"
    },
    "head": {
      "ref": "feature-branch",
      "sha": "abc123...",
      "repo": {
        "full_name": "contributor/repo"
      }
    },
    "base": {
      "ref": "main",
      "sha": "def456...",
      "repo": {
        "full_name": "owner/repo"
      }
    },
    "merged": false,
    "mergeable": true,
    "mergeable_state": "clean",
    "additions": 50,
    "deletions": 10,
    "changed_files": 3,
    "commits": 2,
    "html_url": "https://github.com/owner/repo/pull/42",
    "created_at": "2024-01-15T10:00:00Z",
    "updated_at": "2024-01-15T10:30:00Z"
  }
}
```

### Issue event

```json
{
  "action": "opened",
  "issue": {
    "id": 456,
    "number": 10,
    "title": "Bug report",
    "body": "Description of the bug",
    "state": "open",
    "user": {
      "login": "reporter"
    },
    "labels": [
      {
        "name": "bug",
        "color": "d73a4a"
      }
    ],
    "assignees": [
      {
        "login": "developer"
      }
    ],
    "milestone": {
      "title": "v1.0"
    },
    "html_url": "https://github.com/owner/repo/issues/10",
    "created_at": "2024-01-15T10:00:00Z"
  }
}
```

### Issue comment event

```json
{
  "action": "created",
  "issue": {
    "number": 10,
    "title": "Bug report"
  },
  "comment": {
    "id": 789,
    "body": "Thanks for reporting! I'll look into it.",
    "user": {
      "login": "maintainer"
    },
    "html_url": "https://github.com/owner/repo/issues/10#issuecomment-789",
    "created_at": "2024-01-15T11:00:00Z"
  }
}
```

## Валидация подписи (Secret)

Для безопасности webhook следует использовать secret и проверять подпись запросов.

### Как работает подпись

1. При настройке webhook указывается secret
2. GitHub вычисляет HMAC-SHA256 от payload с использованием secret
3. Подпись отправляется в заголовке `X-Hub-Signature-256`
4. Ваш сервер проверяет подпись

### Заголовки webhook запроса

```
X-GitHub-Event: push
X-GitHub-Delivery: abc123-uuid
X-Hub-Signature: sha1=...
X-Hub-Signature-256: sha256=...
Content-Type: application/json
User-Agent: GitHub-Hookshot/abc123
```

### Валидация в Node.js

```javascript
const crypto = require('crypto');

function verifySignature(payload, signature, secret) {
  const hmac = crypto.createHmac('sha256', secret);
  const digest = 'sha256=' + hmac.update(payload).digest('hex');

  // Используем timingSafeEqual для защиты от timing attacks
  return crypto.timingSafeEqual(
    Buffer.from(digest),
    Buffer.from(signature)
  );
}

// Express middleware
function webhookMiddleware(req, res, next) {
  const signature = req.headers['x-hub-signature-256'];
  const payload = JSON.stringify(req.body);

  if (!signature) {
    return res.status(401).send('Missing signature');
  }

  if (!verifySignature(payload, signature, process.env.WEBHOOK_SECRET)) {
    return res.status(401).send('Invalid signature');
  }

  next();
}
```

### Валидация в Python

```python
import hmac
import hashlib

def verify_signature(payload: bytes, signature: str, secret: str) -> bool:
    expected = 'sha256=' + hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(expected, signature)

# Flask пример
from flask import Flask, request, abort

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def webhook():
    signature = request.headers.get('X-Hub-Signature-256')

    if not signature:
        abort(401, 'Missing signature')

    if not verify_signature(request.data, signature, WEBHOOK_SECRET):
        abort(401, 'Invalid signature')

    event = request.headers.get('X-GitHub-Event')
    payload = request.json

    # Обработка события
    handle_event(event, payload)

    return 'OK', 200
```

### Валидация в Go

```go
package main

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "io"
    "net/http"
    "strings"
)

func verifySignature(payload []byte, signature, secret string) bool {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(payload)
    expected := "sha256=" + hex.EncodeToString(mac.Sum(nil))

    return hmac.Equal([]byte(expected), []byte(signature))
}

func webhookHandler(w http.ResponseWriter, r *http.Request) {
    signature := r.Header.Get("X-Hub-Signature-256")

    payload, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "Error reading body", http.StatusBadRequest)
        return
    }

    if !verifySignature(payload, signature, webhookSecret) {
        http.Error(w, "Invalid signature", http.StatusUnauthorized)
        return
    }

    event := r.Header.Get("X-GitHub-Event")
    // Обработка события
    handleEvent(event, payload)

    w.WriteHeader(http.StatusOK)
}
```

## Обработка webhook на сервере

### Express.js пример

```javascript
const express = require('express');
const crypto = require('crypto');

const app = express();
app.use(express.json());

// Middleware для валидации
function validateWebhook(req, res, next) {
  const signature = req.headers['x-hub-signature-256'];
  const payload = JSON.stringify(req.body);

  const hmac = crypto.createHmac('sha256', process.env.WEBHOOK_SECRET);
  const expected = 'sha256=' + hmac.update(payload).digest('hex');

  if (!crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature || ''))) {
    return res.status(401).send('Invalid signature');
  }

  next();
}

// Webhook endpoint
app.post('/webhook', validateWebhook, async (req, res) => {
  const event = req.headers['x-github-event'];
  const action = req.body.action;
  const payload = req.body;

  console.log(`Received ${event}.${action}`);

  try {
    switch (event) {
      case 'push':
        await handlePush(payload);
        break;
      case 'pull_request':
        await handlePullRequest(payload);
        break;
      case 'issues':
        await handleIssue(payload);
        break;
      case 'issue_comment':
        await handleComment(payload);
        break;
      default:
        console.log(`Unhandled event: ${event}`);
    }

    res.status(200).send('OK');
  } catch (error) {
    console.error('Error handling webhook:', error);
    res.status(500).send('Internal error');
  }
});

// Handlers
async function handlePush(payload) {
  const { ref, commits, pusher, repository } = payload;
  const branch = ref.replace('refs/heads/', '');

  console.log(`Push to ${repository.full_name}/${branch} by ${pusher.name}`);
  console.log(`Commits: ${commits.length}`);

  for (const commit of commits) {
    console.log(`- ${commit.id.slice(0, 7)}: ${commit.message}`);
  }

  // Автоматический деплой при push в main
  if (branch === 'main') {
    await triggerDeploy(repository.full_name);
  }
}

async function handlePullRequest(payload) {
  const { action, pull_request, repository } = payload;

  console.log(`PR #${pull_request.number} ${action} in ${repository.full_name}`);

  switch (action) {
    case 'opened':
      // Приветственное сообщение
      await postComment(
        repository.full_name,
        pull_request.number,
        'Thanks for the PR! A maintainer will review it soon.'
      );
      // Автоматическое добавление labels
      await addLabels(repository.full_name, pull_request.number);
      break;

    case 'closed':
      if (pull_request.merged) {
        console.log('PR was merged!');
      }
      break;

    case 'synchronize':
      console.log('New commits pushed to PR');
      break;
  }
}

async function handleIssue(payload) {
  const { action, issue, repository } = payload;

  if (action === 'opened') {
    // Проверка на дубликаты
    const duplicates = await findDuplicateIssues(
      repository.full_name,
      issue.title
    );

    if (duplicates.length > 0) {
      await postComment(
        repository.full_name,
        issue.number,
        `This might be a duplicate of: ${duplicates.map(d => `#${d}`).join(', ')}`
      );
    }
  }
}

async function handleComment(payload) {
  const { action, comment, issue, repository } = payload;

  if (action !== 'created') return;

  const body = comment.body.toLowerCase();

  // Slash commands
  if (body.includes('/assign')) {
    const match = body.match(/\/assign\s+@?(\w+)/);
    if (match) {
      await assignUser(repository.full_name, issue.number, match[1]);
    }
  }

  if (body.includes('/label')) {
    const match = body.match(/\/label\s+(\w+)/);
    if (match) {
      await addLabel(repository.full_name, issue.number, match[1]);
    }
  }

  if (body.includes('/close')) {
    await closeIssue(repository.full_name, issue.number);
  }
}

app.listen(3000, () => {
  console.log('Webhook server running on port 3000');
});
```

### FastAPI (Python) пример

```python
from fastapi import FastAPI, Request, HTTPException, Header
import hmac
import hashlib
import os

app = FastAPI()

WEBHOOK_SECRET = os.environ.get('WEBHOOK_SECRET')

def verify_signature(payload: bytes, signature: str) -> bool:
    if not WEBHOOK_SECRET:
        return True  # Skip verification in development

    expected = 'sha256=' + hmac.new(
        WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(expected, signature)

@app.post('/webhook')
async def webhook(
    request: Request,
    x_github_event: str = Header(...),
    x_hub_signature_256: str = Header(...)
):
    payload = await request.body()

    if not verify_signature(payload, x_hub_signature_256):
        raise HTTPException(status_code=401, detail='Invalid signature')

    data = await request.json()
    action = data.get('action')

    print(f'Received {x_github_event}.{action}')

    handlers = {
        'push': handle_push,
        'pull_request': handle_pull_request,
        'issues': handle_issues,
        'issue_comment': handle_comment,
    }

    handler = handlers.get(x_github_event)
    if handler:
        await handler(data)

    return {'status': 'ok'}

async def handle_push(data: dict):
    ref = data['ref']
    commits = data['commits']
    repository = data['repository']['full_name']

    branch = ref.replace('refs/heads/', '')
    print(f'Push to {repository}/{branch}')
    print(f'Commits: {len(commits)}')

    for commit in commits:
        print(f"- {commit['id'][:7]}: {commit['message']}")

async def handle_pull_request(data: dict):
    action = data['action']
    pr = data['pull_request']
    repo = data['repository']['full_name']

    print(f"PR #{pr['number']} {action} in {repo}")

    if action == 'opened':
        # Добавить labels на основе файлов
        await auto_label_pr(repo, pr['number'])
    elif action == 'closed' and pr['merged']:
        print('PR merged!')

async def handle_issues(data: dict):
    action = data['action']
    issue = data['issue']
    repo = data['repository']['full_name']

    print(f"Issue #{issue['number']} {action} in {repo}")

async def handle_comment(data: dict):
    action = data['action']
    comment = data['comment']
    issue = data['issue']

    if action == 'created':
        body = comment['body'].lower()

        # Handle bot commands
        if '/help' in body:
            await post_help_message(issue['number'])

if __name__ == '__main__':
    import uvicorn
    uvicorn.run(app, host='0.0.0.0', port=3000)
```

## Примеры использования

### CI/CD триггер

```javascript
async function handlePush(payload) {
  const { ref, repository, head_commit } = payload;
  const branch = ref.replace('refs/heads/', '');

  // Только для main/master
  if (!['main', 'master'].includes(branch)) {
    return;
  }

  // Пропустить [skip ci]
  if (head_commit.message.includes('[skip ci]')) {
    console.log('Skipping CI');
    return;
  }

  // Запустить билд
  const buildResponse = await fetch('https://ci.example.com/api/builds', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${CI_TOKEN}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      repository: repository.clone_url,
      branch,
      commit: head_commit.id,
    }),
  });

  const build = await buildResponse.json();

  // Обновить статус коммита
  await octokit.repos.createCommitStatus({
    owner: repository.owner.login,
    repo: repository.name,
    sha: head_commit.id,
    state: 'pending',
    target_url: build.url,
    description: 'Build started',
    context: 'ci/build',
  });
}
```

### Slack уведомления

```javascript
async function notifySlack(event, payload) {
  const messages = {
    'push': () => {
      const { pusher, repository, commits, compare } = payload;
      return {
        text: `New push to ${repository.full_name}`,
        blocks: [
          {
            type: 'section',
            text: {
              type: 'mrkdwn',
              text: `*${pusher.name}* pushed ${commits.length} commit(s) to *${repository.full_name}*`,
            },
          },
          {
            type: 'section',
            text: {
              type: 'mrkdwn',
              text: commits.slice(0, 5).map(c =>
                `\`${c.id.slice(0, 7)}\` ${c.message.split('\n')[0]}`
              ).join('\n'),
            },
          },
          {
            type: 'actions',
            elements: [
              {
                type: 'button',
                text: { type: 'plain_text', text: 'View changes' },
                url: compare,
              },
            ],
          },
        ],
      };
    },

    'pull_request': () => {
      const { action, pull_request, sender } = payload;
      if (!['opened', 'closed', 'merged'].includes(action)) return null;

      const emoji = action === 'opened' ? ':new:' :
                    pull_request.merged ? ':merged:' : ':closed:';

      return {
        text: `PR ${action}: ${pull_request.title}`,
        blocks: [
          {
            type: 'section',
            text: {
              type: 'mrkdwn',
              text: `${emoji} *${sender.login}* ${action} PR: *${pull_request.title}*`,
            },
            accessory: {
              type: 'button',
              text: { type: 'plain_text', text: 'View PR' },
              url: pull_request.html_url,
            },
          },
        ],
      };
    },
  };

  const getMessage = messages[event];
  if (!getMessage) return;

  const message = getMessage();
  if (!message) return;

  await fetch(SLACK_WEBHOOK_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(message),
  });
}
```

### Auto-labeler на основе файлов

```javascript
async function autoLabelPR(payload) {
  const { pull_request, repository } = payload;

  // Получить измененные файлы
  const { data: files } = await octokit.pulls.listFiles({
    owner: repository.owner.login,
    repo: repository.name,
    pull_number: pull_request.number,
  });

  const labels = new Set();

  const rules = {
    'docs/**': 'documentation',
    '*.md': 'documentation',
    'src/**/*.test.*': 'tests',
    'tests/**': 'tests',
    'package.json': 'dependencies',
    'package-lock.json': 'dependencies',
    '.github/**': 'ci',
    'src/components/**': 'frontend',
    'src/api/**': 'backend',
  };

  for (const file of files) {
    for (const [pattern, label] of Object.entries(rules)) {
      if (minimatch(file.filename, pattern)) {
        labels.add(label);
      }
    }
  }

  // Размер PR
  const totalChanges = pull_request.additions + pull_request.deletions;
  if (totalChanges < 50) labels.add('size/S');
  else if (totalChanges < 200) labels.add('size/M');
  else if (totalChanges < 500) labels.add('size/L');
  else labels.add('size/XL');

  if (labels.size > 0) {
    await octokit.issues.addLabels({
      owner: repository.owner.login,
      repo: repository.name,
      issue_number: pull_request.number,
      labels: [...labels],
    });
  }
}
```

### Автоматический reviewer assignment

```javascript
async function assignReviewers(payload) {
  const { pull_request, repository } = payload;

  // Конфигурация команд
  const teams = {
    'frontend': ['alice', 'bob'],
    'backend': ['charlie', 'dave'],
    'devops': ['eve'],
  };

  // Определить нужных reviewers на основе файлов
  const { data: files } = await octokit.pulls.listFiles({
    owner: repository.owner.login,
    repo: repository.name,
    pull_number: pull_request.number,
  });

  const neededTeams = new Set();

  for (const file of files) {
    if (file.filename.startsWith('src/components/')) {
      neededTeams.add('frontend');
    }
    if (file.filename.startsWith('src/api/') || file.filename.startsWith('server/')) {
      neededTeams.add('backend');
    }
    if (file.filename.includes('Dockerfile') || file.filename.startsWith('.github/')) {
      neededTeams.add('devops');
    }
  }

  // Выбрать reviewers (исключая автора PR)
  const reviewers = [];
  for (const team of neededTeams) {
    const teamMembers = teams[team].filter(m => m !== pull_request.user.login);
    if (teamMembers.length > 0) {
      // Выбрать случайного члена команды
      reviewers.push(teamMembers[Math.floor(Math.random() * teamMembers.length)]);
    }
  }

  if (reviewers.length > 0) {
    await octokit.pulls.requestReviewers({
      owner: repository.owner.login,
      repo: repository.name,
      pull_number: pull_request.number,
      reviewers: [...new Set(reviewers)],
    });
  }
}
```

## Отладка webhooks

### Локальная разработка с ngrok

```bash
# Установка ngrok
npm install -g ngrok

# Запуск туннеля
ngrok http 3000

# Получите URL типа https://abc123.ngrok.io
# Используйте его как Payload URL в настройках webhook
```

### Просмотр доставленных webhooks

В настройках webhook на GitHub:
1. Recent Deliveries - список последних запросов
2. Redeliver - повторная отправка для тестирования
3. Response - ответ вашего сервера

### Логирование

```javascript
app.post('/webhook', (req, res) => {
  const delivery = req.headers['x-github-delivery'];
  const event = req.headers['x-github-event'];

  console.log(JSON.stringify({
    timestamp: new Date().toISOString(),
    delivery,
    event,
    action: req.body.action,
    repository: req.body.repository?.full_name,
    sender: req.body.sender?.login,
  }));

  // ... обработка
});
```

### Webhook Tester

GitHub предоставляет инструмент для тестирования:
1. Перейдите в настройки webhook
2. "Test" - отправить тестовый ping event
3. Просмотрите результат в Recent Deliveries

## Best Practices

### 1. Быстрый ответ

GitHub ожидает ответ в течение 10 секунд. Для долгих операций используйте очередь:

```javascript
const Queue = require('bull');
const webhookQueue = new Queue('webhooks');

app.post('/webhook', async (req, res) => {
  // Быстро поставить в очередь
  await webhookQueue.add({
    event: req.headers['x-github-event'],
    payload: req.body,
  });

  res.status(202).send('Accepted');
});

// Обработчик очереди
webhookQueue.process(async (job) => {
  const { event, payload } = job.data;
  await processWebhook(event, payload);
});
```

### 2. Идемпотентность

Webhook может быть доставлен несколько раз:

```javascript
const processedDeliveries = new Set();

app.post('/webhook', (req, res) => {
  const deliveryId = req.headers['x-github-delivery'];

  if (processedDeliveries.has(deliveryId)) {
    return res.status(200).send('Already processed');
  }

  processedDeliveries.add(deliveryId);

  // Обработка...
});
```

### 3. Обработка ошибок

```javascript
app.post('/webhook', async (req, res) => {
  try {
    await handleWebhook(req);
    res.status(200).send('OK');
  } catch (error) {
    console.error('Webhook error:', error);

    // Вернуть 200, чтобы GitHub не считал это ошибкой
    // (если не хотите retry)
    res.status(200).send('Error logged');

    // Или вернуть 500 для retry
    // res.status(500).send('Internal error');
  }
});
```

### 4. Безопасность

- Всегда используйте HTTPS
- Всегда проверяйте подпись
- Не доверяйте данным из payload без проверки
- Используйте rate limiting

```javascript
const rateLimit = require('express-rate-limit');

const webhookLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 минута
  max: 100, // 100 запросов
  message: 'Too many webhooks',
});

app.post('/webhook', webhookLimiter, validateWebhook, handleWebhook);
```

## Полезные ресурсы

- [GitHub Webhooks Documentation](https://docs.github.com/en/developers/webhooks-and-events/webhooks)
- [Webhook Events and Payloads](https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads)
- [Securing Your Webhooks](https://docs.github.com/en/developers/webhooks-and-events/webhooks/securing-your-webhooks)
- [ngrok](https://ngrok.com/) - для локальной разработки
- [smee.io](https://smee.io/) - webhook proxy для разработки

---
[prev: 02-github-apps](./02-github-apps.md) | [next: 13-github-security](../13-github-security.md)