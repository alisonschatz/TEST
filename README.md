# API RH Net - HML (coleção Bruno)

Coleção Bruno (formato **OpenCollection YAML**) reorganizada em arquitetura
limpa para testes manuais e documentação viva da API RH Net Social.

## Estrutura

```
rhnetsocial-api/
├── opencollection.yml            # config da coleção
├── environments/
│   └── HML.yml                   # baseUrl, authUrl, perfis e token
├── funcionario/
│   ├── folder.yml                # auth + login automático da pasta
│   ├── lista-admissoes-completas.yml
│   ├── cria-admissao-completa.yml
│   ├── atualiza-admissao-completa.yml
│   ├── remove-admissao-completa.yml
│   ├── lista-admissoes-preliminares.yml
│   ├── cria-admissoes-preliminares.yml
│   ├── atualiza-admissao-preliminar.yml
│   └── remove-admissao-preliminar.yml
└── liberacoes/
    ├── folder.yml                # auth + login automático da pasta
    ├── atualiza-liberacao-admissao-completa.yml
    ├── atualiza-liberacao-admissao-preliminar.yml
    └── atualiza-liberacao-servico-autonomo.yml
```

Nomes de arquivo usam `kebab-case`, sem espaços ou acentos (compatível com
qualquer SO/terminal/Git). O nome amigável, com espaços e acentos, continua
disponível dentro de cada arquivo em `info.name` — é ele que aparece na
interface do Bruno.

## O que mudou em relação à coleção original

1. **Autenticação centralizada por pasta, não por request.**
   Antes, o script de login (perfil fixo, chamada Basic Auth,
   `bru.setVar("token_atual", ...)`) estava **duplicado em 6 dos 11
   requests**, e os outros 5 dependiam de uma variável `{{token}}` que nunca
   era preenchida automaticamente. Agora o script mora **uma única vez** em
   cada `folder.yml` (`funcionario/folder.yml` e `liberacoes/folder.yml`) e
   roda antes de **qualquer** request da pasta.
   Resultado: **cada endpoint é 100% independente** — dá pra abrir qualquer
   request e mandar "Send" sem rodar nada antes, ideal para teste manual.

2. **Perfil de teste configurável só na environment.**
   O perfil usado (1 a 8) não está mais hardcoded dentro do script
   (`const perfil_escolhido = 2`). Ele vem de `perfil_escolhido` na
   environment `HML`. Trocar de perfil = trocar um valor, sem tocar em
   código.

3. **`auth: inherit` em todos os requests.**
   Nenhum request define header `Authorization` nem bloco `auth` próprio.
   Todos herdam o Bearer `{{token}}` definido no `folder.yml`, que é
   preenchido pelo script de login.

4. **Correção de bug:** o body padrão de `remove-admissao-completa.yml`
   tinha `"empresa_id": ,` (JSON inválido). Corrigido para um valor
   condizente com os exemplos do próprio request.

5. **Testes automáticos básicos em cada request** (`runtime.scripts` tipo
   `tests`), verificando status HTTP e o formato mínimo esperado da
   resposta (`sucesso`, `retorno`, `paginacao` quando aplicável). Isso torna
   a coleção também executável via `bru run` / CI, além de servir como
   documentação de contrato.

6. **`seq` renumerado** por pasta (1..N) para refletir a ordem lógica de uso
   (listar → criar → atualizar → remover), em vez dos números avulsos
   herdados do export original.

7. Os `examples:` (200/401/403/422) de cada request foram mantidos como
   estavam — são a documentação de contrato mais valiosa da coleção e não
   precisavam de alteração.

## Autenticação: matriz de tokens (sistema × contexto)

O login (Basic Auth → `POST /api/v1/auth/credencial/login`) usa dois tokens
combinados:

- **token de sistema** (`user` do Basic Auth): identifica o vínculo —
  `unico`, `practice` ou `parceiro`. São **3 valores fixos e imutáveis**.
- **token de contexto** (`pass` do Basic Auth): identifica o papel —
  cliente/empresa ou administrador/contabilidade. Único e Practice têm cada
  um o seu próprio par; **Parceiro reaproveita esses mesmos 4 tokens**
  (é um cliente externo, não tem par próprio). Ou seja, só **4 valores**,
  não 6.

Isso dá **7 tokens no total** (`environments/HML.yml` → `tokens_sistema` e
`tokens_contexto`), que são a fonte única da verdade para os 8 perfis —
antes cada perfil guardava seu próprio par duplicado, agora o script da
pasta monta a combinação certa em tempo de execução:

| # | Perfil | sistema | contexto |
|---|---|---|---|
| 1 | Único · Cliente/Empresa | `unico` | `cliente_unico` |
| 2 | Único · Administrador/Contabilidade | `unico` | `adm_unico` |
| 3 | Practice · Cliente/Empresa | `practice` | `cliente_practice` |
| 4 | Practice · Administrador/Contabilidade | `practice` | `adm_practice` |
| 5 | Parceiro (via Único) · Cliente/Empresa | `parceiro` | `cliente_unico` |
| 6 | Parceiro (via Único) · Administrador/Contabilidade | `parceiro` | `adm_unico` |
| 7 | Parceiro (via Practice) · Cliente/Empresa | `parceiro` | `cliente_practice` |
| 8 | Parceiro (via Practice) · Administrador/Contabilidade | `parceiro` | `adm_practice` |

Essa tabela vive no script `before-request` de cada `folder.yml` (não em
JSON na environment), porque é lógica de mapeamento, não segredo — só os
7 tokens em si são secretos.

## Como usar

1. Abra a pasta `rhnetsocial-api/` como coleção no Bruno.
2. Selecione a environment **HML**.
3. Os 7 tokens (`tokens_sistema` e `tokens_contexto`) já vêm preenchidos em
   `environments/HML.yml`, marcados como `secret: true` (o Bruno mascara o
   valor na interface). Se algum token de contexto for rotacionado, troque
   **só ele** ali — os tokens de sistema não mudam.
4. Ajuste `perfil_escolhido` (1 a 8, tabela acima) conforme o perfil que
   quiser usar nos testes.
5. Rode qualquer request da pasta `Funcionário` ou `Liberações`
   diretamente — o login acontece automaticamente antes da chamada.
6. **Nunca** versione `HML.yml` com tokens reais em um repositório
   compartilhado — se o time usa Git, mantenha-o fora do versionamento (ou
   use um cofre de segredos) e versione apenas um `HML.sample.yml` com os
   campos vazios.

## Referência rápida dos endpoints

| Pasta | Request | Método | Rota |
|---|---|---|---|
| Funcionário | Lista admissões completas | GET | `/api/v1/funcionario/completa` |
| Funcionário | Cria admissão completa | POST | `/api/v1/funcionario/completa` |
| Funcionário | Atualiza admissão completa (PUT) | PUT | `/api/v1/funcionario/completa` |
| Funcionário | Remove admissão completa | DELETE | `/api/v1/funcionario/completa` |
| Funcionário | Lista de admissões preliminares | GET | `/api/v1/funcionario/preliminar` |
| Funcionário | Cria/adiciona admissões preliminares | POST | `/api/v1/funcionario/preliminar` |
| Funcionário | Atualiza admissão preliminar | PUT | `/api/v1/funcionario/preliminar` |
| Funcionário | Remove admissão preliminar | DELETE | `/api/v1/funcionario/preliminar` |
| Liberações | Atualiza liberação da admissão completa | POST | `/api/v1/liberacao/completa` |
| Liberações | Atualiza liberação da admissão preliminar | POST | `/api/v1/liberacao/preliminar` |
| Liberações | Atualiza liberação de serviço autônomo | POST | `/api/v1/liberacao/autonomo/servico` |

Baseado no OpenAPI `docs_rhnetsocial.json` fornecido, que continua sendo a
fonte de verdade para regras de negócio, campos obrigatórios e enums.
