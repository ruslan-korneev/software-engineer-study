# GitHub API

GitHub предоставляет мощный API для программного взаимодействия с платформой. С его помощью можно автоматизировать рутинные задачи, создавать интеграции и разрабатывать собственные инструменты для работы с репозиториями.

## REST API v3

REST API - это основной способ взаимодействия с GitHub. Базовый URL: `https://api.github.com`

### Основные endpoints

```
GET /users/{username}              # Информация о пользователе
GET /repos/{owner}/{repo}          # Информация о репозитории
GET /repos/{owner}/{repo}/issues   # Список issues
GET /repos/{owner}/{repo}/pulls    # Список pull requests
GET /repos/{owner}/{repo}/commits  # Список коммитов
POST /repos/{owner}/{repo}/issues  # Создание issue
```

### Пример запроса с curl

```bash
# Получить информацию о пользователе
curl -H "Accept: application/vnd.github+json" \
  https://api.github.com/users/octocat

# Получить список репозиториев пользователя
curl -H "Accept: application/vnd.github+json" \
  https://api.github.com/users/octocat/repos

# Создать issue (требует аутентификации)
curl -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer <TOKEN>" \
  https://api.github.com/repos/owner/repo/issues \
  -d '{"title":"Bug report","body":"Description of the bug"}'
```

### Пример ответа

```json
{
  "login": "octocat",
  "id": 1,
  "avatar_url": "https://github.com/images/error/octocat_happy.gif",
  "html_url": "https://github.com/octocat",
  "type": "User",
  "name": "The Octocat",
  "company": "@github",
  "blog": "https://github.blog",
  "location": "San Francisco",
  "public_repos": 8,
  "followers": 20000,
  "following": 0,
  "created_at": "2011-01-25T18:44:36Z"
}
```

## GraphQL API v4

GraphQL API позволяет получать именно те данные, которые вам нужны, одним запросом. Это более эффективно, чем REST, когда нужны связанные данные.

### Endpoint

```
POST https://api.github.com/graphql
```

### Преимущества над REST

- **Гибкость**: запрашиваете только нужные поля
- **Один запрос**: получаете связанные данные за один запрос
- **Строгая типизация**: схема описывает все возможные запросы
- **Интроспекция**: можно исследовать API программно

### Примеры запросов

```graphql
# Получить информацию о пользователе и его репозиториях
query {
  user(login: "octocat") {
    name
    bio
    repositories(first: 5, orderBy: {field: STARGAZERS_COUNT, direction: DESC}) {
      nodes {
        name
        description
        stargazerCount
        forkCount
      }
    }
  }
}
```

```graphql
# Получить issues репозитория
query {
  repository(owner: "facebook", name: "react") {
    issues(first: 10, states: OPEN) {
      totalCount
      nodes {
        title
        createdAt
        author {
          login
        }
        labels(first: 5) {
          nodes {
            name
          }
        }
      }
    }
  }
}
```

### Запрос с curl

```bash
curl -X POST \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  https://api.github.com/graphql \
  -d '{"query": "query { viewer { login name } }"}'
```

### Мутации (изменение данных)

```graphql
mutation {
  createIssue(input: {
    repositoryId: "MDEwOlJlcG9zaXRvcnkxMjM0NTY3ODk="
    title: "Bug report"
    body: "Something is broken"
  }) {
    issue {
      number
      url
    }
  }
}
```

## Аутентификация

### Personal Access Token (PAT)

Самый простой способ аутентификации для личных скриптов.

**Создание токена:**
1. Settings -> Developer settings -> Personal access tokens -> Tokens (classic)
2. Generate new token
3. Выберите нужные scopes (права доступа)
4. Сохраните токен (показывается только один раз!)

**Использование:**
```bash
# В заголовке Authorization
curl -H "Authorization: Bearer ghp_xxxxxxxxxxxx" \
  https://api.github.com/user

# Или в URL (не рекомендуется)
curl https://ghp_xxxxxxxxxxxx@api.github.com/user
```

### Fine-grained Personal Access Token

Более безопасный вариант с гранулярными правами:
- Доступ к конкретным репозиториям
- Точный контроль permissions
- Обязательный срок действия

### OAuth Apps

Для приложений, которым нужен доступ от имени пользователя.

**Процесс OAuth:**
1. Пользователь перенаправляется на GitHub для авторизации
2. GitHub перенаправляет обратно с authorization code
3. Приложение обменивает code на access token
4. Токен используется для API запросов

```javascript
// 1. Redirect пользователя
const authUrl = `https://github.com/login/oauth/authorize?client_id=${CLIENT_ID}&scope=repo,user`;

// 2. После callback, обмен code на token
const response = await fetch('https://github.com/login/oauth/access_token', {
  method: 'POST',
  headers: {
    'Accept': 'application/json',
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    client_id: CLIENT_ID,
    client_secret: CLIENT_SECRET,
    code: authorizationCode,
  }),
});

const { access_token } = await response.json();
```

### GitHub Apps

Предпочтительный способ для интеграций. Подробнее в файле `02-github-apps.md`.

## Rate Limits

GitHub ограничивает количество запросов для защиты от злоупотреблений.

### Лимиты

| Тип | Без аутентификации | С аутентификацией |
|-----|-------------------|-------------------|
| REST API | 60 req/hour | 5000 req/hour |
| GraphQL API | - | 5000 points/hour |
| Search API | 10 req/min | 30 req/min |

### Проверка лимитов

```bash
curl -I https://api.github.com/users/octocat
```

Заголовки ответа:
```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 58
X-RateLimit-Reset: 1234567890
X-RateLimit-Used: 2
```

### Обработка в коде

```javascript
async function makeRequest(url) {
  const response = await fetch(url, {
    headers: { 'Authorization': `Bearer ${TOKEN}` }
  });

  const remaining = response.headers.get('X-RateLimit-Remaining');
  const reset = response.headers.get('X-RateLimit-Reset');

  if (response.status === 403 && remaining === '0') {
    const waitTime = reset * 1000 - Date.now();
    console.log(`Rate limit exceeded. Waiting ${waitTime}ms`);
    await new Promise(resolve => setTimeout(resolve, waitTime));
    return makeRequest(url);
  }

  return response.json();
}
```

### Conditional Requests

Используйте ETags для экономии лимита:

```bash
# Первый запрос - сохраняем ETag
curl -I https://api.github.com/repos/owner/repo
# ETag: "abc123"

# Последующие запросы
curl -H "If-None-Match: \"abc123\"" \
  https://api.github.com/repos/owner/repo
# 304 Not Modified - не расходует лимит
```

## Пагинация

Большие списки возвращаются постранично.

### REST API пагинация

```bash
# Получить первую страницу (30 элементов по умолчанию)
curl "https://api.github.com/repos/owner/repo/issues"

# Указать страницу и размер
curl "https://api.github.com/repos/owner/repo/issues?page=2&per_page=100"
```

**Заголовок Link для навигации:**
```
Link: <https://api.github.com/repos/owner/repo/issues?page=2>; rel="next",
      <https://api.github.com/repos/owner/repo/issues?page=10>; rel="last"
```

### Обработка пагинации в JavaScript

```javascript
async function getAllIssues(owner, repo) {
  const issues = [];
  let page = 1;

  while (true) {
    const response = await fetch(
      `https://api.github.com/repos/${owner}/${repo}/issues?page=${page}&per_page=100`,
      { headers: { 'Authorization': `Bearer ${TOKEN}` } }
    );

    const data = await response.json();

    if (data.length === 0) break;

    issues.push(...data);
    page++;
  }

  return issues;
}
```

### GraphQL пагинация (cursor-based)

```graphql
query($cursor: String) {
  repository(owner: "facebook", name: "react") {
    issues(first: 100, after: $cursor) {
      pageInfo {
        hasNextPage
        endCursor
      }
      nodes {
        title
        number
      }
    }
  }
}
```

## Octokit библиотеки

Octokit - официальные клиентские библиотеки для GitHub API.

### JavaScript/TypeScript (Octokit.js)

```bash
npm install @octokit/rest
# или
npm install octokit  # Все-в-одном пакет
```

```javascript
import { Octokit } from '@octokit/rest';

const octokit = new Octokit({
  auth: 'ghp_xxxxxxxxxxxx'
});

// REST API
const { data: user } = await octokit.users.getByUsername({
  username: 'octocat'
});

console.log(user.name); // The Octocat

// Создание issue
const { data: issue } = await octokit.issues.create({
  owner: 'owner',
  repo: 'repo',
  title: 'New issue',
  body: 'Issue description'
});

// Получение всех issues с автоматической пагинацией
const issues = await octokit.paginate(octokit.issues.listForRepo, {
  owner: 'owner',
  repo: 'repo',
  per_page: 100
});
```

### GraphQL с Octokit

```javascript
import { Octokit } from '@octokit/rest';

const octokit = new Octokit({ auth: TOKEN });

const { repository } = await octokit.graphql(`
  query($owner: String!, $repo: String!) {
    repository(owner: $owner, name: $repo) {
      issues(first: 10, states: OPEN) {
        nodes {
          title
          number
        }
      }
    }
  }
`, {
  owner: 'facebook',
  repo: 'react'
});

console.log(repository.issues.nodes);
```

### Python (PyGithub)

```bash
pip install PyGithub
```

```python
from github import Github

# Аутентификация
g = Github("ghp_xxxxxxxxxxxx")

# Получить пользователя
user = g.get_user("octocat")
print(user.name)  # The Octocat

# Получить репозиторий
repo = g.get_repo("facebook/react")

# Создать issue
issue = repo.create_issue(
    title="Bug report",
    body="Something is broken"
)

# Получить все issues
for issue in repo.get_issues(state="open"):
    print(f"#{issue.number}: {issue.title}")

# Работа с pull requests
pulls = repo.get_pulls(state="open")
for pr in pulls:
    print(f"PR #{pr.number}: {pr.title}")
```

### Другие официальные клиенты

- **Ruby**: `octokit.rb`
- **Go**: `go-github`
- **.NET**: `Octokit.net`

## Практические примеры

### Автоматическое создание release notes

```javascript
import { Octokit } from '@octokit/rest';

const octokit = new Octokit({ auth: TOKEN });

async function generateReleaseNotes(owner, repo, tag) {
  // Получить коммиты между тегами
  const { data: comparison } = await octokit.repos.compareCommits({
    owner,
    repo,
    base: 'v1.0.0',
    head: tag
  });

  // Получить связанные PR
  const prNumbers = comparison.commits
    .map(c => c.commit.message.match(/#(\d+)/)?.[1])
    .filter(Boolean);

  const notes = ['## Changes\n'];

  for (const number of prNumbers) {
    const { data: pr } = await octokit.pulls.get({
      owner,
      repo,
      pull_number: parseInt(number)
    });
    notes.push(`- ${pr.title} (#${number})`);
  }

  return notes.join('\n');
}
```

### Мониторинг репозитория

```javascript
async function getRepoStats(owner, repo) {
  const octokit = new Octokit({ auth: TOKEN });

  const [repoData, issues, pulls, contributors] = await Promise.all([
    octokit.repos.get({ owner, repo }),
    octokit.issues.listForRepo({ owner, repo, state: 'open' }),
    octokit.pulls.list({ owner, repo, state: 'open' }),
    octokit.repos.listContributors({ owner, repo })
  ]);

  return {
    stars: repoData.data.stargazers_count,
    forks: repoData.data.forks_count,
    openIssues: issues.data.length,
    openPRs: pulls.data.length,
    contributors: contributors.data.length
  };
}
```

### CLI инструмент для работы с issues

```javascript
#!/usr/bin/env node
import { Octokit } from '@octokit/rest';
import { program } from 'commander';

const octokit = new Octokit({ auth: process.env.GITHUB_TOKEN });

program
  .command('list <owner> <repo>')
  .description('List open issues')
  .action(async (owner, repo) => {
    const { data } = await octokit.issues.listForRepo({
      owner,
      repo,
      state: 'open'
    });
    data.forEach(issue => {
      console.log(`#${issue.number}: ${issue.title}`);
    });
  });

program
  .command('create <owner> <repo> <title>')
  .option('-b, --body <body>', 'Issue body')
  .action(async (owner, repo, title, options) => {
    const { data } = await octokit.issues.create({
      owner,
      repo,
      title,
      body: options.body
    });
    console.log(`Created issue #${data.number}: ${data.html_url}`);
  });

program.parse();
```

## Best Practices

### 1. Безопасность токенов

```javascript
// Плохо - токен в коде
const token = 'ghp_xxxxxxxxxxxx';

// Хорошо - из переменных окружения
const token = process.env.GITHUB_TOKEN;
```

### 2. Обработка ошибок

```javascript
try {
  const { data } = await octokit.repos.get({ owner, repo });
} catch (error) {
  if (error.status === 404) {
    console.log('Repository not found');
  } else if (error.status === 403) {
    console.log('Access denied or rate limit exceeded');
  } else {
    throw error;
  }
}
```

### 3. Минимальные права токена

Запрашивайте только необходимые scopes:
- `repo` - полный доступ к репозиториям
- `repo:status` - только статусы коммитов
- `public_repo` - только публичные репозитории
- `read:user` - чтение профиля пользователя

### 4. Кэширование

```javascript
const cache = new Map();

async function getCachedUser(username) {
  if (cache.has(username)) {
    return cache.get(username);
  }

  const { data } = await octokit.users.getByUsername({ username });
  cache.set(username, data);

  return data;
}
```

## Полезные ресурсы

- [GitHub REST API Documentation](https://docs.github.com/en/rest)
- [GitHub GraphQL API Documentation](https://docs.github.com/en/graphql)
- [Octokit.js](https://github.com/octokit/octokit.js)
- [GitHub API Explorer (GraphQL)](https://docs.github.com/en/graphql/overview/explorer)
- [PyGithub](https://pygithub.readthedocs.io/)
