# api-design-reviewer

🇺🇸 [Read in English](README.md)

Uma skill para o Claude que revisa o design de APIs REST/HTTP como faria um revisor sênior: encontra os problemas que realmente prejudicam clientes e operação, explica por que cada um importa e mostra a correção no nível do contrato. Por isso, funciona para qualquer linguagem ou framework.

Linters verificam sintaxe ("falta descrição", "operationId duplicado"). Esta skill pega os problemas que exigem julgamento:

- `POST /payments` sem chave de idempotência **cobra o cliente duas vezes** quando a rede falha e o cliente reenvia.
- `GET /invoices/{id}` que só verifica o login deixa qualquer usuário **ler a fatura de outra pessoa** trocando o id.
- Paginação por offset numa tabela movimentada **pula e repete registros** enquanto o usuário navega pelas páginas.
- `"amount": 10.1` como float faz os **totais errarem por centavos** e quebra a conciliação.

## O que ela verifica

32 regras em 9 categorias, cada uma baseada numa RFC, no OWASP API Security Top 10 ou num guia de APIs amplamente usado.

| Categoria | O que procura |
|---|---|
| **SEC** Segurança | Falta de autorização por objeto (BOLA), mass assignment, ausência de rate limit, dados sensíveis na URL, autenticação não declarada |
| **IDM** Idempotência | Pagamentos, pedidos e mensagens que podem ser executados duas vezes num reenvio |
| **CON** Concorrência | Atualizações perdidas quando dois usuários editam o mesmo registro (ETag / If-Match) |
| **ASY** Operações assíncronas | Processamentos longos que deveriam retornar `202 Accepted` |
| **STS** Status codes | `200` com erro no corpo, `201` sem `Location`, `401` vs `403`, `400` vs `409` vs `422` |
| **ERR** Erros | Problem Details (RFC 9457), stack traces vazando, erros de validação por campo |
| **MET** Métodos | `GET` com efeito colateral, `PUT` usado como atualização parcial, `GET` com corpo |
| **PAG** Paginação | Listas sem limite, offset vs cursor, contratos inconsistentes |
| **DAT** Tipos de dados | Dinheiro como float, datas ambíguas, ids grandes perdendo precisão no JavaScript, enums numéricos |
| **VER** Versionamento | Falta de estratégia, breaking changes sem `Deprecation` / `Sunset` |
| **NAM** Nomes | Verbos nas rotas, casing inconsistente, aninhamento profundo |

O catálogo completo está em [references/rules.md](references/rules.md) (em inglês). Ele também serve como guia de estudo sobre design de APIs HTTP.

## Exemplo de achado

````markdown
### 🔴 IDM-001 · Criação de pagamento sem chave de idempotência
**Onde:** `POST /payments`
**Confiança:** Confirmado

**Problema:** se o cliente der timeout depois que o servidor processou o
pagamento e reenviar a requisição, o cliente final é cobrado duas vezes.

**Antes**
```
POST /payments
{ "amount": 150.0 }
```

**Depois**
```
POST /payments
Idempotency-Key: 8e03978e-40d5-43e8-bc93-6894a57f9324
{ "amount": "150.00", "currency": "BRL" }
```

**Breaking change:** Não (o header pode começar opcional)
**Fonte:** IETF draft-ietf-httpapi-idempotency-key-header; RFC 9110 §9.2.2
````

Toda revisão começa com um resumo por severidade, destaca o que a API já faz bem, lista o que não pôde ser revisado e termina com uma ordem sugerida de correção.

## Instalação

**Claude Code**, com a [CLI de skills](https://github.com/vercel-labs/skills):

```bash
npx skills add pedroCifoni/api-design-reviewer
```

Ou clone na sua pasta pessoal de skills:

```bash
git clone https://github.com/pedroCifoni/api-design-reviewer.git ~/.claude/skills/api-design-reviewer
```

**Claude.ai / Claude Desktop:** baixe este repositório como ZIP e envie como skill personalizada nas configurações do Claude.

## Como usar

Entregue ao Claude qualquer um destes e peça uma revisão:

- um arquivo **OpenAPI / Swagger** (YAML ou JSON)
- **código de rotas ou controllers** em qualquer linguagem (Express, ASP.NET Core, Spring, FastAPI, Laravel, Go...)
- uma **proposta escrita** de uma API que você ainda está desenhando
- uma **pergunta pontual** sobre uma decisão

Exemplos de prompt:

```text
Revise o design da API no openapi.yaml.
```

```text
Revise os endpoints em src/controllers antes de eu abrir o PR.
```

```text
Estou desenhando uma API de agendamento: POST /bookings, GET /bookings?userId=,
PUT /bookings/{id} para trocar a data. Que problemas você vê?
```

```text
Esse endpoint deve retornar 403 ou 404 quando o pedido não é do usuário?
```

O relatório vem no seu idioma. IDs das regras, termos HTTP e status codes ficam em inglês.

## Como a revisão funciona

1. **Monta um inventário das rotas:** método, caminho, entradas, respostas e autenticação de cada operação. A partir do código, também anota se cada busca é filtrada pelo usuário atual.
2. **Entende o contexto:** o domínio e os consumidores definem a severidade. Falta de idempotência é crítica em `/payments` e menor em `/feedback`.
3. **Respeita suas convenções:** aponta inconsistências, não escolhas que apenas diferem das preferências dela.
4. **Aplica as regras**, começando por segurança.
5. **Calibra os achados:** cada achado aponta a evidência e vem marcado como **Confirmado** ou **Verificar** (por exemplo, uma checagem de dono do recurso não aparece só no spec). Problemas repetidos são agrupados, e correções que quebrariam clientes existentes vêm com um caminho de migração.

Ela não inventa problemas para encher o relatório. Um relatório curto sobre uma API boa é um bom resultado.

## Fontes

- [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110): HTTP Semantics
- [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457): Problem Details for HTTP APIs
- [OWASP API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [Rascunho IETF do header Idempotency-Key](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)
- RFC 3339, 5789, 6585, 6902, 7396, 8259, 8288, 8594, 9745
- [Google API Improvement Proposals](https://google.aip.dev/), [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines), [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/)

As RFCs definem o protocolo. Os guias são convenções, e a skill os apresenta dessa forma.

## Roadmap

- [ ] **Evals:** APIs com defeitos plantados e um gabarito, comparando o Claude com e sem a skill
- [ ] **Modo Design:** gerar um contrato OpenAPI completo a partir de requisitos
- [ ] **Modo Diff:** detectar breaking changes entre duas versões de um spec

## Contribuindo

Issues e pull requests são bem-vindos, principalmente:

- uma regra que você acha que está faltando, com a fonte,
- uma revisão em que a skill errou (cole a entrada e o achado),
- exemplos de erros reais de design de API.

## Licença

[MIT](LICENSE) © Pedro Cifoni Montezuma
