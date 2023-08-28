# realtime-chat

This repository for use redis pub/sub for realtime chat system

## Architecture

![](https://i.imgur.com/uxveYO5.png)

## server

1. init setup

```shell
pnpm init -y
```

2. install need package

```shell
pnpm add fastify fastify-socket.io ioredis close-with-grace socket.io @fastify/cors dotenv
pnpm add @types/node typescript tsx @types/ws -D
```

3. initial typescript
   
```shell
pnpm tsc --init
```

4. modify outdir to build

5. setup build script

```json
"scripts": {
  "dev": "tsx watch src/main.ts",
  "build": "tsc",
  "dev-run": "tsc && node ./build/main.js"
},
```

6. set fastify server 

```typescript
import dotenv from 'dotenv';
import fastify from 'fastify';
import fastifyCors from '@fastify/cors';
import fastifyIO from 'fastify-socket.io';
import Redis from 'ioredis';
import closeWithGrace from 'close-with-grace';
import { randomUUID } from 'crypto';
dotenv.config();

const PORT = parseInt(process.env.PORT || '3001', 10);
const HOST = process.env.HOST || '0.0.0.0';
const CORS_ORIGIN = process.env.CORS_ORIGIN || 'http://localhost:3000';
const UPSTASH_REDIS_REST_URL = process.env.UPSTASH_REDIS_REST_URL;
const CONNECTION_COUNT_KEY = "chat:connection-count";
const CONNECTION_COUNT_UPDATED_CHANNEL = 'chat:connection-count-updated';
const NEW_MESSAGE_CHANNEL = 'chat:new-message';

if (!UPSTASH_REDIS_REST_URL) {
  console.error('missing UPSTASH_REDIS_REST_URL');
  process.exit(1);
}

const publisher = new Redis(UPSTASH_REDIS_REST_URL);
const subscriber = new Redis(UPSTASH_REDIS_REST_URL); 
let connectedClients = 0;
async function buildServer() {
  const app = fastify();
  await app.register(fastifyCors, {
    origin: CORS_ORIGIN,
  });
  await app.register(fastifyIO);
  const currentCount = await publisher.get(CONNECTION_COUNT_KEY);
  if (!currentCount) {
    await publisher.set(CONNECTION_COUNT_KEY, 0);
  }

  app.io.on('connection', async (io) => {
    console.log('Client connected');
    const incrResult = await publisher.incr(CONNECTION_COUNT_KEY);
    connectedClients++;
    await publisher.publish(CONNECTION_COUNT_UPDATED_CHANNEL, String(incrResult));
    
    io.on(NEW_MESSAGE_CHANNEL, (payload) => {
      const message = payload.message;
      if (!message) {
        return;
      }
      publisher.publish(NEW_MESSAGE_CHANNEL, message.toString());
    });
    io.on('disconnect', async () => {
      console.log('Client disconnected');
      const decrResult = await publisher.decr(CONNECTION_COUNT_KEY);
      connectedClients--;
      await publisher.publish(CONNECTION_COUNT_UPDATED_CHANNEL, String(decrResult));
    })
  });
  subscriber.subscribe(NEW_MESSAGE_CHANNEL, (err, count) => {
    if (err) {
      console.error(`Error subscribing to ${NEW_MESSAGE_CHANNEL}`);
      return;
    }
    console.log(`${count} clients subscribe to ${NEW_MESSAGE_CHANNEL} channel`);
  });
  subscriber.subscribe(CONNECTION_COUNT_UPDATED_CHANNEL, (err, count) => {
    if (err) {
      console.error(`Error subscribing to ${CONNECTION_COUNT_UPDATED_CHANNEL}`, err);
      return
    }
    console.log(`${count} clients subscribe to ${CONNECTION_COUNT_UPDATED_CHANNEL} channel`);
  });

  subscriber.on('message', (channel, text) => {
    if(channel === CONNECTION_COUNT_UPDATED_CHANNEL) {
      app.io.emit(CONNECTION_COUNT_UPDATED_CHANNEL, {
        count: text,
      });
      return;
    }
    if (channel === NEW_MESSAGE_CHANNEL) {
      app.io.emit(NEW_MESSAGE_CHANNEL, {
        message: text,
        id: randomUUID(),
        createdAt: new Date(),
        port: PORT,
      })
      return;
    }
  })
  app.get('/healthcheck', () => {
    return {
      status: 'ok',
      port: PORT,
    }
  });
  return app;
}

async function main() {
  const app = await buildServer();

  try {
    await app.listen({
      port: PORT,
      host: HOST
    });
    
    closeWithGrace({delay: 2000 }, async({ signal, err}) => {
      console.log('shutting down');
      console.log({ err, signal });
      if (connectedClients > 0) {
        console.log(`Removing ${connectedClients} from the count`);
        const currentCount = parseInt(await publisher.get(CONNECTION_COUNT_KEY) || '0', 10);
        const newCount = Math.max(currentCount - connectedClients, 0);
        await publisher.set(CONNECTION_COUNT_KEY, newCount);
      }
      await app.close();
      console.log('shutdonw complete goodbye');
    })
    console.log(`Server started at http://${HOST}:${PORT}`);
  } catch (e) {
    console.error(e);
    process.exit(1);
  }
}
main();
```