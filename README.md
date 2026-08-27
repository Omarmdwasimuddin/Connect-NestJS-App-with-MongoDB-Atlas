# Connect NestJS App with MongoDB Atlas


### install @nestjs/config
```bash
npm i @nestjs/config
```
>root e .env file toiri koro
---

>Note: browse- https://www.mongodb.com/cloud/atlas/register account create koro, clusters create koro .env te database connect koro.
>[MongoDB Atlas Setup](https://github.com/Omarmdwasimuddin/mongodb-atlas)
>##

### `.env`
```bash
MONGODB_USERNAME=""
MONGODB_PASSWORD=""
MONGODB_URI=""
```
---


### Install mongoose
```bash
npm i @nestjs/mongoose mongoose
```
---

>Note: app.module.ts file e add koro- ConfigModule.forRoot(), MongooseModule.forRoot(process.env.MONGODB_URI!)

### `app.module.ts`
```bash
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from '@nestjs/config';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [ ConfigModule.forRoot(), MongooseModule.forRoot(process.env.MONGODB_URI!) ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```
##


> npm run start:dev dile jodi error ashe tahole mongodb connect hoinai ar na ashle thikache.
---
