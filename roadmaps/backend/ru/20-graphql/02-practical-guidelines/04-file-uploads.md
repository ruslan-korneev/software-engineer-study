# Загрузка файлов в GraphQL

[prev: 03-serving-over-http](./03-serving-over-http.md) | [next: 05-authorization](./05-authorization.md)

---

## Введение

GraphQL изначально не поддерживает загрузку файлов, так как работает с JSON. Однако существует несколько подходов для реализации этой функциональности. Наиболее популярный — спецификация **graphql-multipart-request-spec**.

## Подходы к загрузке файлов

| Подход | Описание | Плюсы | Минусы |
|--------|----------|-------|--------|
| Multipart Request | Файлы в multipart/form-data | Единый запрос | Сложнее настроить |
| Signed URLs | Прямая загрузка в хранилище | Масштабируемость | Два запроса |
| Base64 | Кодирование в строку | Простота | Увеличение размера на 33% |
| Отдельный эндпоинт | REST для файлов | Простота | Разделение логики |

## GraphQL Multipart Request Spec

### Принцип работы

Спецификация определяет способ отправки файлов вместе с GraphQL операциями:

```
Content-Type: multipart/form-data

operations: {"query": "mutation($file: Upload!) { uploadFile(file: $file) { url } }", "variables": {"file": null}}
map: {"0": ["variables.file"]}
0: <binary file data>
```

### Структура запроса

1. **operations** — JSON с GraphQL запросом и переменными (файлы заменены на `null`)
2. **map** — маппинг между файлами и путями в переменных
3. **0, 1, 2...** — сами файлы

## Реализация на сервере

### Node.js с graphql-upload

```javascript
import { graphqlUploadExpress } from 'graphql-upload';
import express from 'express';
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';

const typeDefs = `
  scalar Upload

  type File {
    filename: String!
    mimetype: String!
    encoding: String!
    url: String!
  }

  type Mutation {
    uploadFile(file: Upload!): File!
    uploadFiles(files: [Upload!]!): [File!]!
  }
`;

const resolvers = {
  Mutation: {
    uploadFile: async (_, { file }) => {
      const { createReadStream, filename, mimetype, encoding } = await file;

      // Получаем поток данных файла
      const stream = createReadStream();

      // Сохраняем файл
      const url = await saveFile(stream, filename);

      return { filename, mimetype, encoding, url };
    },

    uploadFiles: async (_, { files }) => {
      const results = await Promise.all(
        files.map(async (file) => {
          const { createReadStream, filename, mimetype, encoding } = await file;
          const stream = createReadStream();
          const url = await saveFile(stream, filename);
          return { filename, mimetype, encoding, url };
        })
      );
      return results;
    },
  },
};

const app = express();

// Важно: middleware должен быть перед Apollo
app.use(graphqlUploadExpress({ maxFileSize: 10000000, maxFiles: 10 }));

const server = new ApolloServer({ typeDefs, resolvers });
await server.start();

app.use('/graphql', express.json(), expressMiddleware(server));
```

### Функция сохранения файла

```javascript
import { createWriteStream } from 'fs';
import { pipeline } from 'stream/promises';
import path from 'path';
import { v4 as uuid } from 'uuid';

async function saveFile(stream, filename) {
  const ext = path.extname(filename);
  const newFilename = `${uuid()}${ext}`;
  const filepath = path.join('./uploads', newFilename);

  await pipeline(stream, createWriteStream(filepath));

  return `/uploads/${newFilename}`;
}
```

### Python с Strawberry

```python
import strawberry
from strawberry.file_uploads import Upload
from typing import List
import shutil
from pathlib import Path
import uuid

@strawberry.type
class FileInfo:
    filename: str
    content_type: str
    size: int
    url: str

@strawberry.type
class Mutation:
    @strawberry.mutation
    async def upload_file(self, file: Upload) -> FileInfo:
        content = await file.read()
        filename = f"{uuid.uuid4()}{Path(file.filename).suffix}"
        filepath = Path("uploads") / filename

        with open(filepath, "wb") as f:
            f.write(content)

        return FileInfo(
            filename=file.filename,
            content_type=file.content_type,
            size=len(content),
            url=f"/uploads/{filename}"
        )

    @strawberry.mutation
    async def upload_files(self, files: List[Upload]) -> List[FileInfo]:
        results = []
        for file in files:
            result = await self.upload_file(file)
            results.append(result)
        return results
```

## Клиентская сторона

### JavaScript с Apollo Client

```javascript
import { ApolloClient, InMemoryCache } from '@apollo/client';
import { createUploadLink } from 'apollo-upload-client';

const client = new ApolloClient({
  link: createUploadLink({ uri: '/graphql' }),
  cache: new InMemoryCache(),
});

// Мутация
const UPLOAD_FILE = gql`
  mutation UploadFile($file: Upload!) {
    uploadFile(file: $file) {
      filename
      url
    }
  }
`;

// Использование
function FileUploader() {
  const [uploadFile] = useMutation(UPLOAD_FILE);

  const handleChange = async (e) => {
    const file = e.target.files[0];
    const { data } = await uploadFile({
      variables: { file },
    });
    console.log('Uploaded:', data.uploadFile.url);
  };

  return <input type="file" onChange={handleChange} />;
}
```

### Fetch API

```javascript
async function uploadFile(file) {
  const operations = JSON.stringify({
    query: `
      mutation($file: Upload!) {
        uploadFile(file: $file) {
          filename
          url
        }
      }
    `,
    variables: { file: null },
  });

  const map = JSON.stringify({ '0': ['variables.file'] });

  const formData = new FormData();
  formData.append('operations', operations);
  formData.append('map', map);
  formData.append('0', file);

  const response = await fetch('/graphql', {
    method: 'POST',
    body: formData,
    // НЕ устанавливайте Content-Type вручную!
    // Browser установит его автоматически с boundary
  });

  return response.json();
}
```

## Подход с Signed URLs

### Преимущества

- Файлы загружаются напрямую в облачное хранилище
- Сервер не нагружается передачей данных
- Лучше масштабируется

### Схема

```graphql
type SignedUrl {
  uploadUrl: String!
  fileUrl: String!
  expiresAt: DateTime!
}

type Mutation {
  # Шаг 1: Получить URL для загрузки
  createUploadUrl(
    filename: String!
    contentType: String!
  ): SignedUrl!

  # Шаг 3: Подтвердить загрузку
  confirmUpload(fileUrl: String!): File!
}
```

### Реализация с AWS S3

```javascript
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3Client = new S3Client({ region: 'us-east-1' });

const resolvers = {
  Mutation: {
    createUploadUrl: async (_, { filename, contentType }) => {
      const key = `uploads/${uuid()}/${filename}`;

      const command = new PutObjectCommand({
        Bucket: 'my-bucket',
        Key: key,
        ContentType: contentType,
      });

      const uploadUrl = await getSignedUrl(s3Client, command, {
        expiresIn: 3600,
      });

      return {
        uploadUrl,
        fileUrl: `https://my-bucket.s3.amazonaws.com/${key}`,
        expiresAt: new Date(Date.now() + 3600000),
      };
    },

    confirmUpload: async (_, { fileUrl }) => {
      // Проверяем, что файл действительно загружен
      // Сохраняем информацию в БД
      return db.files.create({ url: fileUrl });
    },
  },
};
```

### Клиентский код

```javascript
async function uploadWithSignedUrl(file) {
  // Шаг 1: Получаем URL для загрузки
  const { data } = await client.mutate({
    mutation: CREATE_UPLOAD_URL,
    variables: {
      filename: file.name,
      contentType: file.type,
    },
  });

  const { uploadUrl, fileUrl } = data.createUploadUrl;

  // Шаг 2: Загружаем файл напрямую в S3
  await fetch(uploadUrl, {
    method: 'PUT',
    body: file,
    headers: {
      'Content-Type': file.type,
    },
  });

  // Шаг 3: Подтверждаем загрузку
  const { data: confirmData } = await client.mutate({
    mutation: CONFIRM_UPLOAD,
    variables: { fileUrl },
  });

  return confirmData.confirmUpload;
}
```

## Валидация файлов

### На сервере

```javascript
const resolvers = {
  Mutation: {
    uploadFile: async (_, { file }) => {
      const { createReadStream, filename, mimetype } = await file;

      // Проверка MIME-типа
      const allowedTypes = ['image/jpeg', 'image/png', 'image/gif'];
      if (!allowedTypes.includes(mimetype)) {
        throw new GraphQLError('Invalid file type', {
          extensions: { code: 'INVALID_FILE_TYPE' },
        });
      }

      // Проверка расширения
      const ext = path.extname(filename).toLowerCase();
      const allowedExts = ['.jpg', '.jpeg', '.png', '.gif'];
      if (!allowedExts.includes(ext)) {
        throw new GraphQLError('Invalid file extension', {
          extensions: { code: 'INVALID_FILE_EXTENSION' },
        });
      }

      // Проверка размера (через stream)
      const stream = createReadStream();
      let size = 0;
      const maxSize = 5 * 1024 * 1024; // 5MB

      for await (const chunk of stream) {
        size += chunk.length;
        if (size > maxSize) {
          throw new GraphQLError('File too large', {
            extensions: { code: 'FILE_TOO_LARGE' },
          });
        }
      }

      // ... сохранение файла
    },
  },
};
```

### Типы файлов в схеме

```graphql
enum AllowedImageType {
  JPEG
  PNG
  GIF
  WEBP
}

input ImageUploadInput {
  file: Upload!
  type: AllowedImageType!
  alt: String
}

type Mutation {
  uploadImage(input: ImageUploadInput!): Image!
}
```

## Обработка ошибок

```javascript
const resolvers = {
  Mutation: {
    uploadFile: async (_, { file }) => {
      try {
        const { createReadStream, filename } = await file;
        const stream = createReadStream();

        // Обработка ошибок потока
        stream.on('error', (error) => {
          throw new GraphQLError('Stream error', {
            extensions: {
              code: 'UPLOAD_STREAM_ERROR',
              details: error.message,
            },
          });
        });

        const url = await saveFile(stream, filename);
        return { filename, url };
      } catch (error) {
        if (error.message.includes('ENOSPC')) {
          throw new GraphQLError('No space left on disk', {
            extensions: { code: 'STORAGE_FULL' },
          });
        }
        throw error;
      }
    },
  },
};
```

## Лучшие практики

1. **Ограничивайте размер файлов** — защита от DoS
2. **Валидируйте MIME-типы** — не доверяйте расширениям
3. **Генерируйте уникальные имена** — избегайте коллизий
4. **Используйте CDN** — для отдачи файлов
5. **Храните метаданные в БД** — имя, размер, тип, владелец
6. **Сканируйте на вирусы** — для критичных приложений

## Заключение

Загрузка файлов в GraphQL требует дополнительной настройки, но существующие решения делают этот процесс достаточно простым. Выбор между Multipart Request и Signed URLs зависит от масштаба приложения и инфраструктуры.

---

[prev: 03-serving-over-http](./03-serving-over-http.md) | [next: 05-authorization](./05-authorization.md)
