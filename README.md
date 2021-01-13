# ts-taskManager

[강의 링크](https://www.youtube.com/watch?v=IefCGB5gNgY&list=PLs1waz0ZKTGP_e-dgbhjP1oTOuFZdLwUy)

## Skills

- **프론트엔드** : Typescript, React, MaterialUI
- **백엔드** : Node.js, Nest.js, TypeORM

## 1. 초기세팅

### 1) 도커 컨테이너, 데이터베이스 생성

[Docker Compose File](https://github.com/hademo/task-management-api/blob/master/docs/task-management-compose.yml)<br>
`$ sudo snap install docker`
`$ sudo apt install docker-compose`
`$ docker-compose -f task-management-compose.yml up -d`
`$ docker ps`

```js
// adminer : localhost:8888
System: PostgreSQL;
Server: db;
Username: postgres;
Password: 123456;
```

### 2) NestJS 프로젝트 시작, 디펜던시 다운

[NestJS](https://docs.nestjs.com/)
`$ yarn global add @nestjs/cli typeorm`
`$ nest new '프로젝트' (yarn)`

- PostgreSQL : 오픈 소스 객체-관계형 데이터베이스 시스템(ORDBMS)

[DB TypeORM](https://docs.nestjs.com/techniques/database)
`$ npm install --save @nestjs/typeorm pg`

### 3) ORM Config

[ORM Config](https://github.com/hademo/task-management-api/blob/master/docs/ormconfig.ts)

```js
// src/ormconfig.ts

const config: ConnectionOptions = {
  username: "postgres",
  password: "123456",
  database: "taskManager",
};
```

### 4) 데이터베이스 모듈 생성

`$ nest g module database`

```js
// src/database/database.module.ts

import { Module } from "@nestjs/common";
import { TypeOrmModule } from "@nestjs/typeorm";
import * as ormconfig from "../ormconfig";

@Module({ imports: [TypeOrmModule.forRoot(ormconfig)] })
export class DatabaseModule {}
```

[TypeORM Scripts](https://github.com/hademo/task-management-api/blob/master/docs/typeorm.json)

```js
// package.json

"scripts": {
    "typeorm": "ts-node -r tsconfig-paths/register ./node_modules/typeorm/cli.js --config src/ormconfig.ts",
    "typeorm:migrate": "yarn typeorm migration:generate -n",
    "typeorm:run": "yarn typeorm migration:run",
    "typeorm:revert": "yarn typeorm migration:revert"
  }
```

### 5) 마이그레이션

`$ yarn typeorm:migrate Init`
`src/migrations`가 생성이 된다.

## 2. REST CRUD API

### 1) POST API

### 데이터베이스에 Task 테이블 생성

```js
// src/entity/task.entity.ts

import { Entity, PrimaryGeneratedColumn, Column } from "typeorm";

export enum TaskStatus {
    Created = 0,
    InProgress = 1,
    Done = 2
}

@Entity()
export class Task {
    @PrimaryGeneratedColumn()
    id: number;

    @Column({ nullable: true, length: 64 })
    title: string;

    @Column({ nullable: true, length: 1024 })
    description: string;

    @Column({ nullable: false, default: TaskStatus.Created })
    status: TaskStatus;
}
```

`$ yarn typeorm:migrate AddTask`<br>
`$ yarn typeorm:run`

### Task 모듈 생성

`$ nest generate module task`<br>
`src/task/task.module.ts`<br>

`$ nest g controller task`<br>
`src/task/task.controller.ts`<br>
`src/task/task.controller.spec.ts`<br>

`$ nest g service task`<br>
`src/task/task.service.ts`<br>
`src/task/task.service.spec.ts`

```js
// src/task/task.module.ts

import { Task } from "../entity/task.entity";

@Module({
  imports: [TypeOrmModule.forFeature([Task])],
})
export class TaskModule {}
```

```js
// src/task/task.service.ts

import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { TaskDTO } from 'src/dto/task.dto';
import { Task, TaskStatus } from 'src/entity/task.entity';
import { Repository } from 'typeorm';
import { CreateTaskDTO } from '../dto/create-task.dto'

@Injectable()
export class TaskService {
    constructor(@InjectRepository(Task) private taskRepository: Repository<Task>) {}

    public async createOne(createTaskRequest: CreateTaskDTO) {
        const task: Task = new Task();
        task.title = createTaskRequest.title;
        task.description = createTaskRequest.description;
        task.status = TaskStatus.Created;

        // DB에 저장
        await this.taskRepository.save(task);

        const taskDTO = new TaskDTO();
        taskDTO.id = task.id;
        taskDTO.title = task.title;
        taskDTO.description = task.description;
        taskDTO.status = task.status;

        return taskDTO;
    }
}
```

```js
// src/dto/create-task.dto.ts

export class CreateTaskDTO {
    title: string;
    description?: string;
}
```

```js
// src/dto/task.dto.ts

import { TaskStatus } from "src/entity/task.entity";

export class TaskDTO {
  id: number;
  title: string;
  description: string;
  status: TaskStatus;
}
```

```js
// src/task/task.controller.ts

import { Controller, Body, Post } from '@nestjs/common';
import { CreateTaskDTO } from 'src/dto/create-task.dto';
import { TaskService } from './task.service';

@Controller('tasks')
export class TaskController {

    constructor(private readonly taskService: TaskService) {}

    @Post()
    public async createOne(@Body() createTaskRequest: CreateTaskDTO) {
        const resp = await this.taskService.createOne(createTaskRequest);

        return resp;
    }

}
```

`$ yarn add rxjs`<br>
`$ yarn start:dev`<br>
`dist`폴더가 생성된다.<br>
콘솔확인 : `Mapped {/tasks, POST} route`

### 테스트

- 요청 : POST 요청(JSON) : `http://localhost:3000/tasks`

```js
{
  "title": "Task #1",
  "description": "My first task"
}

{
	"title": "Task #2",
	"description": "My Second task"
}
```

- 응답

```js
{
  "id": 1,
  "title": "Task #1",
  "description": "My first task",
  "status": 0
}

{
  "id": 2,
  "title": "Task #2",
  "description": "My Second task",
  "status": 0
}
```

### 2) GET API

```js
// src/task/task.service.ts

import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { TaskDTO } from 'src/dto/task.dto';
import { Task, TaskStatus } from 'src/entity/task.entity';
import { Repository } from 'typeorm';
import { CreateTaskDTO } from '../dto/create-task.dto'

@Injectable()
export class TaskService {
    constructor(@InjectRepository(Task) private taskRepository: Repository<Task>) {}

    public async createOne(createTaskRequest: CreateTaskDTO) {
        const task: Task = new Task();
        task.title = createTaskRequest.title;
        task.description = createTaskRequest.description;
        task.status = TaskStatus.Created;

        // DB 저장
        await this.taskRepository.save(task);

        const taskDTO = this.entityToDTO(task);

        return taskDTO;
    }

    private entityToDTO(task: Task): TaskDTO {
        const taskDTO = new TaskDTO();
        taskDTO.id = task.id;
        taskDTO.title = task.title;
        taskDTO.description = task.description;
        taskDTO.status = task.status;

        return taskDTO;
    }

    public async getAll() {
        const tasks: Task[] = await this.taskRepository.find();
        const tasksDTO: TaskDTO[] = tasks.map(x => this.entityToDTO(x));

        return tasksDTO;
    }
}
```

```js
// src/task/task.controller.ts

import { Controller, Get } from '@nestjs/common';
import { TaskService } from './task.service';

@Controller('tasks')
export class TaskController {

    constructor(private readonly taskService: TaskService) {}

    @Get()
    public async getAll() {
        const resp = await this.taskService.getAll();

        return resp;
    }

}
```

### 테스트

- 요청 : GET요청(JSON) `http://localhost:3000/tasks`
- 응답

```js
[
  {
    id: 1,
    title: "Task #1",
    description: "My first task",
    status: 0,
  },
  {
    id: 2,
    title: "Task #2",
    description: "My Second task",
    status: 0,
  },
];
```

### 3) GET/:id API

```js
// src/task/task.service.ts

import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { TaskDTO } from 'src/dto/task.dto';
import { Task } from 'src/entity/task.entity';
import { Repository } from 'typeorm';

@Injectable()
export class TaskService {

    constructor(@InjectRepository(Task) private taskRepository: Repository<Task>) {}

    public async getOne(taskId: number) {
        const task: Task = await this.taskRepository.findOne(taskId);

        if(!task) throw new NotFoundException(`Task with the id ${taskId} was not found`)

        const taskDTO: TaskDTO = this.entityToDTO(task);

        return taskDTO;
    }

}
```

```js
// src/task/task.controller.ts

import { Controller, Get, Param } from '@nestjs/common';
import { TaskService } from './task.service';

@Controller('tasks')
export class TaskController {

    constructor(private readonly taskService: TaskService) {}

    @Get('/:id')
    public async getOne(@Param('id') taskId: number) {
        const resp = await this.taskService.getOne(taskId);

        return resp;
    }

}
```

### 테스트

- 요청 : GET요청(JSON) `http://localhost:3000/tasks/1`
- 응답

```js
{
  "id": 1,
  "title": "Task #1",
  "description": "My first task",
  "status": 0
}
```

- 요청 : GET요청(JSON) `http://localhost:3000/tasks/5`
- 응답

```js
{
  "statusCode": 404,
  "message": "Task with the id 5 was not found",
  "error": "Not Found"
}
```

### 4) PUT API

```js
// src/task/task.service.ts

import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { TaskDTO } from 'src/dto/task.dto';
import { UpdateTaskDTO } from 'src/dto/update-task.dto';
import { Task } from 'src/entity/task.entity';
import { Repository } from 'typeorm';

@Injectable()
export class TaskService {

    constructor(@InjectRepository(Task) private taskRepository: Repository<Task>) {}

    public async updateOne(taskId: number, updateTaskRequest: UpdateTaskDTO) {
        // fetch and check if the task exists
        const task: Task = await this.getOne(taskId);

        // check which properties are set in dto
        task.title = updateTaskRequest.title || task.title;
        task.description = updateTaskRequest.description || task.description;
        task.status = updateTaskRequest.status || task.status;

        // update the properties on the task
        await this.taskRepository.save(task);

        // return the task as a dto
        const taskDTO: TaskDTO = this.entityToDTO(task);

        return taskDTO;
    }
}
```

```js
// src/dto/update-task.dto.ts

import { TaskStatus } from "src/entity/task.entity";

export class UpdateTaskDTO {
    title?: string;
    description?: string;
    status?: TaskStatus;
}
```

```js
// src/task/task.controller.ts

import { Controller, Body, Param, Put } from '@nestjs/common';
import { UpdateTaskDTO } from 'src/dto/update-task.dto';
import { TaskService } from './task.service';

@Controller('tasks')
export class TaskController {

    constructor(private readonly taskService: TaskService) {}

    @Put('/:id')
    public async updateOne(@Param('id') taskId: number, @Body() updateTaskRequest: UpdateTaskDTO) {
        const resp = await this.taskService.updateOne(taskId, updateTaskRequest);

        return resp;
    }

}
```

### 테스트

- 요청 : PUT 요청(JSON) `http://localhost:3000/tasks/1`

```js
{
  "title": "UPDATE Task #1",
  "description": "UPDATE My Second task",
  "status": 1
}
```

- 응답

```js
{
  "id": 1,
  "title": "UPDATE Task #1",
  "description": "UPDATE My Second task",
  "status": 1
}
```

### 5) DELETE API

```js
// src/task/task.service.ts

import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Task  } from 'src/entity/task.entity';
import { Repository } from 'typeorm';

@Injectable()
export class TaskService {

    constructor(@InjectRepository(Task) private taskRepository: Repository<Task>) {}

    public async deleteOne(taskId: number) {
        // fetch and check if the task exists
        const task: Task = await this.getOne(taskId);

        // delete the task
        await this.taskRepository.remove(task);
    }
}
```

```js
// src/task/task.controller.ts

import { Controller, Param, Delete, HttpCode, HttpStatus } from '@nestjs/common';
import { TaskService } from './task.service';

@Controller('tasks')
export class TaskController {

    constructor(private readonly taskService: TaskService) {}

    @Delete('/:id')
    @HttpCode(HttpStatus.NO_CONTENT)
    public async deleteOne(@Param('id') taskId: number) {
        await this.taskService.deleteOne(taskId);
    }
}
```

### 테스트

- 요청 : DELETE 요청(JSON) `http://localhost:3000/tasks/1`

- 확인 : GET 요청(JSON) `http://localhost:3000/tasks`

```js
[
  {
    id: 2,
    title: "Task #2",
    description: "My Second task",
    status: 0,
  },
];
```

## 3. 프론트엔드
