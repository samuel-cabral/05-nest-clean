# Controller de criação de conta — primeira rota "de verdade"

**Commit do gabarito:** [`5cbdc99`](https://github.com/rocketseat-education/05-nest-clean/commit/5cbdc997e2e49a0e78addae566bab3f9aa9cc2ee)
**Commit no nosso repo:** `2ce22f9`

## O problema

Até esta aula o projeto tinha um `AppController` genérico com uma rota `GET /api/hello`. Funcional, mas **sem responsabilidade clara**. Numa aplicação real, à medida que rotas vão surgindo (criar conta, autenticar, listar perguntas, comentar...), jogar tudo num único `AppController` vira um arquivo gigante e impossível de manter.

A pergunta da aula é: **como organizar controllers**?

## A resposta do Diego

> "Um controller por rota."

Quase. A versão mais útil dessa regra: **um controller por feature/recurso**, e quando o controller começa a ficar grande demais ou misturar coisas distintas, quebra em mais de um. Na dúvida, prefira **um por rota** — é mais fácil juntar dois controllers depois do que separar um monstro de dez rotas.

Vantagens práticas:
- Arquivo pequeno = busca por nome no editor é direta (`Cmd+P → create-account` → achou).
- Mudança numa rota não toca código de outra (menos chance de quebrar o que não deveria).
- Padrão de nomenclatura `*.controller.ts` deixa óbvio o que é o quê.

## O que mudou no código

```
- src/app.controller.ts       (deletado)
- src/app.service.ts          (deletado)
+ src/controllers/create-account.controller.ts
```

E o `AppModule` passou a registrar `CreateAccountController` em vez de `AppController`.

## Dissecando o controller

```ts
@Controller('/accounts')
export class CreateAccountController {
  constructor(private prisma: PrismaService) {}

  @Post()
  @HttpCode(201)
  async handle(@Body() body: any) {
    const { name, email, password } = body

    const userWithSameEmail = await this.prisma.user.findUnique({
      where: { email },
    })

    if (userWithSameEmail) {
      throw new ConflictException(
        'User with same e-mail address already exists.',
      )
    }

    await this.prisma.user.create({
      data: { name, email, password },
    })
  }
}
```

Pontos não óbvios:

- **`@Controller('/accounts')`** — prefixo da rota. O `@Post()` sem argumento herda o prefixo, então o endpoint final é `POST /accounts`. Se quisesse `POST /accounts/admin` seria `@Post('/admin')`.
- **`@HttpCode(201)`** — por padrão `@Post` retorna 200. Mas a semântica HTTP correta para "criei recurso" é **201 Created**. O decorator sobrescreve.
- **Nome do método `handle`** — é só convenção. O Nest não liga; poderia ser `execute`, `create`, qualquer coisa. Convencionar `handle` em controllers que fazem uma coisa só ajuda na leitura.
- **`@Body() body: any`** — pega o JSON do request. O `any` é provisório (vai ser substituído pelo `ZodValidationPipe` na próxima aula).
- **Injeção do `PrismaService`** — pelo construtor. O Nest resolve a injeção porque `PrismaService` é provider do `AppModule`.

## Por que `ConflictException` em vez de retornar status manualmente

Você poderia fazer:

```ts
@Res() res: Response
// ...
return res.status(409).json({ message: '...' })
```

Mas isso quebra o fluxo de exceptions do Nest. Quando você lança uma `HttpException` (ou subclasse, como `ConflictException`):

- O Nest formata o body de erro de forma consistente em toda a aplicação.
- Filters de exceção (interceptors, loggers) capturam o erro automaticamente.
- Você não acopla o controller ao `Response` do Express — mais fácil migrar pra Fastify, testar, etc.

A regra prática: **erros de negócio = lança exception. Sucesso = retorna do método.** O Nest cuida do resto.

`ConflictException` mapeia pra HTTP **409 Conflict** — semântica correta pra "tentou criar algo que já existe".

## Resposta da rota

Repara que o método é `async` mas **não retorna nada**. Combinado com `@HttpCode(201)`, a resposta é `201` com **body vazio**. Isso é proposital aqui — criamos o usuário e ponto. Se quisesse devolver o usuário criado, seria `return user`.

## Pegadinhas pra lembrar

- **Esquecer o `@HttpCode(201)`** — endpoint retorna 200 por padrão; quem consome esperando 201 vai estranhar.
- **Tentar acessar `body.email` sem desestruturar primeiro** — funciona, mas vira fonte de typo silencioso (`body.eMail` não dá erro).
- **Salvar senha em texto puro** — sim, esta aula faz isso. **A próxima aula corrige** com bcrypt.
- **`@Body() body: any`** — provisório. A aula seguinte do `ZodValidationPipe` tipa isso direito.

## Conexão com o que já tinha

- Reusa o `PrismaService` que já existia (aula anterior, da configuração do Prisma).
- O `User` tem `email @unique` no schema do Prisma — então mesmo sem a checagem manual, um segundo `create` com mesmo email já estouraria erro do banco. Por que checar antes então? Porque o erro do banco é cru (`P2002 Unique constraint`), e a gente quer devolver um **409 com mensagem amigável** pro cliente. A checagem manual é a tradução do erro pra linguagem HTTP.

## Prepara o terreno pra...

- **Aula 02:** hash da senha (porque salvar texto puro é dado sensível).
- **Aula 03:** ZodValidationPipe (porque `@Body() body: any` é convite a bug).
