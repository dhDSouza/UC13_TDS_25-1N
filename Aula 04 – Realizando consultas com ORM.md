# 🎓 **Aula 4 – Realizando consultas com ORM**

## 🎯 **Objetivos da Aula**

* Entender o que é um **ORM** e como o TypeORM facilita o trabalho com banco de dados.
* Comparar o uso de SQL manual (`mysql2/promise`) com ORM.
* Criar entidades e mapear para tabelas com **decorators**.
* Implementar **relacionamentos** (One-to-Many, Many-to-One) no TypeORM.
* Usar `relations` para simular **JOINs** automaticamente.

---

## 🧩 O que é um ORM?

ORM significa **Object-Relational Mapping** (*Mapeamento Objeto-Relacional*).
Basicamente, é uma ferramenta que permite interagir com o banco de dados **usando objetos e métodos** ao invés de escrever SQL puro.

💡 **Por que usar ORM?**

* 📉 Menos repetição de código SQL
* 🛡️ Menos risco de SQL Injection (ele faz o tratamento automático)
* 🗂️ Melhor organização do código
* 🔄 Portabilidade entre diferentes bancos de dados

## 🧠 **Como seria sem ORM**

* Usamos `mysql2/promise` para **escrever queries SQL diretamente**.
* Controlamos tudo: Desde o `SELECT` até as condições, paginação e `JOINs`.

Exemplo no **mysql2/promise**:

```ts
const [rows] = await connection.query(
  'SELECT * FROM usuarios WHERE id = ?',
  [id]
);
```

---

Agora, no **TypeORM**:

```ts
const user = await userRepository.findOneBy({ id });
```

💡 A diferença:

* No mysql2 → você escreve o SQL manualmente.
* No TypeORM → você descreve *o que quer*, e ele gera o SQL.

---

## 🏗️ **Configurando o TypeORM**

O `TypeORM` é um ORM para `JavaScript` e `TypeScript` que facilita a interação com bancos de dados relacionais de forma orientada a objetos, através do uso de decoradores.

## **🤔 Como iniciar um projeto com TypeORM?**

Instalar dependências:

```bash
npm init -y
npm install express dotenv mysql2 typeorm reflect-metadata
npm install -D typescript@5.9.3 ts-node-dev @types/express @types/node
```

Configurar transpilador do TypeScript:

```bash
npx tsc --init
```

Altere o coneúdo do arquivo `tsconfig.json`

```json
{
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist",
    "module": "nodenext",
    "target": "esnext",
    "strict": true,
    "skipLibCheck": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "strictPropertyInitialization": false
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules"]
}
```

> [!NOTE]
> <details>
> <summary>Explicação dos comandos adicionais</summary>
>    
> Esses dois comandos são essenciais quando trabalhamos com bibliotecas que usam **decorators** (como TypeORM, NestJS, class-validator etc.).
> 
> - **`experimentalDecorators`** → Habilita o uso da sintaxe de *decorators* no TypeScript (`@Algo`). Como decorators ainda não fazem parte oficial e final da especificação do JavaScript (estão em estágio avançado, mas não finalizado), o TypeScript trata como um recurso "experimental" e precisa dessa flag para permitir o uso.
> 
>   Exemplo:
>   ```ts
>   @Entity()
>   class User { ... }
>   ```
> 
> - **`emitDecoratorMetadata`** → Faz o TypeScript **emitir metadados adicionais** no JavaScript compilado sempre que encontrar um decorator. Esses metadados permitem que bibliotecas consigam ler informações de tipos em tempo de execução usando `Reflect.metadata`.
> 
>   Exemplo: o TypeORM consegue saber automaticamente que `name: string` é uma *string* por conta desses metadados.
> 
> - **`strictPropertyInitialization`** → O TypeScript exige que todo atributo da classe receba um valor, seja diretamente na declaração ou dentro do construtor. Isso evita que propriedades sejam acessadas como `undefined` sem querer.
> 
> 💡 **Em Resumo:**  
> Sem `experimentalDecorators`, você nem consegue usar `@decorator`.  
> Sem `emitDecoratorMetadata`, muitos frameworks que dependem de informações de tipos nos decorators simplesmente não funcionam corretamente.
> </details>
---

Arquivo `src/config/data-source.ts`:

```ts
import 'reflect-metadata';
import { DataSource } from 'typeorm';
import dotenv from "dotenv";

dotenv.config();

const { DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_DATABASE } = process.env;

export const AppDataSource = new DataSource({
    type: 'mysql',
    host: DB_HOST,
    port: Number(DB_PORT || "3306"),
    username: DB_USER,
    password: DB_PASSWORD,
    database: DB_DATABASE,
    entities: ['src/models/*.ts'],
    synchronize: true,
    logging: true
});
```

> [!CAUTION]
> `synchronize: true` Apenas em desenvolvimento (em ambiente de produção será false)

---

## 📦 **Criando Entidades**

O TypeORM usa **classes** para representar tabelas.

### `src/models/User.ts`

```ts
import { Entity, PrimaryGeneratedColumn, Column, OneToMany } from 'typeorm';
import { Post } from './Post';

@Entity('users') // Informa para o ORM que essa classe será uma Entidade do Banco de Dados
export class User {

    // Define que o campo será uma Chave Primária (PK) e Auto Incrementável (AI)
    @PrimaryGeneratedColumn() 
    id: number;

    // Define que o tamanho do campo é de 100 caracteres, e não pode ser nulo.
    @Column({ length: 100, nullable: false })
    name: string;

    // Define que o campo é Único (UK)
    @Column({ length: 100, unique: true })
    email: string;

    /*
        - Indica para o ORM que existe uma relação de 1 para Muitos (1:N) com a Entidade Posts.
        - Essa Relação será indicada da outra entidade também, e o ORM irá criar a Chave Estrangei (FK) automaticamente.
        - Essa prática é extremamente importante para que possam ser realizadas consultas em múltiplas tabelas posteriormente.
    */
    @OneToMany(() => Post, post => post.user)
    posts: Post[];
}
```

### `src/models/Post.ts`

```ts
import { Entity, PrimaryGeneratedColumn, Column, ManyToOne } from 'typeorm';
import { User } from './User';

@Entity('posts')
export class Post {
    
    @PrimaryGeneratedColumn()
    id: number;

    /*
        - Define o campo como sendo um VARCHAR.
        - Essa definição é opcional pois o ORM identifica pelo tipo da propriedade no TypeScript.
    */
    @Column({ type: "varchar", length: 100, nullable: false })
    title: string;


    /*
        - Indica para o ORM que existe uma relação de Muitos para 1 (N:1) com a Entidade Users.
        - Essa Relação foi indicada da outra entidade também, e o ORM irá criar a Chave Estrangeira (FK) automaticamente.
        - Sempre que ouver relacões entre entidades precisamos declarar a "ida e a volta".
        - Ou seja, se a relação entre Users e Posts for de 1:N a relação entre Posts e Users será de N:1.
        - Essa referência cruzada é obrigatória para que o ORM crie corretamente as Chaves Estrangeiras (FK)
    */
    @ManyToOne(() => User, user => user.posts)
    user: User;
}
```

### Diagrama de Entidade e Relacionamento (DER)

```mermaid
erDiagram
    USERS {
        id INT PK
        name VARCHAR(100)
        email VARCHAR(255) UK
    }
    POSTS {
        id INT PK
        title VARCHAR(100)
        user_id INT FK
    }

    USERS ||--o{ POSTS : "possui"
```

---

## 🔗 **Relacionamentos e JOINs**

No MySQL puro, um **INNER JOIN** ficaria assim:

```sql
SELECT u.*, p.*
FROM users u
INNER JOIN posts p ON p.user_id = u.id;
```

No TypeORM, a mesma coisa:

```ts
const users = await userRepository.find({
    relations: { posts: true }
});
```

* `relations` → diz quais tabelas devem ser carregadas junto.
* TypeORM **gera o JOIN automaticamente**.

---

## ⚙️ **Criando Services**

### `src/services/UserService.ts`

```ts
import { AppDataSource } from '../config/dataSource';
import { User } from '../models/User';

const userRepository = AppDataSource.getRepository(User);

export class UserService {
    async list() {
        return userRepository.find({
            relations: { posts: true },
            order: {
                id: 'ASC'
            }
        });
    }

    async show(id: number) {
        const user = await userRepository.findOne({
            where: { id },
            relations: { posts: true }
        });

        if (!user) {
            throw new Error('User not found');
        }

        return user;
    }

    async create(name: string, email: string) {
        if (!name || !email) {
            throw new Error('Name and email are required');
        }

        const exists = await userRepository.findOneBy({ email });

        if (exists) {
            throw new Error('Email already in use');
        }

        const user = userRepository.create({
            name,
            email
        });

        await userRepository.save(user)

        return user;
    }

    async update(id: number, name?: string, email?: string) {
        const user = await userRepository.findOneBy({ id });

        if (!user) {
            throw new Error('User not found');
        }

        if (name) {
            user.name = name;
        }

        if (email) {
            const exists = await userRepository.findOneBy({ email });

            if (exists && exists.id !== user.id) {
                throw new Error('Email already in use');
            }

            user.email = email;
        }

        await userRepository.save(user);

        return user;
    }

    async delete(id: number) {
        const user = await userRepository.findOneBy({ id });

        if (!user) {
            throw new Error('User not found');
        }

        await userRepository.remove(user);
    }
}
```

### `src/services/PostService.ts`

```ts
import { AppDataSource } from '../config/dataSource';
import { Post } from '../models/Post';
import { User } from '../models/User';

const postRepository = AppDataSource.getRepository(Post);
const userRepository = AppDataSource.getRepository(User);

export class PostService {
    async list() {
        return postRepository.find({
            relations: { user: true },
            order: { id: 'ASC' }
        });
    }

    async show(id: number) {
        const post = await postRepository.findOne({
            where: { id },
            relations: { user: true }
        });

        if (!post) {
            throw new Error('Post not found');
        }

        return post;
    }

    async create(title: string, userId: number) {
        if (!title || !userId) {
            throw new Error('Title and userId are required');
        }

        const user = await userRepository.findOneBy({ id: userId });

        if (!user) {
            throw new Error('User not found');
        }

        const post = postRepository.create({
            title,
            user
        });

        await postRepository.save(post);

        return post;
    }

    async update(id: number, title?: string, userId?: number) {
        const post = await postRepository.findOneBy({ id });

        if (!post) {
            throw new Error('Post not found');
        }

        if (title) {
            post.title = title;
        }

        if (userId) {
            const user = await userRepository.findOneBy({ id: userId });

            if (!user) {
                throw new Error('User not found');
            }

            post.user = user;
        }

        await postRepository.save(post);

        return post;
    }

    async delete(id: number) {
        const post = await postRepository.findOneBy({ id });

        if (!post) {
            throw new Error('Post not found');
        }

        await postRepository.remove(post);
    }
}
```

## 🚦 **Criando Controllers**

### `src/controllers/UserController.ts`

```ts
import { Request, Response } from 'express';
import { UserService } from '../services/UserService';

const userService = new UserService();

export class UserController {
    async list(req: Request, res: Response) {
        try {
            const users = await userService.list();
            return res.json(users);
        } catch {
            return res.status(500).json({ message: 'Internal server error' });
        }
    }

    async show(req: Request, res: Response) {
        try {
            const user = await userService.show(Number(req.params.id));
            return res.json(user);
        } catch (error: any) {
            if (error.message === 'User not found') {
                return res.status(404).json({ message: error.message });
            }

            return res.status(500).json({ message: 'Internal server error' });
        }
    }

    async create(req: Request, res: Response) {
        try {
            const { name, email } = req.body;

            const user = await userService.create(name, email);

            return res.status(201).json(user);
        } catch (error: any) {
            if (
                error.message === 'Name and email are required' ||
                error.message === 'Email already in use'
            ) {
                return res.status(400).json({ message: error.message });
            }

            return res.status(500).json({ message: 'Internal server error' });
        }
    }

    async update(req: Request, res: Response) {
        try {
            const { name, email } = req.body;

            const user = await userService.update(
                Number(req.params.id),
                name,
                email
            );

            return res.json(user);
        } catch (error: any) {
            if (
                error.message === 'User not found' ||
                error.message === 'Email already in use'
            ) {
                return res.status(400).json({ message: error.message });
            }

            return res.status(500).json({ message: 'Internal server error' });
        }
    }

    async delete(req: Request, res: Response) {
        try {
            await userService.delete(Number(req.params.id));

            return res.status(204).send();
        } catch (error: any) {
            if (error.message === 'User not found') {
                return res.status(404).json({ message: error.message });
            }

            return res.status(500).json({ message: 'Internal server error' });
        }
    }
}
```

### `src/controllers/PostController.ts`

```ts
import { Request, Response } from 'express';
import { PostService } from '../services/PostService';

const postService = new PostService();

export class PostController {
    async list(req: Request, res: Response) {
        try {
            return res.json(await postService.list());
        } catch {
            return res.status(500).json({ message: 'Internal server error' });
        }
    }

    async show(req: Request, res: Response) {
        try {
            return res.json(await postService.show(Number(req.params.id)));
        } catch (error: any) {
            if (error.message === 'Post not found') {
                return res.status(404).json({ message: error.message });
            }

            return res.status(500).json({ message: 'Internal server error' });
        }
    }

    async create(req: Request, res: Response) {
        try {
            const { title, userId } = req.body;

            const post = await postService.create(title, Number(userId));

            return res.status(201).json(post);
        } catch (error: any) {
            return res.status(400).json({ message: error.message });
        }
    }

    async update(req: Request, res: Response) {
        try {
            const { title, userId } = req.body;

            const post = await postService.update(
                Number(req.params.id),
                title,
                userId ? Number(userId) : undefined
            );

            return res.json(post);
        } catch (error: any) {
            return res.status(400).json({ message: error.message });
        }
    }

    async delete(req: Request, res: Response) {
        try {
            await postService.delete(Number(req.params.id));

            return res.status(204).send();
        } catch (error: any) {
            if (error.message === 'Post not found') {
                return res.status(404).json({ message: error.message });
            }

            return res.status(500).json({ message: 'Internal server error' });
        }
    }
}
```

---

## 🌐 **Rotas**

`src/routes/userRoutes.ts`

```ts
import { Router } from 'express';
import { UserController } from '../controllers/UserController';

const routes = Router();
const userController = new UserController();

// Rotas de Usuários
routes.get('/users', userController.list);          // Listar todos
routes.get('/users/:id', userController.show);      // Mostrar um
routes.post('/users', userController.create);       // Criar
routes.patch('/users/:id', userController.update);  // Atualizar
routes.delete('/users/:id', userController.delete); // Deletar

export default routes;
```

`src/routes/postRoutes.ts`

```ts
import { Router } from 'express';
import { PostController } from '../controllers/PostController';

const routes = Router();
const postController = new PostController();

// Rotas de Posts
routes.get('/posts', postController.list);          // Listar todos
routes.get('/posts/:id', postController.show);      // Mostrar um
routes.post('/posts', postController.create);       // Criar
routes.patch('/posts/:id', postController.update);  // Atualizar
routes.delete('/posts/:id', postController.delete); // Deletar

export default routes;
```

## 🚀 **Arquivo Principal `src/server.ts`**

```ts
import express, { Application } from "express";
import { AppDataSource } from "./config/data-source";
import userRoutes from "./routes/userRoutes";
import postRoutes from "./routes/postRoutes";

const app: Application = express();
const PORT: number = Number(process.env.PORT || "3000");

app.use(express.json());

// Utilizando as rotas na aplicação
app.use("/api", userRoutes);
app.use("/api", postRoutes);

// Inicializando conexão com o banco de dados
AppDataSource.initialize().then(() => {
    console.log("Database connected successfully!");
    app.listen(PORT, () => {
        console.log(`Server running on http://localhost:${PORT}`);
    });
}).catch((error) => {
    console.error("Error connecting to database.", error);
});
```

---

## 📝 **Exercícios Práticos**

1. Criar as entidades `Category` e `Product`.
2. Relacione `Category` com `Product` (One-to-Many).
3. Criar rota `/products` que já traga junto a categoria (usando `relations`).

---

## ✅ **Resumo da Aula**

* ORM = camada que converte **objetos/classe** em **tabelas**.
* TypeORM usa **decorators** para mapear tabelas.
* `relations` carrega dados de outras tabelas (JOIN).
* Relacionamentos no TypeORM:

  * `@OneToMany`
  * `@ManyToOne`

* Dá pra fazer tudo que fizemos com SQL puro, mas com menos código repetitivo.
