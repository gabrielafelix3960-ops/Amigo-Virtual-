# Guia de Implantação: Amigo Virtual

Este documento serve como um guia técnico completo para empacotar a aplicação web "Amigo Virtual" em um aplicativo móvel nativo e submetê-lo à Google Play Store e à Apple App Store. O guia assume o uso da tecnologia **Capacitor** devido à sua modernidade e integração profunda com o ecossistema de desenvolvimento nativo.

## 1. Análise Crítica e Reforço do Código-Fonte (Code Hardening)

Antes de iniciar o processo de empacotamento, é **imperativo** corrigir vulnerabilidades críticas de segurança e arquitetura presentes no código-fonte atual. A versão atual, embora funcional como protótipo, não é segura ou robusta para um ambiente de produção.

### **Questão Investigativa: O código atual está pronto para produção?**

**Conclusão:** Não. A análise revela três áreas de risco elevado: segurança de API, gestão de estado e autenticação. Ignorar estes pontos resultará em exposição de dados, uma experiência de usuário frágil e provável rejeição nas lojas de aplicativos.

---

### **Plano de Ação para Reforço**

#### **1.1. Segurança da API e Chaves de Acesso**

-   **Problema:** A chave da API do OpenRouter (`API_KEY`) está hardcoded no `api_service.js` do front-end. Qualquer pessoa que inspecione o código da aplicação pode roubar essa chave, gerando custos e abusos em seu nome.
-   **Solução (Não Negociável):**
    1.  **Remova a chave do front-end.** A variável `API_KEY` deve ser completamente eliminada do `api_service.js`.
    2.  **Centralize as chamadas no seu Backend.** O front-end **nunca** deve se comunicar diretamente com a API do OpenRouter. Em vez disso, deve chamar um endpoint no seu próprio backend (ex: `/api/chat`).
    3.  **Proxy de Backend:** Seu backend Node.js, que já está parcialmente implementado, deve receber a mensagem do usuário, adicionar a `API_KEY` (armazenada de forma segura como uma variável de ambiente no servidor) e, então, fazer a chamada para o OpenRouter.

**Exemplo de Refatoração:**

*Front-end (`api_service.js`):*
```javascript
// A chave e a URL direta foram removidas.
// O front-end agora chama o seu próprio backend.
const APP_API_URL = "https://your-backend-api.com/api/chat"; 

export async function getAiResponse(chatHistory, authToken) {
    // O token de autenticação do usuário é enviado para o seu backend
    const response = await fetch(APP_API_URL, {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${authToken}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            messages: chatHistory 
        })
    });
    // ...
}
```

*Backend (novo ou adaptado `chat_controller.js`):*
```javascript
const OPENROUTER_API_KEY = process.env.OPENROUTER_API_KEY; // Carregado de forma segura
const OPENROUTER_API_URL = "https://openrouter.ai/api/v1/chat/completions";

const postMessage = async (req, res) => {
    const userId = req.user._id; // Extraído do middleware de autenticação
    const { messages } = req.body;

    // ... (Lógica para obter o system prompt do usuário, etc.)
    const systemPrompt = getSystemPromptForUser(userId); 

    const response = await fetch(OPENROUTER_API_URL, {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${OPENROUTER_API_KEY}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            model: "deepseek/deepseek-chat-v3-0324:free",
            messages: [
                { role: "system", content: systemPrompt },
                ...messages
            ]
        })
    });
    // ... (Retornar a resposta para o front-end)
};
```

#### **1.2. Autenticação e Gestão de Sessão**

-   **Problema:** O token de autenticação (`userToken`) e os dados do usuário são armazenados em `localStorage`. Este é um local inseguro, vulnerável a ataques XSS e não persistente de forma garantida em dispositivos móveis.
-   **Solução:**
    1.  Utilize o plugin **Capacitor Secure Storage** para armazenar o JWT (JSON Web Token) de forma criptografada no dispositivo nativo (Keychain no iOS, EncryptedSharedPreferences no Android).
    2.  O fluxo deve ser: Login/Registro -> Receber JWT do backend -> Salvar JWT com Secure Storage -> Anexar JWT no cabeçalho `Authorization: Bearer <token>` em todas as chamadas subsequentes para *seu* backend.

#### **1.3. Persistência de Dados (Estado da Aplicação)**

-   **Problema:** A aplicação depende exclusivamente do `localStorage` para armazenar histórico de chat, configurações do amigo, questionário, etc. Estes dados podem ser perdidos se o SO limpar o cache do WebView.
-   **Solução:**
    1.  **Backend como Fonte da Verdade:** Utilize o backend com MongoDB, que já possui os modelos (`Conversation`, `UserLifeData`, etc.), como a fonte primária e autoritativa de todos os dados.
    2.  **Fluxo de Dados:** Ao iniciar o aplicativo, após o login, o front-end deve fazer chamadas à API para buscar o histórico de chat, as configurações do amigo e os dados do questionário.
    3.  **Cache Local (Opcional):** `localStorage` pode ser usado como um *cache* para melhorar a performance de carregamento, mas não como o armazenamento principal.

## 2. Preparação do Ambiente de Desenvolvimento Móvel

Antes de integrar o Capacitor, certifique-se de que seu ambiente está configurado.

1.  **Node.js:** Instale a versão LTS mais recente a partir do [site oficial](https://nodejs.org/).
2.  **Android Studio:** Instale o [Android Studio](https://developer.android.com/studio) para compilar e emular a versão Android. Siga as instruções de instalação para instalar o Android SDK.
3.  **Xcode:** Necessário para compilar e emular a versão iOS. Disponível apenas em macOS através da Mac App Store.
4.  **IDE:** Visual Studio Code é recomendado.

## 3. Integrando o Capacitor ao Projeto Web

Com o código-fonte reforçado, vamos transformá-lo em um projeto de aplicativo móvel.

1.  **Instale a CLI do Capacitor:**
    ```bash
    npm install @capacitor/core @capacitor/cli --save-dev
    ```

2.  **Inicialize o Capacitor no Projeto:**
    Execute o comando na raiz do seu projeto e siga as instruções.
    ```bash
    npx cap init "Amigo Virtual" "com.amigovirtual.app"
    ```
    -   *App Name:* Amigo Virtual
    -   *App ID:* `com.amigovirtual.app` (Este é um identificador único, como um domínio reverso. Use o seu.)

    Isso criará o arquivo `capacitor.config.ts`. Edite-o para apontar para o seu diretório de build (a pasta que contém `index.html`, `style.css`, etc.). Se seus arquivos estão na raiz, a configuração será:
    ```typescript
    import { CapacitorConfig } from '@capacitor/cli';

    const config: CapacitorConfig = {
      appId: 'com.amigovirtual.app',
      appName: 'Amigo Virtual',
      webDir: '.', // Se seus arquivos estão na raiz. Se estiverem em 'dist' ou 'build', mude aqui.
      server: {
        androidScheme: 'https'
      }
    };

    export default config;
    ```

3.  **Adicione as Plataformas Nativas:**
    ```bash
    # Para Android
    npx cap add android

    # Para iOS (apenas em macOS)
    npx cap add ios
    ```
    Isso criará as pastas `android/` e `ios/` na raiz do seu projeto, que contêm os projetos nativos.

## 4. Melhorando a Experiência com Plugins Nativos

Para que a aplicação se comporte como um aplicativo nativo e não como um site encapsulado, instale os seguintes plugins essenciais:

```bash
# Plugin para armazenamento seguro (para o JWT)
npm install @capacitor-community/secure-storage-plugin

# Plugins para controlar a barra de status, tela de splash e teclado
npm install @capacitor/status-bar @capacitor/splash-screen @capacitor/keyboard @capacitor/app

# Plugin para compras no aplicativo (obrigatório para as assinaturas)
npm install @capacitor-community/in-app-purchase
```

**Sincronize os plugins com os projetos nativos:**
```bash
npx cap sync
```

**Implementação:**
-   **Secure Storage:** Substitua todas as chamadas `localStorage.setItem('userToken', ...)` e `localStorage.getItem('userToken')` pelas funções do plugin de armazenamento seguro.
-   **Status Bar:** No seu `app.js`, configure a barra de status para se adequar ao design do seu app.
    ```javascript
    import { StatusBar, Style } from '@capacitor/status-bar';
    window.addEventListener('load', () => {
        StatusBar.setStyle({ style: Style.Light }); // Ou Dark
        StatusBar.setBackgroundColor({ color: '#ffffff' }); // Cor do seu cabeçalho
    });
    ```
-   **Splash Screen & Ícone:** Substitua os arquivos de ícone e splash screen padrão nas pastas `android/app/src/main/res` e `ios/App/App/Assets.xcassets`. Use uma ferramenta como o [Capacitor Assets](https://github.com/ionic-team/capacitor-assets) para automatizar a geração de todos os tamanhos necessários a partir de uma única imagem de 1024x1024.

## 5. Compilação para Produção (Build)

O processo consiste em sincronizar seus arquivos web e depois usar as ferramentas nativas (Android Studio e Xcode) para compilar os pacotes finais.

### **Android (.aab - Android App Bundle)**

1.  **Sincronize seu código web:**
    ```bash
    npx cap sync android
    ```
2.  **Abra o projeto no Android Studio:**
    ```bash
    npx cap open android
    ```
3.  **Gere uma Chave de Assinatura (Keystore):**
    -   No Android Studio, vá para **Build > Generate Signed Bundle / APK...**.
    -   Selecione **Android App Bundle** e clique em *Next*.
    -   Clique em **Create new...** para criar um novo Keystore.
    -   Preencha as informações e **guarde o arquivo `.jks` e as senhas em um local extremamente seguro.** *Perder esta chave significa que você não poderá mais atualizar seu aplicativo.*
4.  **Configure o `build.gradle`:**
    Adicione a configuração da sua chave de assinatura no arquivo `android/app/build.gradle` para automatizar a assinatura.
5.  **Compile o App Bundle:**
    -   Siga os passos em **Build > Generate Signed Bundle / APK...** novamente, mas desta vez selecione sua chave recém-criada.
    -   Escolha a variante de build `release`.
    -   Ao finalizar, o Android Studio gerará um arquivo `.aab` na pasta `android/app/release/`. **Este é o arquivo que você enviará para a Google Play.**

### **iOS (.ipa)**

1.  **Pré-requisito:** Você precisa de uma conta ativa no **Apple Developer Program** ($99/ano).
2.  **Sincronize seu código web:**
    ```bash
    npx cap sync ios
    ```
3.  **Abra o projeto no Xcode:**
    ```bash
    npx cap open ios
    ```
4.  **Configure a Assinatura e Capacidades (Signing & Capabilities):**
    -   No Xcode, selecione o projeto principal na barra de navegação à esquerda.
    -   Vá para a aba **Signing & Capabilities**.
    -   Marque "Automatically manage signing" e selecione sua equipe de desenvolvedor (Apple Developer Account).
    -   O `Bundle Identifier` deve corresponder ao que você configurou no Capacitor (`com.amigovirtual.app`).
    -   Na mesma aba, clique em `+ Capability` e adicione `In-App Purchase`.
5.  **Archive o Build:**
    -   No menu superior, selecione **Product > Archive**. (Certifique-se de que o dispositivo de destino selecionado seja "Any iOS Device (arm64)").
6.  **Distribua o App:**
    -   Após o arquivamento ser concluído, a janela "Organizer" abrirá.
    -   Com o seu arquivo selecionado, clique em **Distribute App**.
    -   Escolha **App Store Connect** como o método de distribuição e siga os passos. O Xcode irá compilar, assinar e fazer o upload do seu build (`.ipa`) diretamente para a App Store Connect.

## 6. Submissão às Lojas de Aplicativos

Esta é a fase mais detalhada e propensa a rejeições. **Atenção máxima é necessária.**

### **Ativos Comuns Necessários**
-   **Ícone do App:** 1024x1024px, formato PNG, sem transparência.
-   **Screenshots:** Imagens de alta qualidade mostrando as principais telas do app (Login, Chat, Questionário, Calendário, etc.). Prepare-as para diferentes tamanhos de tela (iPhone, iPad, celulares Android, tablets).
-   **URL da Política de Privacidade:** Um link público para uma página web com sua política de privacidade. **Isto é obrigatório.**
-   **Credenciais de Teste:** Um nome de usuário e senha de uma conta de teste para que os revisores possam acessar todas as funcionalidades do app, incluindo as pagas.

### **Google Play Store (via Play Console)**

1.  **Crie o App:** No [Play Console](https://play.google.com/console/), clique em "Criar app" e preencha os detalhes iniciais.
2.  **Listagem da Loja:** Preencha o nome, descrição curta, descrição completa, e faça o upload dos ícones e screenshots.
3.  **Upload do AAB:** Vá para a seção de "Produção" (ou "Teste Interno" para começar) e crie uma nova versão. Faça o upload do seu arquivo `.aab`.
4.  **Conteúdo do App (Seção Crítica):**
    -   **Acesso a apps:** Forneça as credenciais da conta de teste.
    -   **Anúncios:** Declare se o seu app contém anúncios (se não, marque "Não").
    -   **Classificação do conteúdo:** Responda ao questionário para gerar a classificação etária.
    -   **Público-alvo:** Defina as idades do seu público.
    -   **Segurança de dados (Data Safety):** Esta seção é fundamental. Você deve declarar **todos** os dados que coleta. Com base no seu código, a lista inclui:

| Tipo de Dado | Coletado | Compartilhado | Finalidade |
| :--- | :--- | :--- | :--- |
| **Informações pessoais** | Sim | Não | Funcionalidade do app, Gerenciamento de contas |
| *Nome* | Sim (indireto, se o usuário informar) | Não | Funcionalidade do app |
| *Endereço de e-mail* | Sim | Não | Funcionalidade do app, Gerenciamento de contas |
| *Identificadores de usuário (ID da conta)* | Sim | Não | Funcionalidade do app, Gerenciamento de contas |
| **Informações financeiras** | Sim | Sim (com o processador de pagamento) | Funcionalidade do app (para In-App Purchases) |
| *Histórico de compras* | Sim | Sim | Funcionalidade do app |
| **Saúde e fitness** | Sim | Não | Funcionalidade do app, Personalização |
| *Informações de saúde (medicação)* | Sim | Não | Funcionalidade do app |
| **Mensagens** | Sim | Não | Funcionalidade do app, Personalização |
| *E-mails* | Não | Não | - |
| *Outras mensagens no app (Chat)* | Sim | Não | Funcionalidade do app |
| **Atividade no app** | Sim | Não | Funcionalidade do app, Análise (se você usar) |
| *Interações no app (histórico de chat)* | Sim | Não | Funcionalidade do app |
| *Outro conteúdo gerado pelo usuário* | Sim | Não | Funcionalidade do app, Personalização |
| **Informações do app e desempenho** | Sim | Não | Análise, Diagnóstico |
| *Registros de falhas* | Sim | Não | Diagnóstico |

5.  **Revisão e Lançamento:** Após preencher tudo, clique em "Revisar versão" e envie para análise.

### **Apple App Store (via App Store Connect)**

1.  **Crie o App:** No [App Store Connect](https://appstoreconnect.apple.com/), vá para "Meus apps" e clique no ícone `+` para criar um novo app. Preencha os dados iniciais.
2.  **Informações do App:** Preencha a descrição, palavras-chave, URL de suporte e URL de marketing.
3.  **Privacidade do App (Rótulos de Privacidade):**
    -   Esta seção é similar à "Segurança de Dados" do Google. Você deve declarar os dados coletados. Use a tabela acima como guia. A Apple é extremamente rigorosa aqui. Seja honesto e completo. Por exemplo, para "Endereço de e-mail", você marcará:
        -   *Tipo de dado:* Informações de Contato -> Endereço de e-mail.
        -   *Finalidade:* Funcionalidade do App, Gerenciamento de Conta.
        -   *Vinculado ao usuário:* Sim.
        -   *Usado para rastreamento:* Não.
4.  **Compras dentro do app (In-App Purchases):**
    -   Você deve criar os produtos de assinatura (Mensal, Anual) na seção "Compras dentro do app". Os preços e durações devem corresponder ao que está no seu app.
5.  **Selecione o Build:** Na seção "Versão para iOS", selecione o build que você enviou pelo Xcode.
6.  **Informações para Revisão de App:**
    -   Forneça as **credenciais de teste** na seção "Anexos e informações de revisão de app".
    -   Em "Notas", explique claramente o propósito do aplicativo, especialmente a interação com a IA e o modelo de assinatura.
7.  **Enviar para Revisão:** Clique em "Adicionar para revisão" e depois em "Enviar para Revisão".

## 7. Considerações Estratégicas e Políticas de Conformidade

Para garantir 100% de aceitação, preste atenção aos seguintes pontos de risco:

### **Risco 1: Monetização com Pagamentos Externos**
> **Política:** Apple (Guideline 3.1.1) e Google (Payments policy) exigem que conteúdos ou funcionalidades digitais (como acesso premium a um chat) sejam vendidos exclusivamente através de seus sistemas de In-App Purchase.

-   **Ação:** O `paywall-screen.html` e a lógica de pagamento com Stripe/PayPal **devem ser completamente removidos**. Implemente a compra usando o plugin `@capacitor-community/in-app-purchase`. O fluxo será: usuário clica em "Assinar" -> seu app chama `purchaseProduct()` -> o sistema operacional (iOS/Android) mostra a tela de pagamento nativa -> seu app verifica a compra e libera o acesso.

### **Risco 2: Privacidade de Dados e IA**
> **Política:** Ambas as lojas exigem total transparência sobre a coleta e uso de dados. O uso de IA para interagir com dados pessoais sensíveis (saúde, filhos, etc.) aumenta o escrutínio.

-   **Ação:**
    1.  **Tenha uma Política de Privacidade impecável.** Contrate um serviço ou profissional para criá-la. Ela deve detalhar exatamente quais dados são coletados, como são usados (para personalizar a experiência do chat de IA), onde são armazenados e como o usuário pode solicitar a exclusão.
    2.  **Preencha os formulários de privacidade (Data Safety/Nutrition Labels) com 100% de precisão.** Discrepâncias entre o que você declara e o que o app faz levam à rejeição.

### **Risco 3: Persona da IA e Potencial Engano**
> **Política:** Apple (Guideline 1.1.6 - False Information) e Google (Deceptive Behavior policy) proíbem apps que enganam os usuários. Fazer a IA negar ser uma IA pode ser interpretado como engano.

-   **Ação (Mitigação de Risco):**
    1.  **Adicione um Disclaimer Transparente:** Durante o onboarding ou em uma tela "Sobre" facilmente acessível, inclua um texto claro: *"O Amigo Virtual é uma inteligência artificial avançada, projetada para ser um companheiro de conversação. Embora ele interaja com uma persona de amigo, por favor, esteja ciente de que você está se comunicando com um programa de IA."*
    2.  **Nota para o Revisor:** Na seção de notas para revisão do App Store Connect, explique proativamente essa dinâmica: "Nosso aplicativo usa uma IA que adota uma persona de 'amigo' para uma experiência mais imersiva. O usuário é informado na primeira utilização que está interagindo com uma IA, garantindo total transparência." Isso demonstra boa fé e reduz a chance de uma rejeição por engano.