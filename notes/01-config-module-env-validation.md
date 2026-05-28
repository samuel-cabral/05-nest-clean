# Validação de variáveis de ambiente com `@nestjs/config` + Zod

**Commit do gabarito:** [`e874303`](https://github.com/rocketseat-education/05-nest-clean/commit/e8743037b8789b13d975b081db3347bdd0676894)
**Commit no nosso repo:** `b91a29b`

## O problema

Antes desta aula, a aplicação rezava pra `process.env.DATABASE_URL` existir e estar bem formatada. Se faltasse ou estivesse errada, a gente só descobria **em runtime**, no meio de uma requisição, quando o Prisma tentasse conectar e estourasse um erro feio.

Variável de ambiente é **input externo**. Toda fronteira da aplicação que recebe input externo deveria ser validada — body de request a gente já validava com Zod (aula anterior, do `ZodValidationPipe`), o `process.env` não.

## A solução em uma frase

Plugar um schema Zod no `ConfigModule.forRoot({ validate })` do `@nestjs/config`. Se o `.env` não bate com o schema, a aplicação **não sobe**.

> **Princípio geral: falhe na inicialização, não em produção.**

## Peças

### 1. `src/env.ts` — o contrato

```ts
import { z } from 'zod'

export const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  PORT: z.coerce.number().optional().default(3333),
})

export type Env = z.infer<typeof envSchema>
```

Pontos não óbvios:

- **`.url()`** — `string()` aceita qualquer string. `.url()` exige formato de URL válido. Se alguém escrever `DATABASE_URL=banana`, Zod reprova.
- **`z.coerce.number()`** — esse é o **pulo do gato**. Variável de ambiente **sempre chega como string**, mesmo que pareça número. Sem o `.coerce`, o schema reprovaria `PORT=3333` porque o valor recebido é `"3333"`. Com `.coerce`, Zod converte **antes** de validar.
- **`.default(3333)`** — cobre o caso da variável não existir; só dispara depois do `.optional()`.
- **`z.infer<typeof envSchema>`** — extrai o tipo TypeScript do schema. É o que vai dar tipagem ao `ConfigService` lá no `main.ts`. Define uma vez, ganha tipo de graça.

### 2. `src/app.module.ts` — plugar no Nest

```ts
ConfigModule.forRoot({
  validate: (env) => envSchema.parse(env),
  isGlobal: true,
})
```

- **`validate`** é uma função que recebe `process.env` e devolve o objeto validado, ou **explode**. Como roda na inicialização, qualquer erro **impede a app de subir**.
- **`isGlobal: true`** — sem isso, todo módulo que quisesse injetar `ConfigService` precisaria reimportar o `ConfigModule`. Com `true`, fica disponível em qualquer lugar do app.

### 3. `src/main.ts` — ler a porta tipada

```ts
const configService = app.get<ConfigService<Env, true>>(ConfigService)
const port = configService.get('PORT', { infer: true })
```

Três detalhes:

- **`app.get<ConfigService<Env, true>>(ConfigService)`** — pega a instância do `ConfigService` do container de DI do Nest. O genérico `Env` diz "esse config representa essas variáveis"; o `true` ativa o **strict mode**, que faz o tipo de retorno ser não-nullable (você está afirmando que as variáveis existem — coerente, porque o `validate` já garantiu).
- **`configService.get('PORT', { infer: true })`** — o `'PORT'` é uma chave de `Env`; o `{ infer: true }` faz o TS inferir o tipo exato (`number`), em vez de retornar `unknown`.
- **Resultado:** autocomplete funciona, typo em chave é pego em compile time (`'DABASE_URL'` quebra build), e nada de cast manual.

## Por que Zod aqui (em vez de Joi, class-validator, etc.)

O projeto já usa Zod pra validar body de request. Reusar evita ter duas bibliotecas de validação fazendo a mesma coisa. E o `z.infer` casa bem com TS — você define schema **e** tipo no mesmo arquivo, sem duplicação.

A interface `validate` do `ConfigModule` é **agnóstica**: qualquer função `(env) => validated` serve. Você pode trocar Zod por outra coisa sem mexer no resto.

## Demonstração da validação em ação

Schema rodado isoladamente com inputs ruins:

```
Caso 1: DATABASE_URL inválida ('nao-eh-url')
  → ZodError: [{ validation: 'url', code: 'invalid_string', message: 'Invalid url', path: ['DATABASE_URL'] }]

Caso 2: DATABASE_URL ausente
  → ZodError: [{ code: 'invalid_type', expected: 'string', received: 'undefined', message: 'Required', path: ['DATABASE_URL'] }]

Caso 3: env válido com PORT='4000' (string)
  → parsed: { DATABASE_URL: '...', PORT: 4000 }
  → typeof parsed.PORT === 'number'   ← coerce em ação
```

Nos casos 1 e 2, a app não subiria. No caso 3, `PORT` chega como string e sai como number tipado.

## Pegadinhas pra lembrar

- **Esquecer `.coerce` em campos numéricos do env** — bug clássico. App não sobe e a mensagem do Zod fala em `expected number, received string`. Lembre que env é sempre string.
- **Não setar `isGlobal: true`** — depois você esquece, e fica tentando injetar `ConfigService` num módulo que não importou `ConfigModule`, e o Nest reclama em runtime que não conhece o provider.
- **Esquecer o `, true` no genérico `ConfigService<Env, true>`** — sem ele, todo `get` retorna `T | undefined`, e você acaba forçando `!` em todo lugar.
- **Misturar `validate` com `validationSchema`** — `validationSchema` é a opção legada baseada em Joi. A nova é `validate`, uma função qualquer. Use `validate`.

## Conexão com o que já tinha

- A aula do `ZodValidationPipe` ensinou a validar **body de request**. Esta aula aplica a mesma ferramenta em outra fronteira (env vars).
- Padrão: **toda entrada externa passa por Zod**. Body, env, e mais pra frente provavelmente query params, headers etc.
