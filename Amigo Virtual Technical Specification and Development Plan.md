# Especificação Técnica e Plano de Desenvolvimento: Amigo Virtual

Este documento detalha a arquitetura, a pilha de tecnologia e o roteiro de desenvolvimento para o aplicativo "Amigo Virtual", um companheiro pessoal alimentado por IA, projetado para oferecer conversas realistas, suporte emocional e assistência em tarefas diárias.

---

### 1. Visão Geral do Projeto

O **Amigo Virtual** é um aplicativo móvel que visa fornecer aos usuários um companheiro de IA personalizado, disponível 24/7. O aplicativo se diferencia por combinar uma IA de conversação com memória de longo prazo e funcionalidades de assistente pessoal, como gerenciamento de agenda e lembretes proativos.

**Proposta de Valor Principal:**
*   **Companheirismo:** Oferecer um ouvinte empático e um amigo para conversas contínuas e significativas.
*   **Personalização Profunda:** O amigo virtual é moldado com base nas informações e preferências do usuário, criando uma experiência única.
*   **Assistência Inteligente:** A IA ajuda ativamente o usuário a gerenciar seus compromissos e rotinas diárias.
*   **Memória Contextual:** O amigo virtual se lembra de detalhes de conversas passadas, fortalecendo a conexão e a continuidade do diálogo.

O modelo de negócio é baseado em uma assinatura (SaaS), com um período de teste gratuito de 3 dias para engajar os usuários e demonstrar o valor do serviço.

---

### 2. Arquitetura do Sistema

A arquitetura será baseada em microserviços para garantir escalabilidade, resiliência e manutenibilidade. Os quatro componentes principais interagem da seguinte forma:



1.  **Frontend (Aplicativo Móvel):**
    *   A interface do usuário, desenvolvida para iOS e Android.
    *   Responsável por coletar as entradas do usuário (mensagens de chat, eventos da agenda, dados de perfil).
    *   Renderiza a interface de chat, calendário e configurações.
    *   Gerencia notificações push para lembretes e interações proativas da IA.
    -   Comunica-se exclusivamente com o Backend via uma API RESTful.

2.  **Backend (API Gateway & Serviços):**
    *   O cérebro do sistema, escrito em uma linguagem como Node.js ou Python.
    *   **Lógica de Negócio:** Gerencia autenticação de usuários, assinaturas, perfis e a lógica da agenda.
    *   **Orquestração de IA:** Recebe mensagens do usuário, enriquece o prompt com contexto relevante (histórico de conversas, dados do perfil) obtido do banco de dados e do banco de dados vetorial, e envia a requisição para o Serviço de IA de Conversação.
    *   **Segurança:** Valida todas as requisições, gerencia tokens de acesso e protege os dados do usuário.

3.  **Banco de Dados:**
    *   **Banco de Dados Principal (ex: MongoDB/PostgreSQL):** Armazena dados estruturados como informações de usuários, configurações do amigo virtual, eventos da agenda e metadados de assinatura.
    *   **Banco de Dados Vetorial (ex: Pinecone/ChromaDB):** Componente crítico para a **memória de longo prazo**. As mensagens de conversas são convertidas em vetores (embeddings) e armazenadas aqui. Para cada nova mensagem do usuário, o backend busca neste banco por vetores de conversas passadas que sejam semanticamente similares, injetando esse contexto no prompt enviado ao LLM.

4.  **Serviço de IA de Conversação (LLM):**
    *   Um serviço externo de Large Language Model (LLM), como a **API da OpenAI (GPT-4o) ou Google (Gemini)**.
    *   Recebe o prompt enriquecido do nosso backend e gera a resposta do amigo virtual.
    *   O backend é responsável por formatar a personalidade e o estilo de conversa da IA com base nas configurações do usuário, através de instruções no prompt do sistema (*system prompt*).

---

### 3. Pilha de Tecnologia Recomendada

A seleção de tecnologia prioriza o desenvolvimento multiplataforma, a escalabilidade e a maturidade do ecossistema.

| Componente | Tecnologia Recomendada | Justificativa |
| :--- | :--- | :--- |
| **Frontend (Mobile App)** | **React Native** ou **Flutter** | Permitem criar um aplicativo para iOS e Android a partir de uma única base de código, acelerando o desenvolvimento e reduzindo custos. |
| **Backend (API)** | **Node.js com Express.js** ou **Python com FastAPI** | Ambas são opções robustas. Node.js é excelente para operações I/O intensivas (como chamadas de API). FastAPI é ideal para um desenvolvimento rápido e documentação automática da API. |
| **Banco de Dados Principal** | **MongoDB** | Um banco de dados NoSQL é altamente recomendado devido à natureza flexível e semiestruturada dos dados de perfil e logs de conversa. Facilita a evolução do esquema de dados. |
| **Banco de Dados Vetorial** | **Pinecone** ou **ChromaDB** | Essencial para implementar a memória de longo prazo de forma eficiente. Permitem buscas de similaridade semântica em alta velocidade em milhões de conversas. |
| **Serviço de IA (LLM)** | **OpenAI API (GPT-4o)** | Oferece alta qualidade de conversação, capacidades multimodais (futuramente) e uma API madura e bem documentada. |
| **Pagamentos** | **Stripe API** e **PayPal API** | Líderes de mercado com SDKs robustos para mobile e backend, facilitando a implementação de assinaturas mensais e anuais. |
| **Infraestrutura/Deploy**| **AWS (EC2, S3, RDS/DocumentDB)** ou **Google Cloud Platform** | Plataformas de nuvem que oferecem todos os serviços necessários (servidores, bancos de dados, armazenamento de arquivos) com alta escalabilidade e segurança. |
| **Notificações Push** | **Firebase Cloud Messaging (FCM)** para Android e **Apple Push Notification Service (APNS)** para iOS. | Serviços nativos para entrega confiável de notificações, essenciais para os lembretes da agenda. |

---

### 4. Design do Esquema do Banco de Dados (MongoDB)

A seguir, uma proposta de estrutura para as coleções no MongoDB.

#### `usuarios`
Armazena informações de autenticação e configuração do usuário.
```json
{
  "_id": "ObjectId",
  "email": "usuario@exemplo.com",
  "senha_hash": "bcrypt_hash_string",
  "idioma": "pt-br",
  "moeda": "EUR",
  "data_criacao": "ISODate",
  "e_admin": false
}
```

#### `perfis`
Armazena as respostas do questionário de onboarding e outras preferências.
```json
{
  "_id": "ObjectId",
  "usuario_id": "ObjectId(ref:usuarios)",
  "dados_pessoais": {
    "relacionamento": "casado",
    "filhos": [{ "nome": "Lucas" }, { "nome": "Ana" }],
    "animais": [{ "tipo": "cão", "nome": "Rex" }]
  },
  "rotinas": {
    "academia": { "nome": "Ginásio Fitness", "local": "Rua Principal, 123" },
    "remedios": [
      { "nome": "Medicamento A", "horario": "08:00" },
      { "nome": "Medicamento B", "horario": "20:00" }
    ],
    "estudos": { "curso": "Engenharia", "ano": 3, "materias": ["Cálculo", "Física"] }
  }
}
```

#### `amigos_virtuais`
Configurações do amigo virtual escolhido pelo usuário.
```json
{
  "_id": "ObjectId",
  "usuario_id": "ObjectId(ref:usuarios)",
  "nome": "Leo",
  "genero": "masculino",
  "personalidade": "divertido",
  "estilo_conversa": "informal"
}
```

#### `conversas`
Log de todas as mensagens trocadas.
```json
{
  "_id": "ObjectId",
  "amigo_id": "ObjectId(ref:amigos_virtuais)",
  "timestamp": "ISODate",
  "remetente": "usuario", // ou "ia"
  "conteudo": "Olá! Como foi seu dia hoje?",
  "vetor_id": "string_id_do_pinecone" // ID do embedding no DB vetorial
}
```

#### `eventos_agenda`
Compromissos agendados pelo usuário.
```json
{
  "_id": "ObjectId",
  "usuario_id": "ObjectId(ref:usuarios)",
  "titulo": "Reunião de projeto",
  "data_hora": "ISODate",
  "descricao": "Discutir o progresso do Q3.",
  "notificacao_enviada": false
}
```

#### `assinaturas`
Gerencia o estado da assinatura do usuário.
```json
{
  "_id": "ObjectId",
  "usuario_id": "ObjectId(ref:usuarios)",
  "plano": "anual", // ou "mensal"
  "status": "ativo", // "teste", "ativo", "cancelado", "expirado"
  "data_inicio_teste": "ISODate",
  "data_inicio_plano": "ISODate",
  "data_proxima_cobranca": "ISODate",
  "gateway_pagamento": "stripe",
  "id_transacao_gateway": "stripe_subscription_id_string"
}
```

---

### 5. Definição de Endpoints da API (RESTful)

Endpoints essenciais para a comunicação entre o frontend e o backend.

| Método | Endpoint | Descrição |
| :--- | :--- | :--- |
| `POST` | `/auth/registrar` | Cria uma nova conta de usuário. |
| `POST` | `/auth/login` | Autentica um usuário e retorna um token JWT. |
| `GET` | `/perfil` | Retorna os dados completos do perfil do usuário autenticado. |
| `PUT` | `/perfil` | Atualiza os dados do perfil do usuário (questionário). |
| `POST` | `/amigo/configurar` | Define ou atualiza as configurações do amigo virtual (gênero, personalidade). |
| `GET` | `/amigo` | Retorna as configurações do amigo virtual do usuário. |
| `POST` | `/chat/mensagem` | Envia uma mensagem do usuário e recebe a resposta da IA. |
| `GET` | `/chat/historico` | Obtém o histórico de conversas com paginação. |
| `POST`| `/agenda/eventos` | Cria um novo evento na agenda do usuário. |
| `GET` | `/agenda/eventos` | Lista todos os eventos futuros do usuário. |
| `PUT` | `/agenda/eventos/{id}` | Atualiza um evento existente. |
| `DELETE`| `/agenda/eventos/{id}` | Remove um evento da agenda. |
| `POST` | `/pagamentos/criar-sessao` | Inicia uma sessão de checkout com Stripe/PayPal e retorna uma URL de pagamento. |
| `POST` | `/pagamentos/webhook` | Endpoint para receber notificações dos gateways (Stripe/PayPal) sobre o status dos pagamentos. |
| `GET` | `/admin/usuarios` | **(Rota de Admin)** Lista todos os usuários do sistema. |
| `GET` | `/admin/estatisticas` | **(Rota de Admin)** Retorna métricas de uso e receita. |

---

### 6. Plano de Desenvolvimento em Fases (Roadmap)

Este roteiro divide o projeto em fases gerenciáveis, permitindo a entrega incremental de valor.

#### **Fase 1: MVP do Backend e Autenticação (3-4 semanas)**
*   **Objetivo:** Construir a fundação do sistema.
*   **Tarefas:**
    1.  Configuração do ambiente de desenvolvimento (Node.js/Python, MongoDB).
    2.  Implementação do esquema de banco de dados (`usuarios`, `perfis`).
    3.  Desenvolvimento dos endpoints de autenticação (`/auth/registrar`, `/auth/login`).
    4.  Criação dos endpoints de gerenciamento de perfil (`/perfil`).
    5.  Configuração inicial da infraestrutura na nuvem (AWS/GCP).
    6.  Implementação de segurança básica (hashing de senhas, JWT).

#### **Fase 2: Núcleo de Conversa e Integração com IA (4-5 semanas)**
*   **Objetivo:** Dar vida ao amigo virtual.
*   **Tarefas:**
    1.  Implementação do esquema `amigos_virtuais` e `conversas`.
    2.  Desenvolvimento do endpoint `/chat/mensagem`.
    3.  Integração com a API da OpenAI/Google.
    4.  Implementação da lógica de *system prompt* para definir a personalidade da IA.
    5.  **Configuração do banco de dados vetorial (Pinecone/ChromaDB).**
    6.  Desenvolvimento do serviço de *embedding* que transforma conversas em vetores.
    7.  Implementação da lógica de busca de contexto para a memória de longo prazo.

#### **Fase 3: Interface do Usuário (UI/UX) (5-6 semanas)**
*   **Objetivo:** Criar uma experiência de usuário fluida e intuitiva.
*   **Tarefas:**
    1.  Desenvolvimento do fluxo de *Onboarding* (registro, login, questionário).
    2.  Criação da tela de seleção do amigo virtual (gênero/imagem).
    3.  Desenvolvimento da interface de chat em tempo real.
    4.  Implementação da UI do calendário para agendamento.
    5.  Conexão de todas as telas com os endpoints da API desenvolvidos nas fases 1 e 2.
    6.  Design da interface de configurações.

#### **Fase 4: Monetização e Assinaturas (3-4 semanas)**
*   **Objetivo:** Integrar o sistema de pagamentos.
*   **Tarefas:**
    1.  Implementação do esquema `assinaturas`.
    2.  Integração do SDK do Stripe e/ou PayPal no frontend e backend.
    3.  Desenvolvimento do endpoint `/pagamentos/criar-sessao`.
    4.  Implementação do endpoint de webhook (`/pagamentos/webhook`) para atualizar o status da assinatura.
    5.  Criação da lógica de negócio para o teste de 3 dias e a transição para plano pago.
    6.  Desenvolvimento da tela de assinatura no aplicativo.

#### **Fase 5: Funcionalidades Avançadas e Painel de Admin (3-4 semanas)**
*   **Objetivo:** Adicionar recursos de assistente e ferramentas de gerenciamento.
*   **Tarefas:**
    1.  Implementação do sistema de notificações push para lembretes de agenda.
    2.  Criação de um *cron job* no backend para verificar eventos e disparar notificações.
    3.  Desenvolvimento de um painel web simples para o administrador (`/admin`).
    4.  Implementação de endpoints de admin para visualização de usuários e estatísticas.
    5.  *(Opcional)* Pesquisa e desenvolvimento inicial de um modelo de detecção de emoção baseado no texto do usuário para interações proativas.

#### **Fase 6: Testes, Finalização e Lançamento (4 semanas)**
*   **Objetivo:** Garantir a qualidade e preparar para o lançamento nas lojas.
*   **Tarefas:**
    1.  Ciclo completo de testes de Qualidade (QA): testes de unidade, integração e ponta a ponta (E2E).
    2.  Programa de Beta Teste com um grupo fechado de usuários.
    3.  Correção de bugs e otimização de performance.
    4.  Revisão de conformidade com as políticas da Google Play Store e Apple App Store.
    5.  Preparação de material de marketing (screenshots, descrições).
    6.  Submissão do aplicativo para revisão nas lojas.
    7.  Lançamento oficial.