## Nest.js: IsUnique from TypeORM decorator for DTO
### Version: 30.01.2023

# RU:
```bash
Как в Nest.js сделать декоратор, который позволил бы валидировать в моём DTO поле,
проверяя его на уникальность в базе данных (TypeORM), при этом что бы была возможность
указать WHERE на основе данных запроса взятых из Request?
```


#### 1. Edit file: `main.ts`
```
import { useContainer } from "class-validator";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  useContainer(app.select(AppModule), { fallbackOnErrors: true });
  ...
}
```

#### 2. Create file `decorators/typeorm-validate.decorator.ts`
```bash
import {
  registerDecorator,
  ValidationArguments,
  ValidationOptions,
  ValidatorConstraint,
  ValidatorConstraintInterface
} from "class-validator";
import { DataSource, Repository } from "typeorm";
import { Inject, Injectable } from "@nestjs/common";
import { FindOptionsWhere } from "typeorm/find-options/FindOptionsWhere";

type IsUniqueWhere<E, DK> = (dto: DK) => FindOptionsWhere<E> | FindOptionsWhere<E>[];

export const IsUnique = <E, K extends keyof E, D>(entity: new() => E, field: K, dto?: new() => D, where?: IsUniqueWhere<E, D>, validationOptions?: ValidationOptions) =>
  (object: any, propertyName: string) =>
    registerDecorator({
      name: "IsUnique",
      target: object.constructor,
      propertyName: propertyName,
      options: validationOptions,
      constraints: [entity, field, where],
      validator: IsUniqueRule
    });


@Injectable()
@ValidatorConstraint({ name: "IsUnique", async: true })
export class IsUniqueRule<E, D> implements ValidatorConstraintInterface {

  constructor(
    @Inject(DataSource) private readonly dataSource: DataSource
  ) {
  }

  async validate(value: string, args: ValidationArguments): Promise<boolean> {
    const [entity, field, where] = args.constraints;
    const repository: Repository<E> = this.dataSource.getRepository(entity);
    let findOptionsWhere: FindOptionsWhere<E> = {};

    if (where) {
      const dto = args.object as D;

      findOptionsWhere = {
        ...findOptionsWhere,
        ...await where(dto)
      };
    }
    findOptionsWhere[field] = value;

    return (await repository.countBy(findOptionsWhere)) === 0;
  }

  defaultMessage = () => `BUSY`;
}
```

#### 3. Create file `interceptors/body-context.interceptor.ts`
```bash
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler
} from "@nestjs/common";
import { Observable } from "rxjs";
import { Request } from "express";

@Injectable()
export class BodyContextInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request: Request = context.switchToHttp().getRequest();

    if (request.body) {
      request.body._params = request.params;
      request.body._query = request.query;
    }

    return next.handle();
  }
}

export interface BodyContextDTO {
  _params: Record<string, any>;
  _query: Record<string, any>;
}
```

#### 4. Create TypeORM Entity `my.entity.ts`
```bash
import { Column, Entity, PrimaryGeneratedColumn } from "typeorm";

@Entity()
class MyEntity {
  
  @PrimaryGeneratedColumn()
  id: number;
  
  @Column({
    unique: true
  })
  name: string;
  
  ...
}
```

#### 5. Create DTO `dto/my.dto.ts`
```bash
import { Allow } from "class-validator";
import { IsUnique } from "./decorators/typeorm-validate.decorator";
import { BodyContextDTO } from "./interceptors/body-context.interceptor";

export class MyDTO implements BodyContextDTO {
  ...
  @IsUnique(Entity, 'name')
  name: string;
  
  @Allow()
  _params: Record<string, any>;

  @Allow()
  _query: Record<string, any>;
}
```

#### 6. Your controller `app.controller.ts`
```bash
import { Controller, Post, Body, UseInterceptors } from "@nestjs/common";
import { BodyContextInterceptor } from "./interceptors/body-context.interceptor";
import { UpdateEquivalentDto } from "./my.dto";

@Controller('app')
export class AppController {
  
  @Post()
  @UseInterceptors(BodyContextInterceptor)
  index(@Body() body: MyDto) {
    ...
  }
}
```

#### 7. Finish! Now field name is checking for unique using TypeORM!

### Extra settings:
```bash
@IsUnique(
    Entity,
    "field",
    MyDTO,
    (dto: MyDTO) => ({
      // dto._params - URL params
      // dto._query - Query params
      id: Not(dto._params["id"])
    }, {
      message: "MESSAGE ON ERROR"
    })
  )
```