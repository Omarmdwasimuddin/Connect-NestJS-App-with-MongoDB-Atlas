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
MONGO_URL=mongodb+srv://mdwasimu015_db_user:KSavMq0fMHVqQUQF@cluster0.kt25fpa.mongodb.net/?appName=Cluster0
mongodbPassword=KSavMq0fMHVqQUQF
```
---


### Install mongoose
```bash
npm i @nestjs/mongoose mongoose
```
---

>Note: app.module.ts file e add koro- import { MongooseModule } from '@nestjs/mongoose'; & imports: [EmployeeModule, CategoryModule, StudentModule, CustomerModule, ConfigModule.forRoot(), MongooseModule.forRoot(process.env.MONGO_URL!)], also note: app.module.ts file dekhe korte hobe github e .env er properties onno file thekeo hide kore dey!!!

### `app.module.ts`
```bash
import { MiddlewareConsumer, Module, NestModule } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { UserController } from './user/user.controller';
import { ProductService } from './product/product.service';
import { ProductController } from './product/product.controller';
import { EmployeeModule } from './employee/employee.module';
import { CategoryModule } from './category/category.module';
import { StudentModule } from './student/student.module';
import { CustomerModule } from './customer/customer.module';
import { MynameController } from './myname/myname.controller';
import { UserRolesController } from './user-roles/user-roles.controller';
import { ExceptionController } from './exception/exception.controller';
import { LoggerMiddleware } from './middleware/logger/logger.middleware';
import { DatabaseService } from './database/database.service';
import { DatabaseController } from './database/database.controller';
import { ConfigModule } from '@nestjs/config';
import { EvService } from './ev/ev.service';
import { EvController } from './ev/ev.controller';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [EmployeeModule, CategoryModule, StudentModule, CustomerModule, ConfigModule.forRoot(), MongooseModule.forRoot(process.env.MONGO_URL!)],
  controllers: [AppController, UserController, ProductController, MynameController, UserRolesController, ExceptionController, DatabaseController, EvController],
  providers: [AppService, ProductService, DatabaseService, EvService],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer){
    consumer.apply(LoggerMiddleware).forRoutes('*');
  } 
}
```
##


> npm run start:dev dile jodi error ashe tahole mongodb connect hoinai ar na ashle thikache.
---
