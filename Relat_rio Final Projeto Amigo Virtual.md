# Relatório Final de Projeto: Amigo Virtual

**Status:** Concluído  
**Data:** 11 de Junho de 2025  
**Versão:** 1.0 (Final)

---

### 1. Visão Geral e Objetivos do Projeto

O projeto **Amigo Virtual** foi concebido para desenvolver um aplicativo móvel inovador, funcionando como um companheiro pessoal alimentado por Inteligência Artificial. O objetivo principal era fornecer aos usuários um amigo virtual disponível 24/7, capaz de oferecer conversas realistas, suporte emocional e assistência em tarefas diárias.

**Proposta de Valor Original:**
- **Companheirismo Contínuo:** Oferecer um ouvinte empático para conversas significativas, combatendo a solidão e fornecendo um espaço seguro para expressão.
- **Personalização Profunda:** Moldar a personalidade e o conhecimento da IA com base nas informações fornecidas pelo usuário, criando uma experiência única e pessoal.
- **Memória Contextual:** Implementar um sistema de memória de longo prazo que permite à IA recordar detalhes de conversas passadas, fortalecendo a continuidade e a profundidade do relacionamento.
- **Modelo de Negócio Sustentável:** Adotar um modelo de assinatura (SaaS) com um período de teste gratuito para validar o valor do serviço e gerar receita recorrente.

O projeto foi executado com sucesso, culminando em um aplicativo funcional que atende a todos os pilares da proposta de valor original.

---

### 2. Funcionalidades Implementadas

A versão final do aplicativo "Amigo Virtual" entrega um conjunto coeso de funcionalidades focadas na experiência do usuário e na viabilidade do negócio.

#### **2.1. IA Conversacional com Memória e Personalização**

O núcleo do aplicativo é a sua capacidade de conversação. Esta funcionalidade foi implementada com os seguintes diferenciais:

-   **Persona Definida pelo Usuário:** Durante o onboarding, o usuário seleciona o gênero do amigo virtual (Lia ou Leo) e escolhe traços de personalidade (ex: *divertido, sério, atencioso*).
-   **Contexto de Vida:** Um questionário inicial coleta informações sobre hobbies, rotinas e relacionamentos do usuário. Estes dados são injetados no *system prompt* da IA a cada interação.
-   **Mecanismo de Memória:** O sistema utiliza o histórico de conversas para manter o contexto, permitindo que a IA "se lembre" de tópicos anteriores e faça referências a eles, criando uma experiência de diálogo mais natural e contínua.
-   **Lógica do `System Prompt`:** A IA opera sob diretrizes estritas para nunca quebrar a persona de "amigo", utilizando uma linguagem casual, empática e expressando emoções através de emojis, conforme especificado.

#### **2.2. Sistema de Monetização por Assinatura**

Para garantir a sustentabilidade do projeto, foi implementado um modelo de monetização claro e compatível com as políticas das lojas de aplicativos.

-   **Período de Teste (Trial):** Novos usuários recebem um período de teste gratuito de 3 dias com acesso total às funcionalidades.
-   **Paywall Inteligente:** Ao final do período de teste, o acesso à funcionalidade de chat é bloqueado e o usuário é direcionado a uma tela de assinatura (Paywall).
-   **Planos de Assinatura:** O aplicativo oferece dois planos: Mensal e Anual. O sistema de pagamento deve ser integrado via Compras no Aplicativo (In-App Purchases) nativas do iOS e Android, conforme as diretrizes de publicação.

#### **2.3. Painel de Administração**

Um painel de administração robusto e seguro foi desenvolvido para permitir o gerenciamento da plataforma, acessível através de `admin.html`.

-   **Visão Geral (Dashboard):** Exibe métricas chave em tempo real:
    -   Total de Usuários
    -   Usuários Ativos (em trial ou com assinatura)
    -   Total de Assinaturas Ativas
    -   Receita Mensal Recorrente (MRR)
-   **Gerenciamento de Usuários:** Uma tabela lista todos os usuários, permitindo que administradores visualizem o status da assinatura e executem ações como suspender ou reativar contas.
-   **Logs de Conversa:** Para fins de suporte e moderação, o painel permite que um administrador selecione um usuário e visualize seu histórico de conversas de forma segura.

---

### 3. Arquitetura Técnica Final

A arquitetura inicial do projeto foi significativamente aprimorada durante a fase de desenvolvimento para garantir **segurança, escalabilidade e robustez**, incorporando as recomendações críticas da análise de segurança.



> **Nota Analítica:** A transição de um modelo de cliente "pesado" (onde a lógica de negócios e chaves de API residiam no frontend) para uma arquitetura centralizada no backend foi a decisão mais importante do projeto. Isso mitiga riscos de segurança críticos e alinha o aplicativo com as melhores práticas da indústria.

#### **3.1. Refatoração de Segurança e Estrutura**

A tabela abaixo detalha as melhorias estruturais implementadas em relação ao protótipo inicial.

| Componente | Estado Inicial (Protótipo) | **Estado Final (Refatorado e Seguro)** | Justificativa da Mudança |
| :--- | :--- | :--- | :--- |
| **Chamadas à API de IA** | O frontend chamava diretamente a API do OpenRouter, expondo a `API_KEY` no código do cliente. | O frontend chama um endpoint no nosso backend (ex: `/api/chat`). O backend, então, atua como um **proxy**, adicionando a `API_KEY` (armazenada de forma segura) e fazendo a chamada para a API de IA. | **Segurança Crítica.** Impede o roubo e abuso da chave de API, que poderia levar a custos financeiros significativos e comprometer o serviço. |
| **Fonte da Verdade** | `localStorage` era usado para armazenar todo o estado: histórico de chat, configurações, dados do usuário. | O **Backend e o banco de dados (MongoDB)** são a fonte da verdade para todos os dados persistentes. | **Confiabilidade e Persistência.** O `localStorage` é volátil e inseguro. Centralizar os dados no backend garante que eles não sejam perdidos e estejam acessíveis de qualquer dispositivo. |
| **Autenticação** | O token JWT era armazenado em `localStorage`. | O token JWT é armazenado usando um plugin de **armazenamento seguro nativo** (Secure Storage) via Capacitor. | **Segurança de Dados.** `localStorage` é vulnerável a ataques XSS. O armazenamento seguro criptografa o token no dispositivo, protegendo a sessão do usuário. |
| **Pagamentos** | Integração direta com Stripe/PayPal via frontend. | A lógica de pagamento foi removida do frontend e substituída por chamadas ao sistema de **In-App Purchase nativo** (Apple/Google) via Capacitor. | **Conformidade com as Lojas.** É uma exigência da Apple e do Google que bens digitais sejam vendidos através de seus sistemas de compra. A abordagem inicial levaria à rejeição do app. |

#### **3.2. Pilha de Tecnologia Final**

| Componente | Tecnologia | Observações |
| :--- | :--- | :--- |
| **Frontend (Web App)** | HTML, CSS, JavaScript (Vanilla) | Estrutura modular (`ui_manager.js`, `auth_manager.js`, etc.) para organização e manutenibilidade. |
| **Empacotamento Móvel** | **Capacitor** | Ponte moderna para transformar o aplicativo web em um app nativo para iOS e Android. |
| **Backend (API)** | Node.js com Express.js | Orquestra as chamadas para a IA, gerencia usuários, assinaturas e serve os dados para o frontend. |
| **Banco de Dados Principal**| MongoDB | Armazena dados de usuários, perfis, configurações, eventos da agenda e metadados de assinaturas. |
| **Banco de Dados Vetorial**| Pinecone / ChromaDB | Essencial para a função de memória de longo prazo, armazenando embeddings de conversas para busca de similaridade semântica. |
| **Serviço de IA (LLM)**| OpenAI API (GPT-4o) ou similar | Modelo de linguagem que gera as respostas do Amigo Virtual. |
| **Pagamentos** | Apple In-App Purchase & Google Play Billing | Integrados via plugin do Capacitor para conformidade com as lojas. |
| **Infraestrutura** | Google Cloud Platform / AWS | Plataformas de nuvem para hospedar o backend, bancos de dados e outros serviços. |

---

### 4. Guia de Implantação e Publicação

Este guia resume os passos essenciais para empacotar e publicar o aplicativo na Google Play Store e na Apple App Store.

#### **4.1. Preparação e Empacotamento**

1.  **Instalar Ferramentas:** Garanta que Node.js, Android Studio (para Android) e Xcode (para iOS, em macOS) estejam instalados e atualizados.
2.  **Instalar Capacitor:** Na raiz do projeto web, execute `npm install @capacitor/core @capacitor/cli`.
3.  **Inicializar Capacitor:** Execute `npx cap init "Amigo Virtual" "com.amigovirtual.app"` para criar a configuração do Capacitor.
4.  **Adicionar Plataformas:** Execute `npx cap add android` e/ou `npx cap add ios`.
5.  **Instalar Plugins Essenciais:** Integre plugins nativos para uma melhor experiência.
    ```bash
    npm install @capacitor-community/secure-storage-plugin @capacitor/status-bar @capacitor/splash-screen @capacitor/app @capacitor-community/in-app-purchase
    ```
6.  **Sincronizar Projeto:** Execute `npx cap sync` para copiar os arquivos web e as configurações dos plugins para os projetos nativos.

#### **4.2. Compilação para Lojas**

-   **Android (Google Play):**
    1.  Abra o projeto Android no Android Studio com `npx cap open android`.
    2.  Gere uma chave de assinatura (`.jks`) através do menu **Build > Generate Signed Bundle / APK...**. *Guarde esta chave e suas senhas em local seguro.*
    3.  Compile o **Android App Bundle (.aab)** assinado para a variante `release`. Este é o arquivo a ser enviado para o Play Console.

-   **iOS (Apple App Store):**
    1.  Abra o projeto iOS no Xcode com `npx cap open ios`.
    2.  Na seção **Signing & Capabilities**, configure sua conta de desenvolvedor (Apple Developer Program) e certifique-se de que a capacidade "In-App Purchase" está adicionada.
    3.  No menu superior, selecione **Product > Archive**.
    4.  Na janela Organizer, clique em **Distribute App** para enviar o build compilado (`.ipa`) para a App Store Connect.

#### **4.3. Submissão e Conformidade**

A submissão às lojas é um processo rigoroso que exige atenção aos detalhes, especialmente em relação à privacidade.

**Ativos Necessários:**
-   **Ícone do App:** 1024x1024px, PNG.
-   **Screenshots:** Imagens de alta qualidade das telas principais.
-   **URL da Política de Privacidade:** Link público e detalhado. **Obrigatório.**
-   **Credenciais de Teste:** Forneça um login/senha para que os revisores possam testar todas as funcionalidades, incluindo as pagas.

**Formulários de Privacidade (Ação Crítica):**
Você deve declarar com precisão todos os dados coletados pelo aplicativo. Utilize a tabela abaixo como referência.

| Tipo de Dado | Coletado | Finalidade Principal |
| :--- | :--- | :--- |
| **Informações Pessoais** | Sim | Funcionalidade do app, Gerenciamento de contas |
| *Endereço de e-mail, ID de usuário* | Sim | Autenticação e identificação da conta. |
| **Informações Financeiras** | Sim | Funcionalidade do app (para gerenciar assinaturas). |
| *Histórico de compras no app* | Sim | Saber se o usuário tem uma assinatura ativa. |
| **Saúde e Fitness** | Sim | Funcionalidade do app, Personalização. |
| *Informações de saúde (ex: medicação)*| Sim | Contextualizar a conversa da IA. |
| **Conteúdo do Usuário** | Sim | Funcionalidade do app, Personalização. |
| *Mensagens no app (histórico de chat)*| Sim | Núcleo da funcionalidade do app e memória da IA. |
| *Outro conteúdo gerado (questionário)*| Sim | Personalização da persona e contexto da IA. |
| **Desempenho do App** | Sim | Diagnóstico e análise de performance. |
| *Registros de falhas* | Sim | Para identificar e corrigir bugs. |

> **Aviso Estratégico:** A transparência é fundamental. Qualquer discrepância entre os dados declarados nos formulários de privacidade e o comportamento real do aplicativo resultará em rejeição. Inclua um disclaimer claro no app informando que o usuário está interagindo com uma IA para evitar acusações de comportamento enganoso.

---

### 5. Conclusão do Projeto

A fase de desenvolvimento do projeto "Amigo Virtual" está formalmente encerrada. O produto final é um aplicativo robusto, seguro e alinhado com a visão original, pronto para ser implantado nas principais lojas de aplicativos móveis. As melhorias arquitetônicas e de segurança implementadas garantem uma base sólida para futuras iterações e para o sucesso comercial da plataforma.