### **Relatório de Análise Técnica: Falha na Integração do Módulo de Calendário**

**Para:** Equipe de Desenvolvimento, Amigo Virtual  
**De:** Arquiteto de Software Sênior  
**Data:** 11/06/2025  
**Assunto:** Análise de causa raiz e plano de ação corretivo para a funcionalidade de calendário.

---

### 1. Diagnóstico e Causa Raiz

A tentativa de integração da funcionalidade de calendário falhou devido a uma **incompatibilidade arquitetônica fundamental e uma implementação incompleta da infraestrutura de backend**. A análise do código existente revela uma transição em andamento de um protótipo puramente client-side (usando `localStorage`) para uma aplicação cliente-servidor robusta. A funcionalidade de calendário foi tentada nesse "meio do caminho", sem o suporte necessário para persistência de dados e lógica de servidor.

#### **Hipóteses Investigadas e Conclusões**

| Hipótese                                                                   | Análise                                                                                                                                                                                                                                                                                                 | Veredito                                                                                                   |
| -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| **1. Complexidade Excessiva para um Único Refino**                         | A tarefa de "adicionar um calendário" foi subestimada. Ela exige a criação de uma nova entidade de dados (compromissos), implementação completa de endpoints de API (CRUD), desenvolvimento de uma nova UI e, crucialmente, um mecanismo de servidor para lembretes proativos. Tentar fazer tudo isso em uma única etapa, sem a base pronta, estava fadado ao fracasso. | **Confirmada**                                                                                             |
| **2. Ausência de Endpoints de API Funcionais**                             | **Esta é a causa técnica primária.** O backend (`amigo-virtual-backend`) possui o modelo de dados `appointment_model.js` e as rotas em `appointment_routes.js`. No entanto, o controlador `appointment_controller.js` contém apenas stubs que retornam `501 Not Implemented`. O frontend não tinha uma API funcional contra a qual se desenvolver. | **Confirmada (Causa Direta)**                                                                              |
| **3. Inexistência de Componentes de UI e Lógica no Frontend**              | Como consequência direta da falha da API, nenhum dos códigos do frontend (`index.html`, `ui_controller.js`, etc.) contém elementos de interface (calendários, formulários, botões) ou lógica (`api_service.js`) para interagir com compromissos. O desenvolvimento do frontend foi bloqueado.                                           | **Confirmada (Sintoma)**                                                                                   |
| **4. Arquitetura Inadequada na Tentativa Inicial**                         | A primeira versão do aplicativo (baseada em `localStorage`) é estruturalmente incapaz de suportar um calendário com lembretes. Compromissos precisam ser armazenados em um banco de dados centralizado e um processo de servidor (backend) é indispensável para disparar lembretes de forma autônoma, independentemente de o usuário estar com o aplicativo aberto. | **Confirmada (Causa Fundamental)**                                                                         |

> **Conclusão Sumária:** A falha não foi um simples bug, mas um **colapso de planejamento estratégico**. A funcionalidade foi solicitada antes que a transição arquitetônica para um modelo cliente-servidor estivesse completa e funcional para a entidade de "Compromissos".

---

### 2. Plano de Ação Corretivo Detalhado

O plano a seguir é dividido em fases sequenciais para garantir uma implementação estável e testável. Ele utiliza a arquitetura mais recente (frontend refatorado + backend Node.js) como base.

#### **Fase 1: Implementação dos Endpoints do Backend (Node.js)**

O objetivo aqui é tornar a API de compromissos (`/api/appointments`) totalmente funcional.

1.  **Implementar as Funções do Controlador:** Modifique o arquivo `api/controllers/appointment_controller.js` para interagir com o banco de dados.

    ```javascript
    // api/controllers/appointment_controller.js

    const Appointment = require('../models/appointment_model.js');

    // GET /api/appointments - Obter todos os compromissos do usuário logado
    const getAppointments = async (req, res) => {
        try {
            const appointments = await Appointment.find({ user: req.user._id }).sort({ date: 1 });
            res.json(appointments);
        } catch (error) {
            res.status(500).json({ success: false, error: 'Server error while fetching appointments.' });
        }
    };

    // POST /api/appointments - Criar um novo compromisso
    const createAppointment = async (req, res) => {
        const { title, date, time } = req.body;

        if (!title || !date || !time) {
            return res.status(400).json({ success: false, error: 'Title, date, and time are required.' });
        }

        try {
            const appointment = new Appointment({
                user: req.user._id,
                title,
                date,
                time
            });

            const createdAppointment = await appointment.save();
            res.status(201).json(createdAppointment);
        } catch (error) {
            res.status(500).json({ success: false, error: 'Server error while creating appointment.' });
        }
    };

    // PUT /api/appointments/:id - Atualizar um compromisso
    const updateAppointment = async (req, res) => {
        try {
            const appointment = await Appointment.findById(req.params.id);

            if (!appointment) {
                return res.status(404).json({ success: false, error: 'Appointment not found.' });
            }

            // Garante que o usuário só pode editar seus próprios compromissos
            if (appointment.user.toString() !== req.user._id.toString()) {
                return res.status(401).json({ success: false, error: 'User not authorized.' });
            }

            appointment.title = req.body.title || appointment.title;
            appointment.date = req.body.date || appointment.date;
            appointment.time = req.body.time || appointment.time;

            const updatedAppointment = await appointment.save();
            res.json(updatedAppointment);
        } catch (error) {
            res.status(500).json({ success: false, error: 'Server error while updating appointment.' });
        }
    };

    // DELETE /api/appointments/:id - Deletar um compromisso
    const deleteAppointment = async (req, res) => {
        try {
            const appointment = await Appointment.findById(req.params.id);

            if (!appointment) {
                return res.status(404).json({ success: false, error: 'Appointment not found.' });
            }
            
            if (appointment.user.toString() !== req.user._id.toString()) {
                return res.status(401).json({ success: false, error: 'User not authorized.' });
            }

            await appointment.deleteOne(); // Use deleteOne() ou remove() dependendo da versão do Mongoose
            res.json({ success: true, message: 'Appointment removed.' });
        } catch (error) {
            res.status(500).json({ success: false, error: 'Server error while deleting appointment.' });
        }
    };

    module.exports = {
        getAppointments,
        createAppointment,
        updateAppointment,
        deleteAppointment,
    };
    ```

#### **Fase 2: Conexão da API no Frontend**

Agora, vamos criar as funções no frontend para chamar os novos endpoints.

1.  **Adicionar Funções ao `api_service.js`:** Este arquivo é o ponto central para todas as comunicações com o backend.

    ```javascript
    // Em api_service.js, dentro do objeto 'api'

    // ... (funções existentes de login, register, sendMessageToAI)

    getAppointments: async () => {
        const session = JSON.parse(localStorage.getItem('amigo_virtual_session'));
        const response = await fetch('/api/appointments', { // Assumindo proxy ou mesma origem
            headers: { 'Authorization': `Bearer ${session.token}` }
        });
        if (!response.ok) throw new Error('Failed to fetch appointments');
        return await response.json();
    },

    createAppointment: async (appointmentData) => {
        const session = JSON.parse(localStorage.getItem('amigo_virtual_session'));
        const response = await fetch('/api/appointments', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${session.token}`
            },
            body: JSON.stringify(appointmentData)
        });
        if (!response.ok) throw new Error('Failed to create appointment');
        return await response.json();
    },

    deleteAppointment: async (appointmentId) => {
        const session = JSON.parse(localStorage.getItem('amigo_virtual_session'));
        const response = await fetch(`/api/appointments/${appointmentId}`, {
            method: 'DELETE',
            headers: { 'Authorization': `Bearer ${session.token}` }
        });
        if (!response.ok) throw new Error('Failed to delete appointment');
        return await response.json();
    }
    ```

2.  **Atualizar o Gerenciador de Estado (`state.js`):** Incluir uma nova propriedade para armazenar os compromissos.

    ```javascript
    // Em state.js
    export let state = {
        // ... (outras propriedades)
        appointments: [], // Novo!
    };
    ```

#### **Fase 3: Construção da Interface do Calendário no Frontend**

Crie a tela e os componentes visuais para o usuário interagir.

1.  **Adicionar a Tela de Calendário ao `index.html`:** Insira este bloco de HTML dentro do `<div id="mobile-container">`.

    ```html
    <!-- ======================================================= -->
    <!-- CALENDAR SCREEN -->
    <!-- ======================================================= -->
    <div id="calendar-screen" class="screen hidden p-6 flex flex-col h-full bg-gray-50">
        <div class="flex items-center mb-6">
            <button id="back-to-chat-from-calendar" class="text-gray-600 hover:text-blue-500 mr-4">
                <i data-lucide="arrow-left"></i>
            </button>
            <h2 class="text-2xl font-bold text-gray-800">Meus Compromissos</h2>
        </div>

        <!-- Formulário para Adicionar Compromisso (inicialmente escondido) -->
        <form id="add-appointment-form" class="hidden bg-white p-4 rounded-lg shadow-md mb-6 space-y-3">
            <input type="text" id="appointment-title" placeholder="Título do Compromisso" class="form-input" required>
            <div class="flex gap-2">
                <input type="date" id="appointment-date" class="form-input w-1/2" required>
                <input type="time" id="appointment-time" class="form-input w-1/2" required>
            </div>
            <div class="flex gap-2">
                 <button type="submit" class="btn-primary flex-1">Salvar</button>
                 <button type="button" id="cancel-add-appointment" class="btn-secondary flex-1 bg-gray-200 text-gray-700">Cancelar</button>
            </div>
        </form>

        <!-- Lista de Compromissos -->
        <div id="appointments-list" class="flex-1 overflow-y-auto space-y-3">
            <!-- Compromissos serão injetados aqui -->
            <p class="text-center text-gray-500 mt-10">Você ainda não tem compromissos agendados.</p>
        </div>

        <!-- Botão Flutuante para Adicionar -->
        <div class="mt-auto pt-4">
            <button id="show-add-appointment-form" class="btn-primary w-full flex items-center justify-center gap-2">
                <i data-lucide="plus-circle"></i>
                <span>Novo Compromisso</span>
            </button>
        </div>
    </div>
    ```

2.  **Adicionar Lógica de Controle em `ui_controller.js`:**

    ```javascript
    // Em ui_controller.js

    // Na função initializeUI(), adicione:
    initCalendarScreen();

    // Adicione a nova função de inicialização
    function initCalendarScreen() {
        const form = document.getElementById('add-appointment-form');
        const showFormBtn = document.getElementById('show-add-appointment-form');
        const cancelBtn = document.getElementById('cancel-add-appointment');
        
        document.getElementById('back-to-chat-from-calendar').addEventListener('click', () => navigateTo('chat-screen'));

        showFormBtn.addEventListener('click', () => {
            form.classList.remove('hidden');
            showFormBtn.classList.add('hidden');
        });

        cancelBtn.addEventListener('click', () => {
            form.classList.add('hidden');
            form.reset();
            showFormBtn.classList.remove('hidden');
        });

        form.addEventListener('submit', async (e) => {
            e.preventDefault();
            const appointmentData = {
                title: document.getElementById('appointment-title').value,
                date: document.getElementById('appointment-date').value,
                time: document.getElementById('appointment-time').value,
            };

            try {
                const newAppointment = await api.createAppointment(appointmentData);
                updateState({ appointments: [...state.appointments, newAppointment] });
                renderAppointments();
                form.reset();
                form.classList.add('hidden');
                showFormBtn.classList.remove('hidden');
            } catch (error) {
                alert('Erro ao criar compromisso: ' + error.message);
            }
        });
        
        // Adicionar listener para deleção via event delegation
        document.getElementById('appointments-list').addEventListener('click', async (e) => {
            if (e.target.closest('.delete-appointment-btn')) {
                const button = e.target.closest('.delete-appointment-btn');
                const id = button.dataset.id;
                if (confirm('Tem certeza que deseja apagar este compromisso?')) {
                    try {
                        await api.deleteAppointment(id);
                        updateState({ appointments: state.appointments.filter(a => a._id !== id) });
                        renderAppointments();
                    } catch (error) {
                        alert('Erro ao apagar compromisso: ' + error.message);
                    }
                }
            }
        });
    }

    // Adicione uma função para renderizar a lista
    async function renderAppointments() {
        const listContainer = document.getElementById('appointments-list');
        listContainer.innerHTML = ''; // Limpa a lista

        if (state.appointments.length === 0) {
            listContainer.innerHTML = '<p class="text-center text-gray-500 mt-10">Você ainda não tem compromissos agendados.</p>';
            return;
        }

        state.appointments.forEach(app => {
            const appDate = new Date(app.date);
            const formattedDate = appDate.toLocaleDateString('pt-BR', { day: '2-digit', month: 'long', year: 'numeric', timeZone: 'UTC' });
            
            const element = document.createElement('div');
            element.className = 'bg-white p-4 rounded-lg shadow flex items-center justify-between';
            element.innerHTML = `
                <div>
                    <p class="font-bold text-gray-800">${app.title}</p>
                    <p class="text-sm text-gray-600">${formattedDate} às ${app.time}</p>
                </div>
                <button class="delete-appointment-btn text-red-500 hover:text-red-700" data-id="${app._id}">
                    <i data-lucide="trash-2" class="w-5 h-5"></i>
                </button>
            `;
            listContainer.appendChild(element);
        });
        lucide.createIcons(); // Recria os ícones
    }

    // Adicionar um botão de navegação na tela de chat
    // Em index.html, no header do chat-screen, adicione um botão:
    /*
    <button id="go-to-calendar" class="ml-auto mr-4 text-gray-500 hover:text-blue-500">
        <i data-lucide="calendar-days"></i>
    </button>
    */
    // E em ui_controller.js, na initChatScreen():
    /*
    document.getElementById('go-to-calendar').addEventListener('click', async () => {
        try {
            const appointments = await api.getAppointments();
            updateState({ appointments });
            renderAppointments();
            navigateTo('calendar-screen');
        } catch (error) {
            alert('Não foi possível carregar os compromissos.');
        }
    });
    */
    ```

#### **Fase 4: Implementação do Sistema Proativo de Lembretes (Backend)**

Esta é a etapa mais crítica para cumprir o requisito de "lembretes proativos". Usaremos um agendador de tarefas (`cron job`) no servidor.

1.  **Adicionar o Pacote `node-cron`:**
    No terminal do backend, execute: `npm install node-cron`

2.  **Criar o Serviço de Lembretes:** Crie um novo arquivo, por exemplo, `api/services/reminder_service.js`.

    ```javascript
    // api/services/reminder_service.js
    const cron = require('node-cron');
    const Appointment = require('../models/appointment_model');
    const Conversation = require('../models/conversation_model');

    // Função para injetar uma mensagem de lembrete no chat do usuário
    const sendReminderAsChatMessage = async (appointment) => {
        const reminderText = `Lembrete, meu amigo(a)! 😊 Você tem um compromisso agendado: "${appointment.title}" hoje às ${appointment.time}.`;
        
        try {
            await Conversation.create({
                user: appointment.user,
                messageText: reminderText,
                sender: 'ai',
            });

            // Marca o lembrete como enviado para não enviar de novo
            appointment.reminderSent = true;
            await appointment.save();
            console.log(`Reminder sent for appointment ${appointment._id}`);
        } catch (error) {
            console.error(`Failed to send reminder for appointment ${appointment._id}:`, error);
        }
    };

    // Tarefa agendada para rodar a cada 5 minutos
    const startReminderService = () => {
        console.log('Reminder service started. Checking for appointments every 5 minutes.');
        cron.schedule('*/5 * * * *', async () => {
            console.log('Running reminder check...');
            const now = new Date();
            const oneHourFromNow = new Date(now.getTime() + 60 * 60 * 1000);

            try {
                const upcomingAppointments = await Appointment.find({
                    date: {
                        $gte: now.setHours(0, 0, 0, 0), // A partir da meia-noite de hoje
                        $lt: new Date(now.getTime() + 24 * 60 * 60 * 1000).setHours(0, 0, 0, 0) // Até o fim do dia de hoje
                    },
                    reminderSent: false
                });

                for (const app of upcomingAppointments) {
                    // Lógica simples: envia se o compromisso for na próxima hora
                    const [hours, minutes] = app.time.split(':');
                    const appDateTime = new Date(app.date);
                    appDateTime.setHours(hours, minutes, 0, 0);

                    if (appDateTime > now && appDateTime < oneHourFromNow) {
                        await sendReminderAsChatMessage(app);
                    }
                }
            } catch (error) {
                console.error('Error in reminder cron job:', error);
            }
        });
    };

    module.exports = { startReminderService };
    ```

3.  **Iniciar o Serviço no `server.js`:**

    ```javascript
    // No topo do server.js
    const { startReminderService } = require('./api/services/reminder_service');

    // ... (depois de app.listen)
    app.listen(PORT, () => {
        console.log(`Server running on port ${PORT}`);
        createAdminAccount();
        startReminderService(); // Inicia o serviço de lembretes
    });
    ```

**Diagrama do Fluxo de Lembretes:**



Este plano de ação, se seguido metodicamente, resultará na implementação bem-sucedida e robusta da funcionalidade de calendário, corrigindo as falhas fundamentais da tentativa anterior.