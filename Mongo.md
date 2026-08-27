# NestJS Mongo (Mongoose)

## MongoDB Integration-এর দুইটা উপায়

Nest-এ MongoDB-এর সাথে কাজ করার দুইটা পথ আছে:
1. **TypeORM** (এটারও একটা MongoDB connector আছে)
2. **Mongoose** — সবচেয়ে জনপ্রিয় MongoDB object modeling tool, `@nestjs/mongoose` package দিয়ে

এই chapter Mongoose নিয়ে।

```bash
npm i @nestjs/mongoose mongoose
```

Root `AppModule`-এ `MongooseModule` import করা:

```ts
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [MongooseModule.forRoot('mongodb://localhost/nest')],
})
export class AppModule {}
```

`forRoot()`-এ ঠিক Mongoose-এর নিজের `mongoose.connect()`-এর মতোই config object দেওয়া যায়।

---

## Model Injection (Schema, Model, Document)

Mongoose-এর সবকিছু শুরু হয় **Schema** থেকে। প্রতিটা schema একটা MongoDB collection-এর সাথে map হয় এবং সেই collection-এর document-গুলোর shape ঠিক করে। Schema দিয়ে **Model** define হয় — Model-ই database থেকে document create/read করার দায়িত্বে থাকে।

Schema দুইভাবে বানানো যায়:
1. NestJS decorator দিয়ে (recommended — boilerplate কম, readable)
2. Mongoose দিয়ে সরাসরি manually

### Decorator দিয়ে Schema বানানো

```ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument } from 'mongoose';

export type CatDocument = HydratedDocument<Cat>;

@Schema()
export class Cat {
  @Prop()
  name: string;

  @Prop()
  age: number;

  @Prop()
  breed: string;
}

export const CatSchema = SchemaFactory.createForClass(Cat);
```

> **Hint:** `DefinitionsFactory` class (`@nestjs/mongoose` থেকে) দিয়ে raw schema definition-ও generate করা যায় — যেসব edge-case decorator দিয়ে represent করা কঠিন, সেখানে এটা কাজে লাগে।

### `@Schema()` decorator

এটা একটা class-কে schema definition হিসেবে mark করে। `Cat` class-টা একই নামের MongoDB collection-এর সাথে map হয়, কিন্তু শেষে একটা extra "s" যোগ হয় — মানে final collection-এর নাম হবে `cats`। এটা একটা optional argument নেয় — schema options object (ঠিক যেমন `mongoose.Schema` constructor-এর দ্বিতীয় argument হিসেবে দেওয়া হয়)।

### `@Prop()` decorator

Document-এর একটা property define করে। উপরের উদাহরণে `name`, `age`, `breed` — তিনটা property define করা হয়েছে। TypeScript metadata/reflection-এর কারণে এগুলোর schema type automatic ঠিক হয়ে যায়। তবে complex case-এ (যেমন array বা nested object), type explicit ভাবে দিতে হয়:

```ts
@Prop([String])
tags: string[];
```

`@Prop()`-এ options object-ও দেওয়া যায় — property required কিনা, default value, immutable কিনা ইত্যাদি:

```ts
@Prop({ required: true })
name: string;
```

### Relation (অন্য Model-এর সাথে সম্পর্ক)

`Cat`-এর যদি `Owner` থাকে (আলাদা `owners` collection-এ), তাহলে `type` আর `ref` দিতে হবে:

```ts
import * as mongoose from 'mongoose';
import { Owner } from '../owners/schemas/owner.schema';

@Prop({ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' })
owner: Owner;
```

একাধিক owner থাকলে:

```ts
@Prop({ type: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' }] })
owners: Owner[];
```

সবসময় populate না করতে চাইলে, type হিসেবে `mongoose.Types.ObjectId` ব্যবহার করা ভালো:

```ts
@Prop({ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' })
owner: mongoose.Types.ObjectId;
```

পরে দরকার হলে populate করা যায়:

```ts
async findAllPopulated() {
  return this.catModel.find().populate<{ owner: Owner }>("owner");
}
```

> **Hint:** যদি populate করার মতো foreign document না থাকে, type হতে পারে `Owner | null` (Mongoose configuration অনুযায়ী) অথবা error-ও throw হতে পারে, তখন type হবে `Owner`।

### Raw Schema Definition

Nested object-এর জন্য যেটা কোনো class দিয়ে define করা নেই, `raw()` function ব্যবহার করা যায়:

```ts
@Prop(raw({
  firstName: { type: String },
  lastName: { type: String }
}))
details: Record<string, any>;
```

### Decorator ছাড়া Manual Schema

```ts
export const CatSchema = new mongoose.Schema({
  name: String,
  age: Number,
  breed: String,
});
```

> Schema file সাধারণত সংশ্লিষ্ট domain object-এর কাছাকাছি রাখা ভালো — যেমন `cats` folder-এর ভিতরে, যেখানে `CatsModule`-ও আছে।

---

## Module-এ Schema Register করা

```ts
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';
import { Cat, CatSchema } from './schemas/cat.schema';

@Module({
  imports: [MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }])],
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

`MongooseModule.forFeature()` দিয়ে ঠিক করা হয় কোন model current scope-এ register হবে। অন্য module-এও এই model ব্যবহার করতে চাইলে, `CatsModule`-এর `exports`-এ `MongooseModule` যোগ করে সেই module-এ `CatsModule` import করতে হবে।

---

## Model Inject করা

`@InjectModel()` decorator দিয়ে service-এ model inject করা:

```ts
import { Model } from 'mongoose';
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Cat } from './schemas/cat.schema';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  constructor(@InjectModel(Cat.name) private catModel: Model<Cat>) {}

  async create(createCatDto: CreateCatDto): Promise<Cat> {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll(): Promise<Cat[]> {
    return this.catModel.find().exec();
  }
}
```

---

## Connection সরাসরি ব্যবহার করা

কখনো Mongoose-এর native `Connection` object দরকার হতে পারে (native API call করার জন্য):

```ts
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection() private connection: Connection) {}
}
```

---

## Sessions (Transaction-এর জন্য)

Mongoose-এ session শুরু করতে হলে, সরাসরি `mongoose.startSession()` call না করে `@InjectConnection` দিয়ে connection inject করা recommended — এতে NestJS-এর DI system-এর সাথে ভালোভাবে integrate হয়, connection management সঠিক থাকে।

```ts
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection() private readonly connection: Connection) {}

  async startTransaction() {
    const session = await this.connection.startSession();
    session.startTransaction();
    // Your transaction logic here
  }
}
```

Session শুরু করার পর, নিজের logic অনুযায়ী transaction commit বা abort করতে ভুলো না।

---

## একাধিক Database (Multiple Connections)

কিছু project-এ একাধিক database connection দরকার হয়। এর জন্য connection-এর নাম দেওয়া **বাধ্যতামূলক**:

```ts
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionName: 'cats',
    }),
    MongooseModule.forRoot('mongodb://localhost/users', {
      connectionName: 'users',
    }),
  ],
})
export class AppModule {}
```

> **Notice:** নাম ছাড়া বা একই নামে একাধিক connection বানানো যাবে না — তাহলে একটা আরেকটাকে override করে ফেলবে।

`forFeature()`-এ কোন connection ব্যবহার হবে সেটা বলে দিতে হবে:

```ts
@Module({
  imports: [
    MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }], 'cats'),
  ],
})
export class CatsModule {}
```

নির্দিষ্ট connection inject করাও যায়:

```ts
constructor(@InjectConnection('cats') private connection: Connection) {}
```

Custom provider (যেমন factory provider)-এ নির্দিষ্ট connection দিতে `getConnectionToken()` ব্যবহার করা যায়:

```ts
{
  provide: CatsService,
  useFactory: (catsConnection: Connection) => {
    return new CatsService(catsConnection);
  },
  inject: [getConnectionToken('cats')],
}
```

Named database থেকে model inject করতে, `@InjectModel()`-এ দ্বিতীয় parameter হিসেবে connection name দেওয়া যায়:

```ts
constructor(@InjectModel(Cat.name, 'cats') private catModel: Model<Cat>) {}
```

---

## Hooks (Middleware) — Pre/Post

Mongoose-এ **middleware** (বা **pre**/**post** hook) হলো এমন function যেগুলো asynchronous function execute হওয়ার সময় control পায়। এগুলো schema-level-এ define হয়, plugin লেখার জন্য useful।

**গুরুত্বপূর্ণ:** Model compile হয়ে যাওয়ার পর `pre()`/`post()` call করলে কাজ করে না। তাই model register হওয়ার **আগেই** hook register করতে হয় — এর জন্য `MongooseModule.forFeatureAsync()` + factory provider (`useFactory`) ব্যবহার করতে হয়:

```ts
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatSchema;
          schema.pre('save', function () {
            console.log('Hello from pre save');
          });
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

অন্যান্য factory provider-এর মতোই, এই factory function `async` হতে পারে এবং `inject`-এর মাধ্যমে dependency নিতে পারে:

```ts
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        imports: [ConfigModule],
        useFactory: (configService: ConfigService) => {
          const schema = CatSchema;
          schema.pre('save', function() {
            console.log(
              `${configService.get('APP_NAME')}: Hello from pre save`,
            );
          });
          return schema;
        },
        inject: [ConfigService],
      },
    ]),
  ],
})
export class AppModule {}
```

---

## Plugins

নির্দিষ্ট schema-র জন্য plugin register করতে `forFeatureAsync()` ব্যবহার করা:

```ts
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatSchema;
          schema.plugin(require('mongoose-autopopulate'));
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

সবগুলো schema-তে একসাথে plugin register করতে — model তৈরি হওয়ার আগেই connection access করতে হয়, `connectionFactory` দিয়ে:

```ts
@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionFactory: (connection) => {
        connection.plugin(require('mongoose-autopopulate'));
        return connection;
      }
    }),
  ],
})
export class AppModule {}
```

---

## Discriminators (Schema Inheritance)

Discriminator হলো এক ধরনের schema inheritance mechanism — একই underlying MongoDB collection-এর উপর overlapping schema-সহ একাধিক Model রাখা যায়।

যেমন — একই collection-এ বিভিন্ন ধরনের event track করতে চাইলে (প্রতিটার একটা timestamp থাকবে):

```ts
@Schema({ discriminatorKey: 'kind' })
export class Event {
  @Prop({
    type: String,
    required: true,
    enum: [ClickedLinkEvent.name, SignUpEvent.name],
  })
  kind: string;

  @Prop({ type: Date, required: true })
  time: Date;
}

export const EventSchema = SchemaFactory.createForClass(Event);
```

> **Hint:** Mongoose বিভিন্ন discriminator model-এর মধ্যে পার্থক্য করে **"discriminator key"** দিয়ে, যেটার default নাম `__t`। Mongoose নিজে থেকেই `__t` নামের একটা String path schema-তে যোগ করে, যেটা দিয়ে বোঝা যায় কোন document কোন discriminator-এর instance। `discriminatorKey` option দিয়ে এই path-এর নাম বদলানো যায়।

`ClickedLinkEvent` আর `SignUpEvent` — দুটোই generic `Event`-এর same collection-এ store হবে:

```ts
@Schema()
export class ClickedLinkEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  url: string;
}

export const ClickedLinkEventSchema = SchemaFactory.createForClass(ClickedLinkEvent);
```

```ts
@Schema()
export class SignUpEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  user: string;
}

export const SignUpEventSchema = SchemaFactory.createForClass(SignUpEvent);
```

Discriminator register করা — `MongooseModule.forFeature` বা `forFeatureAsync` দুটোতেই কাজ করে:

```ts
@Module({
  imports: [
    MongooseModule.forFeature([
      {
        name: Event.name,
        schema: EventSchema,
        discriminators: [
          { name: ClickedLinkEvent.name, schema: ClickedLinkEventSchema },
          { name: SignUpEvent.name, schema: SignUpEventSchema },
        ],
      },
    ]),
  ]
})
export class EventsModule {}
```

---

## Testing (Mock Model বানানো)

Unit test-এ সাধারণত real database connection avoid করা ভালো — test সহজ ও দ্রুত করতে। কিন্তু class-গুলো model-এর উপর নির্ভরশীল হতে পারে। সমাধান: **mock model** বানানো।

`@nestjs/mongoose`-এ `getModelToken()` function আছে, যেটা একটা token name থেকে injection token তৈরি করে দেয়। এই token দিয়ে `useClass`, `useValue`, `useFactory` — যেকোনো custom provider technique দিয়ে mock দেওয়া যায়:

```ts
@Module({
  providers: [
    CatsService,
    {
      provide: getModelToken(Cat.name),
      useValue: catModel,
    },
  ],
})
export class CatsModule {}
```

এখানে যখনই কোনো consumer `@InjectModel()` দিয়ে `Model<Cat>` inject করবে, একটা hardcoded `catModel` object দেওয়া হবে।

---

## Async Configuration

Module option static-এর বদলে asynchronously পাঠাতে চাইলে `forRootAsync()` ব্যবহার করতে হয়।

### Factory Function দিয়ে

```ts
MongooseModule.forRootAsync({
  useFactory: () => ({
    uri: 'mongodb://localhost/nest',
  }),
});
```

Async ও dependency inject করা যায়:

```ts
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    uri: configService.get<string>('MONGODB_URI'),
  }),
  inject: [ConfigService],
});
```

### Class দিয়ে

```ts
MongooseModule.forRootAsync({
  useClass: MongooseConfigService,
});
```

এখানে `MongooseConfigService`-কে `MongooseOptionsFactory` interface implement করতে হয়:

```ts
@Injectable()
export class MongooseConfigService implements MongooseOptionsFactory {
  createMongooseOptions(): MongooseModuleOptions {
    return {
      uri: 'mongodb://localhost/nest',
    };
  }
}
```

### Existing Provider Reuse করা

```ts
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

---

## Connection Events

`onConnectionCreate` option দিয়ে Mongoose connection event শোনা যায় — `connected`, `open`, `disconnected`, `reconnected`, `disconnecting` ইত্যাদি:

```ts
MongooseModule.forRoot('mongodb://localhost/test', {
  onConnectionCreate: (connection: Connection) => {
    connection.on('connected', () => console.log('connected'));
    connection.on('open', () => console.log('open'));
    connection.on('disconnected', () => console.log('disconnected'));
    connection.on('reconnected', () => console.log('reconnected'));
    connection.on('disconnecting', () => console.log('disconnecting'));

    return connection;
  },
}),
```

| Event | কখন হয় |
|---|---|
| `connected` | Connection সফলভাবে established হলে |
| `open` | Connection পুরোপুরি open এবং operation-এর জন্য ready হলে |
| `disconnected` | Connection হারিয়ে গেলে |
| `reconnected` | Disconnect হওয়ার পর আবার connect হলে |
| `disconnecting` | Connection বন্ধ হওয়ার প্রক্রিয়ায় থাকলে |

`forRootAsync()`-এও `onConnectionCreate` ব্যবহার করা যায়:

```ts
MongooseModule.forRootAsync({
  useFactory: () => ({
    uri: 'mongodb://localhost/test',
    onConnectionCreate: (connection: Connection) => {
      return connection;
    },
  }),
}),
```

---

## Subdocuments

Parent document-এর ভিতরে subdocument nest করা:

```ts
@Schema()
export class Name {
  @Prop()
  firstName: string;

  @Prop()
  lastName: string;
}

export const NameSchema = SchemaFactory.createForClass(Name);
```

Parent schema-তে reference করা:

```ts
@Schema()
export class Person {
  @Prop(NameSchema)
  name: Name;
}

export const PersonSchema = SchemaFactory.createForClass(Person);

export type PersonDocumentOverride = {
  name: Types.Subdocument<Types.ObjectId> & Name;
};

export type PersonDocument = HydratedDocument<Person, PersonDocumentOverride>;
```

একাধিক subdocument (array) দরকার হলে, type override করতে হবে যথাযথভাবে:

```ts
@Schema()
export class Person {
  @Prop([NameSchema])
  name: Name[];
}

export const PersonSchema = SchemaFactory.createForClass(Person);

export type PersonDocumentOverride = {
  name: Types.DocumentArray<Name>;
};

export type PersonDocument = HydratedDocument<Person, PersonDocumentOverride>;
```

---

## Virtuals

**Virtual** হলো একটা property যেটা document-এ থাকে কিন্তু MongoDB-তে actually save হয় না — access করার সময় dynamically compute হয়। সাধারণত derived/computed value-এর জন্য ব্যবহার হয় (যেমন `firstName` + `lastName` জোড়া দিয়ে `fullName` বানানো)।

```ts
class Person {
  @Prop()
  firstName: string;

  @Prop()
  lastName: string;

  @Virtual({
    get: function (this: Person) {
      return `${this.firstName} ${this.lastName}`;
    },
  })
  fullName: string;
}
```

> **Hint:** `@Virtual()` decorator আসে `@nestjs/mongoose` থেকে।

এখানে `fullName` `firstName` আর `lastName` থেকে derive হচ্ছে। Access করলে normal property-র মতোই behave করে, কিন্তু MongoDB document-এ কখনো save হয় না।

---

## সংক্ষেপে (Quick Summary)

| বিষয় | মূল কথা |
|---|---|
| Package | `@nestjs/mongoose` + `mongoose` |
| Connect | `MongooseModule.forRoot(uri)` — root module-এ |
| Schema বানানো | `@Schema()` class + `@Prop()` property + `SchemaFactory.createForClass()` |
| Feature module-এ register | `MongooseModule.forFeature([{ name, schema }])` |
| Model inject | `@InjectModel(Cat.name)` |
| Connection inject | `@InjectConnection()` |
| Session/Transaction | `connection.startSession()` |
| Multiple DB | `connectionName` দিয়ে আলাদা connection, `@InjectModel(name, connectionName)` |
| Pre/Post hook | `forFeatureAsync()` + `schema.pre()/post()` (model compile হওয়ার আগে করতে হবে) |
| Plugin | `schema.plugin()` (single schema) বা `connectionFactory` (সব schema-তে) |
| Discriminator | একই collection-এ একাধিক overlapping schema/model |
| Testing | `getModelToken()` দিয়ে mock model provide করা |
| Async config | `forRootAsync()` — factory/class/existing তিনভাবে |
| Connection events | `onConnectionCreate` দিয়ে connected/disconnected ইত্যাদি listen করা |
| Subdocument | Nested schema প্রোপার্টি হিসেবে বসানো |
| Virtual | Save না হওয়া, dynamically-computed property |
