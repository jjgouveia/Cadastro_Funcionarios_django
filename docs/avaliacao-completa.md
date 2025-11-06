# Avaliação Completa – Cadastro_Funcionarios_django

Data: 06/11/2025
Avaliado: Everton Gabriel
Escopo: avaliação técnica completa considerando arquitetura, qualidade de código, segurança, documentação, testes e prontidão para evolução.

## 1) Visão geral da solução
- **Stack**: Django 5 + Django REST Framework
- **Autenticação**: JWT (simplejwt) com blacklist; serviço de autenticação dedicado (`AuthenticationService`)
- **Documentação**: drf-spectacular (Swagger) exposto em `/schema/swagger-ui/`
- **Apps**:
  - `contas`: usuário custom (`AbstractBaseUser`, `USERNAME_FIELD = 'email'`)
  - `auth`: endpoints de `signin` e `signup` (logout/usuário implementados mas não roteados)
  - `funcionarios`: CRUD de funcionários
  - `empresas`: CRUD de empresas

## 2) Pontos fortes
- **Modelo de usuário custom**: boa escolha para autenticação por e-mail e evolução futura.
- **Serviço de autenticação**: `AuthenticationService` separa regras de login/registro (checagem de senha, normalização, existência), favorecendo testes e reuso.
- **Documentação da API**: Swagger disponível e simples de acessar.
- **Paginação configurada**: uso de `PageNumberPagination` (lista de funcionários paginada por padrão).
- **Integridade de dados**:
  - `Funcionario`: índices em `cpf` e `matricula`, unicidade e FK para `Empresa`.
  - `Empresa`: unicidade de CNPJ.
- **Boas respostas**: status adequados em vários pontos (ex.: 201 em criação; 204 em deleção; 205 no logout).

## 3) Pontos fracos e riscos
### 3.1 Código/Modelos/Serializers
- **Divergência modelo × serializer (funcionários)**:
  - Modelo `Funcionario` não possui campo `nome`, mas o `FuncionarioSerializer` inclui `nome` nos `fields`.
  - Campo `usuario` (FK obrigatória) não consta no serializer — criar/atualizar podem falhar por falta de atribuição do usuário.
  - `extra_kwargs` usa `{'read_only': []}` (inválido). `read_only` espera boolean. Da maneira que está, não quebra nada, mas não vai haver efeito algum.
- **Validação de CPF**:
  - Serializer remove pontuação e valida tamanho 11 (ok), mas o README exige 14 (com pontuação). Documentação x implementação não batem.
- **Auth – avatar**:
  - Views e serializer referenciam `avatar`, porém o modelo `contas.User` não define esse campo. Isso quebra `UserView.patch` e `UserSerializer`.
  - Serializer monta URL com `settings.CURRENT_URL`, inexistente em `settings.py`.
- **Tratamento de erros ausente**:
  - `EmpresasUpdate/DeleteView` e `FuncionarioUpdate/DeleteView` usam `.get(pk=...)` sem `try/except` — 500 se não existir.

### 3.2 Configuração (settings/urls)
- **Segurança/ambiente**:
  - `SECRET_KEY` hardcoded.
  - Ausência de variáveis de ambiente e um local.env.
  - `DEBUG=True`, `ALLOWED_HOSTS=[]` — aceitável em dev, perigoso em prod.
  - Sem `MEDIA_URL`/`MEDIA_ROOT`, porém `UserView.patch` usa storage baseado em `settings.MEDIA_*`.
- **REST_FRAMEWORK duplicado**:
  - Definido três vezes, sobrescrevendo configurações anteriores (p.ex. paginação). Consolidar em um único bloco.
- **JWT**:
  - `SIMPLE_JWT` contém `CANCEL_TOKEN_LIFETIME` (chave não suportada).
- **I18N**:
  - `LANGUAGE_CODE='en-us'` e `TIME_ZONE='UTC'` destoam do contexto PT-BR.
- **URLs**:
  - `core/urls.py` inclui `auth/` duas vezes (`core/v1/auth/` e `auth/`), potencialmente duplicando rotas.
  - `auth/urls.py` expõe somente `signin` e `signup`. `SignOutView` e `UserView` existem mas não foram roteados.
- **Nomenclatura de app `auth`**:
  - Pode confundir com `django.contrib.auth`. Funciona, mas é fonte comum de erro/import equivocado.

### 3.3 Documentação e DX
- **README**:
  - Alguns comandos e exemplos com typos (ex.: `python py manage.py runserver`).
  - Especificações de CPF (formato com pontuação) não condizem com a validação aplicada (sem pontuação, 11 dígitos).
- **Requisitos**:
  - `requirements.txt` existe e está versionado. Há dependências possivelmente desnecessárias (ex.: `jsonschema*`, `attrs`) — revisar para reduzir superfície e tempo de instalação.
- **Testes**:
  - Não há testes automatizados visíveis para fluxos críticos (auth, CRUD, validações).

## 4) Aderência à senioridade (Júnior 1)
- **Adequações positivas**
  - Bom entendimento de conceitos principais (DRF, JWT, Swagger, usuário custom, paginação, unicidade/índices, services para auth).
  - Estrutura modular e rotas claras para CRUD.
- **Pontos a desenvolver**
  - Alinhamento entre modelos, serializers e documentação (campos, validações e respostas).
  - Configurações Django consolidadas e seguras (REST_FRAMEWORK único, JWT válido, MEDIA, i18n, variáveis de ambiente).
  - Tratamento de exceções e status HTTP consistentes.
  - Qualidade de entrega: testes mínimos, limpeza de dependências e revisão do README.

## 5) Dívidas técnicas principais
- Divergências críticas: `avatar` inexistente no modelo vs uso nas views/serializer; `nome` no serializer de funcionários.
- `REST_FRAMEWORK` definido múltiplas vezes; `SIMPLE_JWT` com chave inválida.
- Falta de `MEDIA_URL`/`MEDIA_ROOT` e de `CURRENT_URL` no `settings.py`.
- Falta de rotas para `logout` e `user` no `auth/urls.py`; duplicação de include de `auth/` em `core/urls.py`.
- Ausência de tratamento de `DoesNotExist` em updates/deletes.

## 6) Recomendações priorizadas
- **P0 – Consistência funcional (S–M)**
  - Ajustar `contas.User` ou remover referências a `avatar`. Se for manter avatar:
    - Adicionar `avatar = models.CharField(...)` ou `ImageField(upload_to='avatars/')`, criar `MEDIA_URL`/`MEDIA_ROOT` e migrations.
    - Definir `CURRENT_URL` via variável de ambiente e compor URL corretamente (ou usar `request.build_absolute_uri`).
  - Corrigir `FuncionarioSerializer`:
    - Remover `nome`; incluir `usuario` (ou definir no `perform_create`/view para assumir `request.user`).
    - Corrigir `extra_kwargs` para `{'read_only': True}` onde aplicável.
- **P1 – Settings e URLs (S)**
  - Consolidar `REST_FRAMEWORK` em um único bloco contendo: paginação, schema (spectacular), autenticação e permissões.
  - Remover `CANCEL_TOKEN_LIFETIME` de `SIMPLE_JWT`.
  - Adicionar `MEDIA_URL` e `MEDIA_ROOT`.
  - Unificar rotas de `auth` e expor `SignOutView` e `UserView` em `auth/urls.py`. Remover include duplicado em `core/urls.py`.
  - Considerar renomear app `auth` para `accounts_api` ou similar, evitando colisão conceitual.
- **P2 – Validações e erros (S)**
  - Padronizar validação de CPF: documentar que a API aceita CPF com/sem pontuação e normaliza para 11 dígitos.
  - Envolver `get(pk=...)` em `try/except` com retornos 404 coerentes em `empresas` e `funcionarios`.
  - Em `signup`, retornar 201 Created.
  - Em endpoints que exigem usuário, usar `permission_classes = [IsAuthenticated]` nas views correspondentes.
- **P3 – Segurança e ambiente (XS–S)**
  - Mover `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS` e `CURRENT_URL` para `.env`.
  - Definir `LANGUAGE_CODE='pt-br'` e `TIME_ZONE='America/Sao_Paulo'` para ambiente BR.
- **P3 – Documentação e limpeza (XS–S)**
  - Corrigir README (endpoints, exemplos, typos) e indicar Swagger.
  - Adicionar testes mínimos (auth: signin/signup/logout; funcionários: create/list/update/delete; validação de CPF).

## 7) Roadmap sugerido
1. P0: corrigir `avatar`/`MEDIA`/`CURRENT_URL` e `FuncionarioSerializer`/campos.
2. P1: consolidar `REST_FRAMEWORK`, JWT e URLs; adicionar `MEDIA_*`.
3. P2: tratamento de erros e validações, permissões explícitas e status 201 no `signup`.
4. P3: segurança via `.env`, i18n BR, limpeza de dependências, README e testes mínimos.

## 8) Conclusão
O projeto demonstra boa base para um Dev Júnior 1: usuário custom, DRF com JWT, documentação e paginação. As principais lacunas estão na consistência entre modelo/serializer/views, consolidação das configurações e tratamento de erros. Com as correções listadas, a API se torna mais robusta, previsível e pronta para evolução.

## 9) Legenda de Prioridades
- P0: Prioridade 0
- P1: Prioridade 1
- P2: Prioridade 2
- P3: Prioridade 3

## 10) Legenda de Complexidade
- XS: Complexidade Extra-Simples
- S: Complexidade Simples
- M: Complexidade Média
- L: Complexidade Livre
