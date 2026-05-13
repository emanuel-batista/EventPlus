# Sistema de Gestão de Eventos de Grife
### Disciplina: Sistemas Operacionais | Atividade Prática — Containers com Docker

---

## Visão Geral do Projeto

Inspirado no Met Gala, o **Sistema de Gestão de Eventos de Grife** é uma plataforma distribuída para organizar eventos de moda de alto padrão. O sistema conecta os diferentes atores do evento — produtores, empresários, agentes e celebridades — permitindo o gerenciamento de pessoas, eventos e credenciamento/convites de forma integrada.

---

## Arquitetura de Microsserviços

O sistema é composto por **5 containers**, distribuídos em 3 APIs Python e 2 bancos de dados:

```
sistema-grife/
│
├── api-pessoas/          ← Pessoa 2
│   ├── app.py
│   └── Dockerfile
│
├── api-eventos/          ← Pessoa 3
│   ├── app.py
│   └── Dockerfile
│
├── api-credenciamento/   ← Pessoa 3
│   ├── app.py
│   └── Dockerfile
│
├── docker-compose.yml    ← Pessoa 1
└── README.md             ← Pessoa 1
```

### Containers

| Container            | Tipo              | Banco/Tech   | Porta |
|----------------------|-------------------|--------------|-------|
| api-pessoas          | Aplicação Python  | PostgreSQL   | 8001  |
| api-eventos          | Aplicação Python  | MongoDB      | 8002  |
| api-credenciamento   | Aplicação Python  | Ambos (via rede) | 8003 |
| postgres-db          | Banco de Dados    | PostgreSQL   | 5432  |
| mongo-db             | Banco de Dados    | MongoDB      | 27017 |

### Fluxo de Comunicação (Rede Docker)

```
[api-pessoas]         → postgres-db
[api-eventos]         → mongo-db
[api-credenciamento]  → api-pessoas (HTTP)
[api-credenciamento]  → api-eventos (HTTP)
```

---

## Domínios do Sistema

### 1. API de Pessoas (api-pessoas) — PostgreSQL
Gerencia todos os atores do evento de grife:
- Celebridades
- Produtores
- Agentes
- Empresários
- Estilistas

**Endpoints CRUD:**
- `POST   /pessoas`        → Cadastrar nova pessoa
- `GET    /pessoas`        → Listar todas as pessoas
- `GET    /pessoas/{id}`   → Consultar pessoa por ID
- `PUT    /pessoas/{id}`   → Atualizar dados de pessoa
- `DELETE /pessoas/{id}`   → Remover pessoa

---

### 2. API de Eventos (api-eventos) — MongoDB
Gerencia os eventos de grife e seus detalhes:
- Nome, tema e edição do evento (ex: "Met Gala 2025 — Tempo e Moda")
- Data, local e dress code
- Lista de looks/coleções apresentadas

**Endpoints CRUD:**
- `POST   /eventos`        → Criar evento
- `GET    /eventos`        → Listar eventos
- `GET    /eventos/{id}`   → Consultar evento por ID
- `PUT    /eventos/{id}`   → Atualizar evento
- `DELETE /eventos/{id}`   → Remover evento

---

### 3. API de Credenciamento (api-credenciamento) — Integração
Gerencia convites, confirmações e acreditação dos participantes. Comunica-se com as duas APIs anteriores via rede Docker para validar pessoas e eventos antes de registrar um credenciamento.

**Endpoints CRUD:**
- `POST   /convites`            → Emitir convite (valida pessoa + evento)
- `GET    /convites`            → Listar todos os convites
- `GET    /convites/{id}`       → Consultar convite por ID
- `PUT    /convites/{id}/confirmar` → Confirmar presença
- `DELETE /convites/{id}`       → Cancelar convite

---

## Divisão de Funções por Integrante

---

### 👤 Pessoa 1 — Infraestrutura & DevOps Docker

**Responsabilidade:** Garantir que todo o ambiente Docker funcione corretamente, com todos os serviços conversando entre si.

**Entregas:**
- `docker-compose.yml` completo com todos os 5 serviços
- Configuração da rede Docker (`grife-network`)
- Volumes para persistência de dados (PostgreSQL e MongoDB)
- Variáveis de ambiente (`.env` ou inline no compose)
- `README.md` com instruções de execução do projeto

**Detalhes do docker-compose.yml:**
```yaml
# Serviços a configurar:
services:
  postgres-db     # imagem: postgres:15, volume, env vars
  mongo-db        # imagem: mongo:6, volume
  api-pessoas     # build: ./api-pessoas, depends_on: postgres-db
  api-eventos     # build: ./api-eventos, depends_on: mongo-db
  api-credenciamento  # build: ./api-credenciamento, depends_on: api-pessoas, api-eventos

networks:
  grife-network   # rede bridge interna

volumes:
  postgres-data
  mongo-data
```

**Conteúdo do README.md:**
- Descrição do projeto
- Diagrama da arquitetura
- Como executar: `docker compose up --build`
- Lista de endpoints de cada API

---

### 👤 Pessoa 2 — API de Pessoas + Banco PostgreSQL

**Responsabilidade:** Desenvolver o microsserviço de gestão de pessoas e configurar o banco relacional.

**Entregas:**
- `api-pessoas/app.py` com FastAPI ou Flask
- `api-pessoas/Dockerfile`
- Schema SQL de criação da tabela `pessoas`
- Endpoints CRUD completos com conexão ao PostgreSQL
- Tratamento de erros (404, 400, 500)

**Estrutura da tabela `pessoas` (PostgreSQL):**
```sql
CREATE TABLE pessoas (
  id        SERIAL PRIMARY KEY,
  nome      VARCHAR(100) NOT NULL,
  tipo      VARCHAR(50),   -- celebridade, produtor, agente, etc.
  agencia   VARCHAR(100),
  email     VARCHAR(100) UNIQUE,
  telefone  VARCHAR(20),
  criado_em TIMESTAMP DEFAULT NOW()
);
```

**Tecnologias:** Python + FastAPI (ou Flask) + psycopg2 ou SQLAlchemy

---

### 👤 Pessoa 3 — API de Eventos + API de Credenciamento + MongoDB

**Responsabilidade:** Desenvolver dois microsserviços — o de eventos (com MongoDB) e o de credenciamento (que integra os outros dois serviços via HTTP).

**Entregas:**
- `api-eventos/app.py` com FastAPI ou Flask
- `api-eventos/Dockerfile`
- `api-credenciamento/app.py` com FastAPI ou Flask
- `api-credenciamento/Dockerfile`
- Endpoints CRUD de eventos com conexão ao MongoDB
- Lógica de credenciamento que consulta `api-pessoas` e `api-eventos` via HTTP interno

**Estrutura do documento `evento` (MongoDB):**
```json
{
  "_id": "ObjectId",
  "nome": "Gala da Grife 2025",
  "tema": "Tempo e Alta Costura",
  "data": "2025-05-05",
  "local": "Metropolitan Museum, NY",
  "dress_code": "Black Tie / Haute Couture",
  "criado_em": "ISODate"
}
```

**Estrutura do documento `convite` (MongoDB ou PostgreSQL):**
```json
{
  "_id": "ObjectId",
  "pessoa_id": 1,
  "evento_id": "abc123",
  "status": "confirmado",
  "emitido_em": "ISODate"
}
```

**Tecnologias:** Python + FastAPI (ou Flask) + pymongo + requests (para chamadas HTTP entre containers)

---

## Resumo das Responsabilidades

| Entregável                     | Pessoa 1 | Pessoa 2 | Pessoa 3 |
|-------------------------------|----------|----------|----------|
| docker-compose.yml             | ✅       |          |          |
| Rede Docker (grife-network)    | ✅       |          |          |
| Volumes e persistência         | ✅       |          |          |
| README.md                      | ✅       |          |          |
| api-pessoas/app.py             |          | ✅       |          |
| api-pessoas/Dockerfile         |          | ✅       |          |
| Schema PostgreSQL              |          | ✅       |          |
| CRUD de Pessoas                |          | ✅       |          |
| api-eventos/app.py             |          |          | ✅       |
| api-eventos/Dockerfile         |          |          | ✅       |
| api-credenciamento/app.py      |          |          | ✅       |
| api-credenciamento/Dockerfile  |          |          | ✅       |
| CRUD de Eventos (MongoDB)      |          |          | ✅       |
| Integração entre APIs          |          |          | ✅       |

---

## Como Executar o Projeto

```bash
# Clonar o repositório
git clone <repo>
cd sistema-grife

# Subir todos os containers
docker compose up --build

# Testar as APIs
curl http://localhost:8001/pessoas
curl http://localhost:8002/eventos
curl http://localhost:8003/convites
```

---

## Checklist de Requisitos da Atividade

- [x] Mínimo de 5 containers
- [x] 2 bancos de dados (PostgreSQL + MongoDB)
- [x] 3 aplicações Python
- [x] APIs com endpoints de Cadastro, Consulta, Atualização e Remoção
- [x] Comunicação entre containers via rede Docker
- [x] docker-compose.yml
- [x] Dockerfiles para cada serviço
- [x] Persistência de dados (volumes)
- [x] Variáveis de ambiente
- [x] README.md com instruções
