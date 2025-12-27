# Подписки в GraphQL (Subscriptions)

## Что такое Subscriptions?

**Subscriptions** — это механизм получения данных в реальном времени в GraphQL. В отличие от Query и Mutation, которые следуют модели запрос-ответ, подписки устанавливают постоянное соединение между клиентом и сервером, позволяя серверу отправлять данные клиенту при возникновении событий.

## Сравнение подходов real-time

| Подход | Описание | Преимущества | Недостатки |
|--------|----------|--------------|------------|
| Polling | Периодические запросы | Простота | Нагрузка, задержка |
| Long Polling | HTTP с ожиданием | Совместимость | Накладные расходы |
| SSE | Server-Sent Events | Простота, HTTP | Только сервер → клиент |
| WebSocket | Двунаправленное соединение | Низкая задержка | Сложность |
| **GraphQL Subscriptions** | WebSocket + GraphQL | Типизация, гибкость | Инфраструктура |

## Базовая структура

### Схема

```graphql
type Subscription {
  # Простая подписка
  postCreated: Post!

  # Подписка с аргументами
  commentAdded(postId: ID!): Comment!

  # Подписка на изменения пользователя
  userUpdated(userId: ID!): User!

  # Подписка на уведомления
  notificationReceived: Notification!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
}

type Comment {
  id: ID!
  text: String!
  author: User!
  post: Post!
}

type Notification {
  id: ID!
  type: NotificationType!
  message: String!
  createdAt: DateTime!
}

enum NotificationType {
  NEW_FOLLOWER
  NEW_COMMENT
  NEW_LIKE
  MENTION
}
```

### Запрос подписки

```graphql
subscription OnPostCreated {
  postCreated {
    id
    title
    content
    author {
      id
      name
    }
  }
}

subscription OnCommentAdded($postId: ID!) {
  commentAdded(postId: $postId) {
    id
    text
    author {
      name
      avatar
    }
    createdAt
  }
}
```

## Реализация на сервере

### Apollo Server с graphql-ws

```javascript
const { ApolloServer } = require('@apollo/server');
const { expressMiddleware } = require('@apollo/server/express4');
const { createServer } = require('http');
const { WebSocketServer } = require('ws');
const { useServer } = require('graphql-ws/lib/use/ws');
const { makeExecutableSchema } = require('@graphql-tools/schema');
const { PubSub } = require('graphql-subscriptions');
const express = require('express');

const pubsub = new PubSub();

// Константы для событий
const EVENTS = {
  POST_CREATED: 'POST_CREATED',
  COMMENT_ADDED: 'COMMENT_ADDED',
  USER_UPDATED: 'USER_UPDATED',
  NOTIFICATION: 'NOTIFICATION'
};

const typeDefs = `
  type Query {
    posts: [Post!]!
  }

  type Mutation {
    createPost(input: CreatePostInput!): Post!
    addComment(input: AddCommentInput!): Comment!
  }

  type Subscription {
    postCreated: Post!
    commentAdded(postId: ID!): Comment!
  }

  type Post {
    id: ID!
    title: String!
    content: String!
  }

  type Comment {
    id: ID!
    text: String!
    postId: ID!
  }

  input CreatePostInput {
    title: String!
    content: String!
  }

  input AddCommentInput {
    postId: ID!
    text: String!
  }
`;

const resolvers = {
  Query: {
    posts: () => []
  },

  Mutation: {
    createPost: async (_, { input }) => {
      const post = {
        id: Date.now().toString(),
        ...input
      };

      // Публикация события
      pubsub.publish(EVENTS.POST_CREATED, { postCreated: post });

      return post;
    },

    addComment: async (_, { input }) => {
      const comment = {
        id: Date.now().toString(),
        ...input
      };

      // Публикация события с фильтром по postId
      pubsub.publish(`${EVENTS.COMMENT_ADDED}.${input.postId}`, {
        commentAdded: comment
      });

      return comment;
    }
  },

  Subscription: {
    postCreated: {
      subscribe: () => pubsub.asyncIterator([EVENTS.POST_CREATED])
    },

    commentAdded: {
      subscribe: (_, { postId }) => {
        return pubsub.asyncIterator([`${EVENTS.COMMENT_ADDED}.${postId}`]);
      }
    }
  }
};

async function startServer() {
  const app = express();
  const httpServer = createServer(app);

  const schema = makeExecutableSchema({ typeDefs, resolvers });

  // WebSocket сервер
  const wsServer = new WebSocketServer({
    server: httpServer,
    path: '/graphql'
  });

  const serverCleanup = useServer({ schema }, wsServer);

  const server = new ApolloServer({
    schema,
    plugins: [
      {
        async serverWillStart() {
          return {
            async drainServer() {
              await serverCleanup.dispose();
            }
          };
        }
      }
    ]
  });

  await server.start();

  app.use('/graphql', express.json(), expressMiddleware(server));

  httpServer.listen(4000, () => {
    console.log('Server running on http://localhost:4000/graphql');
  });
}

startServer();
```

### Python (Strawberry)

```python
import strawberry
from strawberry.subscriptions import Subscription
from typing import AsyncGenerator
import asyncio

# Простой in-memory pub/sub
class PubSub:
    def __init__(self):
        self.subscribers = {}

    def subscribe(self, event: str):
        if event not in self.subscribers:
            self.subscribers[event] = []
        queue = asyncio.Queue()
        self.subscribers[event].append(queue)
        return queue

    async def publish(self, event: str, data):
        if event in self.subscribers:
            for queue in self.subscribers[event]:
                await queue.put(data)

pubsub = PubSub()

@strawberry.type
class Post:
    id: str
    title: str
    content: str

@strawberry.type
class Query:
    @strawberry.field
    def posts(self) -> list[Post]:
        return []

@strawberry.type
class Mutation:
    @strawberry.mutation
    async def create_post(self, title: str, content: str) -> Post:
        post = Post(id=str(id(title)), title=title, content=content)
        await pubsub.publish("POST_CREATED", post)
        return post

@strawberry.type
class Subscription:
    @strawberry.subscription
    async def post_created(self) -> AsyncGenerator[Post, None]:
        queue = pubsub.subscribe("POST_CREATED")
        while True:
            post = await queue.get()
            yield post

schema = strawberry.Schema(
    query=Query,
    mutation=Mutation,
    subscription=Subscription
)
```

## Фильтрация событий

### withFilter

```javascript
const { withFilter } = require('graphql-subscriptions');

const resolvers = {
  Subscription: {
    // Фильтрация по аргументу
    commentAdded: {
      subscribe: withFilter(
        () => pubsub.asyncIterator([EVENTS.COMMENT_ADDED]),
        (payload, variables) => {
          // Возвращает true, если событие соответствует фильтру
          return payload.commentAdded.postId === variables.postId;
        }
      )
    },

    // Фильтрация по контексту (например, по пользователю)
    notificationReceived: {
      subscribe: withFilter(
        () => pubsub.asyncIterator([EVENTS.NOTIFICATION]),
        (payload, variables, context) => {
          return payload.notificationReceived.userId === context.userId;
        }
      )
    },

    // Множественные условия фильтрации
    messageReceived: {
      subscribe: withFilter(
        () => pubsub.asyncIterator([EVENTS.MESSAGE]),
        (payload, variables, context) => {
          const { message } = payload.messageReceived;
          const { chatId } = variables;
          const { userId } = context;

          // Сообщение в нужном чате И пользователь участник
          return message.chatId === chatId &&
                 (message.senderId === userId || message.receiverId === userId);
        }
      )
    }
  }
};
```

## Клиентская интеграция

### Apollo Client

```javascript
import {
  ApolloClient,
  InMemoryCache,
  split,
  HttpLink
} from '@apollo/client';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { createClient } from 'graphql-ws';
import { getMainDefinition } from '@apollo/client/utilities';

// HTTP link для queries и mutations
const httpLink = new HttpLink({
  uri: 'http://localhost:4000/graphql'
});

// WebSocket link для subscriptions
const wsLink = new GraphQLWsLink(
  createClient({
    url: 'ws://localhost:4000/graphql',
    connectionParams: {
      authToken: localStorage.getItem('token')
    }
  })
);

// Разделение трафика
const splitLink = split(
  ({ query }) => {
    const definition = getMainDefinition(query);
    return (
      definition.kind === 'OperationDefinition' &&
      definition.operation === 'subscription'
    );
  },
  wsLink,
  httpLink
);

const client = new ApolloClient({
  link: splitLink,
  cache: new InMemoryCache()
});
```

### React хук useSubscription

```javascript
import { useSubscription, gql } from '@apollo/client';

const POST_CREATED_SUBSCRIPTION = gql`
  subscription OnPostCreated {
    postCreated {
      id
      title
      content
      author {
        name
      }
    }
  }
`;

function NewPostsNotification() {
  const { data, loading, error } = useSubscription(POST_CREATED_SUBSCRIPTION);

  if (loading) return <p>Ожидание новых постов...</p>;
  if (error) return <p>Ошибка: {error.message}</p>;

  return (
    <div className="notification">
      Новый пост: {data.postCreated.title}
    </div>
  );
}
```

### subscribeToMore для обновления списка

```javascript
import { useQuery, gql } from '@apollo/client';
import { useEffect } from 'react';

const GET_POSTS = gql`
  query GetPosts {
    posts {
      id
      title
      content
    }
  }
`;

const POST_CREATED = gql`
  subscription OnPostCreated {
    postCreated {
      id
      title
      content
    }
  }
`;

function PostsList() {
  const { data, loading, subscribeToMore } = useQuery(GET_POSTS);

  useEffect(() => {
    const unsubscribe = subscribeToMore({
      document: POST_CREATED,
      updateQuery: (prev, { subscriptionData }) => {
        if (!subscriptionData.data) return prev;

        const newPost = subscriptionData.data.postCreated;

        return {
          ...prev,
          posts: [newPost, ...prev.posts]
        };
      }
    });

    return () => unsubscribe();
  }, [subscribeToMore]);

  if (loading) return <p>Загрузка...</p>;

  return (
    <ul>
      {data.posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

## Продвинутые паттерны

### Heartbeat и переподключение

```javascript
const wsLink = new GraphQLWsLink(
  createClient({
    url: 'ws://localhost:4000/graphql',

    // Переподключение
    retryAttempts: 5,

    // Параметры соединения
    connectionParams: async () => ({
      authToken: await getAuthToken()
    }),

    // Обработчики событий
    on: {
      connected: () => console.log('WebSocket connected'),
      closed: () => console.log('WebSocket closed'),
      error: (error) => console.error('WebSocket error:', error)
    },

    // Пользовательская логика переподключения
    shouldRetry: (errOrCloseEvent) => {
      // Не переподключаться при ошибках аутентификации
      if (errOrCloseEvent?.code === 4401) {
        return false;
      }
      return true;
    }
  })
);
```

### Подписка на несколько событий

```graphql
type Subscription {
  # Union тип для разных событий
  chatEvents(chatId: ID!): ChatEvent!
}

union ChatEvent = MessageReceived | UserJoined | UserLeft | TypingIndicator

type MessageReceived {
  message: Message!
}

type UserJoined {
  user: User!
  joinedAt: DateTime!
}

type UserLeft {
  user: User!
  leftAt: DateTime!
}

type TypingIndicator {
  user: User!
  isTyping: Boolean!
}
```

```graphql
subscription ChatEvents($chatId: ID!) {
  chatEvents(chatId: $chatId) {
    ... on MessageReceived {
      message {
        id
        text
        sender {
          name
        }
      }
    }
    ... on UserJoined {
      user {
        name
      }
      joinedAt
    }
    ... on UserLeft {
      user {
        name
      }
    }
    ... on TypingIndicator {
      user {
        name
      }
      isTyping
    }
  }
}
```

### Подписка с состоянием

```javascript
const resolvers = {
  Subscription: {
    onlineUsers: {
      subscribe: async function* (_, { roomId }, context) {
        // Начальное состояние
        const initialUsers = await getOnlineUsers(roomId);
        yield { onlineUsers: initialUsers };

        // Подписка на изменения
        const iterator = pubsub.asyncIterator([
          `USER_JOINED.${roomId}`,
          `USER_LEFT.${roomId}`
        ]);

        for await (const event of iterator) {
          const currentUsers = await getOnlineUsers(roomId);
          yield { onlineUsers: currentUsers };
        }
      }
    }
  }
};
```

## Масштабирование подписок

### Redis PubSub

```javascript
const { RedisPubSub } = require('graphql-redis-subscriptions');
const Redis = require('ioredis');

const options = {
  host: process.env.REDIS_HOST,
  port: process.env.REDIS_PORT,
  retryStrategy: (times) => Math.min(times * 50, 2000)
};

const pubsub = new RedisPubSub({
  publisher: new Redis(options),
  subscriber: new Redis(options)
});

// Использование такое же, как с обычным PubSub
pubsub.publish('POST_CREATED', { postCreated: post });
pubsub.asyncIterator(['POST_CREATED']);
```

### Архитектура масштабирования

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │     │   Client    │     │   Client    │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       │ WebSocket         │ WebSocket         │ WebSocket
       ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Server 1   │     │  Server 2   │     │  Server 3   │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼──────┐
                    │    Redis    │
                    │   PubSub    │
                    └─────────────┘
```

## Безопасность

### Аутентификация при подключении

```javascript
const wsServer = new WebSocketServer({
  server: httpServer,
  path: '/graphql'
});

useServer(
  {
    schema,

    // Аутентификация при подключении
    onConnect: async (ctx) => {
      const token = ctx.connectionParams?.authToken;

      if (!token) {
        throw new Error('Не предоставлен токен авторизации');
      }

      try {
        const user = await verifyToken(token);
        return { user };
      } catch (error) {
        throw new Error('Невалидный токен');
      }
    },

    // Авторизация для каждой подписки
    onSubscribe: async (ctx, msg) => {
      const { user } = ctx.extra;
      const { operationName, variables } = msg.payload;

      // Проверка прав доступа
      if (operationName === 'PrivateMessages') {
        const hasAccess = await checkChatAccess(user.id, variables.chatId);
        if (!hasAccess) {
          throw new Error('Нет доступа к этому чату');
        }
      }
    },

    // Очистка при отключении
    onDisconnect: async (ctx) => {
      const { user } = ctx.extra;
      if (user) {
        await setUserOffline(user.id);
      }
    }
  },
  wsServer
);
```

### Rate limiting

```javascript
const rateLimiter = new Map();

const onSubscribe = async (ctx, msg) => {
  const { user } = ctx.extra;
  const key = `${user.id}:subscriptions`;

  const current = rateLimiter.get(key) || 0;

  if (current >= 10) { // Максимум 10 подписок на пользователя
    throw new Error('Превышен лимит подписок');
  }

  rateLimiter.set(key, current + 1);
};

const onComplete = async (ctx, msg) => {
  const { user } = ctx.extra;
  const key = `${user.id}:subscriptions`;

  const current = rateLimiter.get(key) || 0;
  rateLimiter.set(key, Math.max(0, current - 1));
};
```

## Практические примеры

### Чат в реальном времени

```graphql
type Subscription {
  messageReceived(chatId: ID!): Message!
  typingIndicator(chatId: ID!): TypingStatus!
  onlineStatus(userId: ID!): OnlineStatus!
}

type Message {
  id: ID!
  text: String!
  sender: User!
  chat: Chat!
  createdAt: DateTime!
  readBy: [User!]!
}

type TypingStatus {
  user: User!
  isTyping: Boolean!
}

type OnlineStatus {
  user: User!
  isOnline: Boolean!
  lastSeen: DateTime
}
```

### Уведомления

```graphql
subscription Notifications {
  notificationReceived {
    id
    type
    title
    body
    data
    read
    createdAt

    ... on FollowNotification {
      follower {
        id
        name
        avatar
      }
    }

    ... on CommentNotification {
      comment {
        text
        post {
          id
          title
        }
      }
    }

    ... on LikeNotification {
      liker {
        name
      }
      post {
        id
        title
      }
    }
  }
}
```

### Обновление данных в реальном времени

```graphql
subscription StockPrices($symbols: [String!]!) {
  stockPriceUpdated(symbols: $symbols) {
    symbol
    price
    change
    changePercent
    updatedAt
  }
}

subscription OrderStatus($orderId: ID!) {
  orderStatusChanged(orderId: $orderId) {
    id
    status
    estimatedDelivery
    currentLocation {
      lat
      lng
      address
    }
    statusHistory {
      status
      timestamp
    }
  }
}
```

## Практические советы

1. **Используйте фильтрацию** — не отправляйте ненужные события
2. **Реализуйте heartbeat** — для обнаружения разрыва соединения
3. **Ограничивайте количество подписок** — предотвращайте злоупотребления
4. **Используйте Redis для масштабирования** — в multi-server окружении
5. **Обрабатывайте переподключения** — graceful reconnect на клиенте
6. **Логируйте подписки** — для мониторинга и отладки

## Заключение

Подписки — мощный инструмент для создания real-time приложений с GraphQL. Они обеспечивают типобезопасную передачу данных в реальном времени, сохраняя все преимущества GraphQL: гибкость запросов, строгую типизацию и самодокументирование. Правильная реализация подписок требует внимания к масштабированию, безопасности и обработке ошибок.
