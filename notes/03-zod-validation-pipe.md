# Validação de body com Pipe customizado do Zod

**Commit do gabarito:** [`c063ee9`](https://github.com/rocketseat-education/05-nest-clean/commit/c063ee9711178fc6be89645329fe024ba52f24c7)
**Commit no nosso repo:** `a9ad6c1`

## O problema

A aula do controller deixou um débito grande:

```ts
async handle(@Body() body: any) {
  const { name, email, password } = body
  // ...
}
```

`any` é **renúncia de tipo** — o TS não te avisa de nada, e em runtime nada garante que `name`, `email` e `password` chegaram, muito menos no formato certo. Cenários ruins possíveis:

- Cliente manda `email: 42` → o Prisma reclama lá em baixo com erro genérico.
- Cliente esquece `password` → erro de unique constraint estranho, ou pior, salva `null` e quebra o login depois.
- Cliente manda email mal-formatado → criamos o usuário e descobrimos no primeiro fluxo de "esqueci minha senha".

A solução é o conceito de **Pipe** do Nest.

## O que é um Pipe

Pipe é um objeto que **intercepta o valor** antes dele chegar no handler do controller. Tem duas responsabilidades clássicas:

1. **Transformar** — ex: converter `"42"` (string da query) em `42` (number).
2. **Validar** — ex: reprovar valor inválido com `BadRequestException`.

A interface tem um método só:

```ts
interface PipeTransform {
  transform(value: unknown, metadata: ArgumentMetadata): any
}
```

Você implementa, o Nest chama. Se o método retorna, o valor (possivelmente transformado) segue pro handler. Se ele lança, o Nest converte a exceção em resposta HTTP.

## A solução em uma frase

Criar um pipe genérico que recebe **qualquer schema Zod** e aplica `.parse()` no body. Se o parse falhar, lança `BadRequestException` com a lista de erros formatada.

## Peças

### 1. `src/pipes/zod-validation-pipe.ts` — o pipe genérico

```ts
import { PipeTransform, BadRequestException } from '@nestjs/common'
import { ZodError, ZodSchema } from 'zod'
import { fromZodError } from 'zod-validation-error'

export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown) {
    try {
      this.schema.parse(value)
    } catch (error) {
      if (error instanceof ZodError) {
        throw new BadRequestException({
          message: 'Validation failed',
          statusCode: 400,
          errors: fromZodError(error),
        })
      }
      throw new BadRequestException('Validation failed')
    }
    return value
  }
}
```

Pontos não óbvios:

- **`constructor(private schema: ZodSchema)`** — o pipe **recebe o schema na instanciação**. Por isso ele não é `@Injectable()` nem provider — é instanciado manualmente no `@UsePipes(new ZodValidationPipe(schema))`.
- **`schema.parse(value)`** — método do Zod que valida e retorna o valor parseado. Se inválido, lança `ZodError`.
- **`fromZodError(error)`** — vem de `zod-validation-error`. O erro cru do Zod é uma estrutura aninhada com `issues[]` técnicos. `fromZodError` formata isso numa mensagem amigável (e mantém os `details` se você quiser ler).
- **`throw new BadRequestException(...)`** — mapeia pra HTTP **400**. Passar um objeto em vez de string controla o body de resposta.
- **`return value`** — repara que retornamos o `value` original, não o parseado. Pra esse caso (só validando, sem transformar), tanto faz. Se o schema tivesse `.transform()` no meio, aí seria importante retornar `this.schema.parse(value)` pra que a transformação chegasse no handler.

### 2. Aplicação no controller

```ts
const createAccountBodySchema = z.object({
  name: z.string(),
  email: z.string().email(),
  password: z.string(),
})

type CreateAccountBodySchema = z.infer<typeof createAccountBodySchema>

@Controller('/accounts')
export class CreateAccountController {
  // ...
  @Post()
  @HttpCode(201)
  @UsePipes(new ZodValidationPipe(createAccountBodySchema))
  async handle(@Body() body: CreateAccountBodySchema) {
    // body agora é tipado!
  }
}
```

Pontos:
- **`@UsePipes` no handler** — aplica o pipe a todos os parâmetros decorados desse handler. Equivalente alternativo seria `@Body(new ZodValidationPipe(...))` direto no parâmetro.
- **`z.infer<typeof schema>`** — gera o tipo TS a partir do schema. **Schema é a única fonte da verdade**: você muda lá, o tipo acompanha. Zero duplicação.
- **Schema fora do controller** — declarado em const no topo do arquivo. Mantém o handler enxuto e permite exportar/reusar se precisar.

## Comportamento testado

Mesmo body, diferentes inputs:

```
body válido                                     → 201
{ email: "not-an-email", ... }                   → 400 + { code: invalid_string, validation: email }
{ email: "x@y.com" }  (sem name)                 → 400 + { code: invalid_type, message: Required }
{ password: 12345 }   (number em vez de string) → 400 + { code: invalid_type, received: number }
```

Os erros são **estruturados** — frontend pode parsear `errors.details[]` pra destacar o campo certo.

## Por que esse padrão é tão bom

- **Reuso total**: o `ZodValidationPipe` serve pra **qualquer** schema. Próximo controller? `@UsePipes(new ZodValidationPipe(outroSchema))`. Não precisa criar um pipe por endpoint.
- **Schema como contrato**: o schema é doc + validação + tipo, tudo num lugar só.
- **Sem repetição entre runtime e compile time**: TS confia no Zod, Zod confia em si mesmo. Você nunca tem um `interface Body { email: string }` separado do `z.object({ email: z.string() })` arriscando ficar dessincronizado.

## Pegadinhas pra lembrar

- **Esquecer `@UsePipes`** — controller compila, app sobe, mas a validação **não roda**. O `@Body() body: CreateAccountBodySchema` te dá tipo em compile time, mas em runtime continua sendo `any`. Sempre pareie schema com pipe.
- **`fromZodError` precisa ser importado** — fácil esquecer e cair no fallback genérico `BadRequestException('Validation failed')`.
- **`@Injectable()` no pipe** — não coloque. Se botar, o Nest tenta resolver via DI, e como ele exige `schema` no construtor, dá pau. Pipes instanciados manualmente ficam fora do DI.
- **Schema `.email()` é estrito demais às vezes** — não aceita certas formas válidas de RFC. Pra produção, considere `.email().toLowerCase()` ou um regex próprio dependendo do caso de uso.
- **Validar `password` apenas como `z.string()`** — sem mínimo de caracteres. Vai querer adicionar `.min(6)` ou similar em algum momento. Aqui está cru pra ficar focado no padrão.

## Conexão com o que já tinha

- Substitui o `body: any` herdado da aula 01.
- Roda **antes** do hash da aula 02 — se o body é inválido, nem chegamos a tentar hashar.
- O conceito de "validar entrada externa com Zod" será generalizado na aula 04 pra **variáveis de ambiente**. Mesma ideia, fronteira diferente.

## Prepara o terreno pra...

- **Aula 04:** validar `process.env` com Zod via `ConfigModule.validate`. O padrão "Zod nas fronteiras" estabelece-se aqui e se replica.
- **Aulas futuras** onde outros controllers precisarem validar body — basta declarar o schema e usar o mesmo `ZodValidationPipe`.
