# Hash da senha com bcrypt

**Commit do gabarito:** [`361d0d7`](https://github.com/rocketseat-education/05-nest-clean/commit/361d0d764a422d9d74b5211bb4665b3f1ba0ae3a)
**Commit no nosso repo:** `9d76762`

## O problema

A aula passada salvou o usuário com a senha **em texto puro** no banco. Isso é uma das piores práticas possíveis em segurança:

- Se o banco vaza, todas as senhas vazam.
- Se o time tem acesso ao banco em produção (devs, DBAs), todo mundo lê senha de usuário.
- Pior: usuários reusam senhas em outros serviços. Vazou aqui, vazou tudo deles.

A regra é universal: **nunca, jamais, em hipótese alguma, salve senha em texto puro**. Salva-se sempre o **hash**.

## Hash ≠ criptografia

Importante distinguir:

- **Criptografia** é **reversível**: você pode descriptografar de volta com a chave certa.
- **Hash** é **uma via só**: dado o hash, é computacionalmente inviável recuperar o texto original.

Pra autenticar o usuário depois, você **não descriptografa** o hash salvo. Você **calcula o hash da senha que ele digitou** e compara com o salvo. Iguais → senha correta.

## A solução em uma frase

Instalar `bcryptjs`, e antes de salvar o usuário, substituir `password` por `hash(password, 8)`.

## Por que `bcryptjs` (e não `bcrypt`)

Existem duas bibliotecas com nomes parecidos:

- **`bcrypt`** — usa native bindings (C++). Mais rápido, mas exige `node-gyp`, build-essentials etc. Atrito pra rodar em ambientes mais limpos (CI, Docker slim).
- **`bcryptjs`** — implementação 100% JavaScript. Um pouco mais lento, mas **funciona em qualquer lugar sem build step**.

Pra um curso (e pra maioria das aplicações onde hash de senha não é hot path), `bcryptjs` ganha pela ausência de fricção.

## O que mudou no código

```ts
import { hash } from 'bcryptjs'

// ...dentro do handler, depois da checagem de email duplicado:
const hashedPassword = await hash(password, 8)

await this.prisma.user.create({
  data: {
    name,
    email,
    password: hashedPassword,   // ← antes era `password` cru
  },
})
```

E `@types/bcryptjs` foi instalado como devDependency (a lib não tem tipos embutidos).

## O parâmetro `8` — salt rounds

`hash(plain, saltRounds)`. Esse `8` é o **custo** (também chamado de "work factor"). Significa `2^8 = 256` iterações internas no algoritmo.

Trade-off:
- **Mais alto = mais seguro contra brute force**, mas mais lento.
- **Mais baixo = mais rápido**, mas mais fácil de quebrar.

Valores típicos: 10–12 em produção. **8** é razoável pra um curso/dev (rápido o suficiente pra testes não doerem). Quanto mais hardware do atacante evolui, mais alto esse número deve ser.

A escala é **exponencial**: 10 é 4x mais lento que 8. 12 é 16x mais lento. 

## "Mas o salt? Onde fica?"

Pergunta clássica. Em outras libs (`crypto.pbkdf2`, etc.) você gera um salt aleatório e salva separado do hash. **Com bcrypt, o salt vai junto no output**.

Olha o formato do que ficou no banco:

```
$2a$08$b5bl8m.//3utS4hlUK3a5uYcyCPXiRsPe3vsXN2SxfEDdfpw6cWUS
└┬┘ └┬┘ └────────────────────┬──────────────────────────────┘
 │   │                       │
 │   │                       └─ salt (22 chars) + hash (31 chars)
 │   └─ saltRounds (cost factor)
 └─ versão do algoritmo ($2a$, $2b$, $2y$ — bcryptjs usa $2a$)
```

Isso é prático: pra verificar uma senha, `bcrypt.compare(plain, hashSalvo)` extrai o cost e o salt do próprio hash, recalcula, e compara. Você nunca precisa gerenciar salt separadamente.

## Verificação testada

Demonstração rodada com a senha em claro `"senha-em-claro-123"`:

```
hash salvo:   $2a$08$b5bl8m.//3utS4hlUK3a5uYcyCPXiRsPe3vsXN2SxfEDdfpw6cWUS
compare("senha-em-claro-123"):   true   ✓
compare("errado"):               false  ✓
```

O hash não revela a senha, mas `compare` valida corretamente quem sabe o original.

## Pegadinhas pra lembrar

- **`hash` é async** — esquecer o `await` te dá um `Promise` indo pro banco como senha. Bug silencioso até alguém tentar logar.
- **Hashar duas vezes a mesma senha dá hashes diferentes** — porque o salt é aleatório. Por isso a comparação **sempre** se faz via `bcrypt.compare`, nunca via igualdade entre hashes.
- **Não tente "validar formato de senha" depois de hashar** — depois de hashada, virou string opaca de 60 chars. Validação (mínimo de caracteres, etc.) é no schema do Zod, antes de chegar aqui.
- **Migração de cost** — se um dia você decidir subir de 8 pra 12, hashes antigos continuam funcionando (o `compare` extrai o cost de dentro do próprio hash). Mas pra **forçar reupgrade**, você precisa rehashar quando o usuário logar — não tem como fazer batch só com o hash.

## Conexão com o que já tinha

- A aula anterior criou o controller. Esta tampa o buraco de segurança que aquela deixou.
- Próxima aula (`ZodValidationPipe`) vai validar **formato** da senha antes de chegar aqui — coisas como "mínimo 6 caracteres". Validação de **forma** é antes do hash; o hash é só persistência segura.
