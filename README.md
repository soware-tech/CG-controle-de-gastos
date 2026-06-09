<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CG - Controle de Gastos Inteligente (Regex Mode)</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    <!-- Biblioteca OCR Tesseract.js para leitura local de imagens sem uso de servidores ou APIs -->
    <script src="https://cdn.jsdelivr.net/npm/tesseract.js@5/dist/tesseract.min.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght=300;400;500;600;700;800;900&display=swap');
        body { font-family: 'Inter', sans-serif; }
        
        @keyframes scan-line {
            0% { top: 0%; }
            100% { top: 100%; }
        }
        .animate-scan-line {
            position: absolute;
            width: 100%;
            height: 2px;
            background: rgba(16, 185, 129, 0.5);
            box-shadow: 0 0 15px rgba(16, 185, 129, 0.5);
            animation: scan-line 2s linear infinite;
        }

        .tab-active {
            background: white;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
            color: #047857; /* emerald-700 */
        }
        
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
    </style>
</head>
<body class="bg-[#f8fafc] text-slate-900 pb-20">

    <!-- NAVBAR -->
    <nav class="bg-white border-b border-slate-200 sticky top-0 z-40">
        <div class="max-w-6xl mx-auto px-4 flex justify-between items-center h-16">
            <div class="flex items-center gap-2">
                <div class="bg-emerald-600 p-1.5 rounded text-white font-bold text-sm">CG</div>
                <span class="font-bold text-lg text-slate-700 hidden sm:block">Controle de Gastos</span>
            </div>
            <div class="flex bg-slate-100 p-1 rounded-xl font-medium" id="tab-container">
                <button onclick="switchTab('lancamentos')" id="btn-tab-lancamentos" class="px-4 py-1.5 rounded-lg text-xs font-black transition-all uppercase tab-active">Lançar</button>
                <button onclick="switchTab('dashboard')" id="btn-tab-dashboard" class="px-4 py-1.5 rounded-lg text-xs font-black transition-all uppercase text-slate-400">Mensal</button>
                <button onclick="switchTab('anual')" id="btn-tab-anual" class="px-4 py-1.5 rounded-lg text-xs font-black transition-all uppercase text-slate-400">Anual</button>
            </div>
            <button onclick="toggleConfig(true)" class="p-2 text-slate-400 hover:text-emerald-600 transition-colors">
                <i data-lucide="settings" class="w-5 h-5"></i>
            </button>
        </div>
    </nav>

    <!-- HEADER DINÂMICO -->
    <div class="bg-emerald-700 text-white py-4 shadow-inner">
        <div class="max-w-5xl mx-auto px-4 flex items-center justify-between">
            <button onclick="changePeriod(-1)" class="p-2 hover:bg-white/10 rounded-full transition-colors">
                <i data-lucide="chevron-left" class="w-5 h-5"></i>
            </button>
            <div class="text-center">
                <h2 id="display-title" class="font-black text-lg uppercase tracking-tight">Janeiro</h2>
                <p id="display-sub" class="text-[10px] opacity-70 font-black tracking-widest uppercase">2026</p>
            </div>
            <button onclick="changePeriod(1)" class="p-2 hover:bg-white/10 rounded-full transition-colors">
                <i data-lucide="chevron-right" class="w-5 h-5"></i>
            </button>
        </div>
    </div>

    <!-- MAIN CONTENT -->
    <main class="max-w-5xl mx-auto p-4 md:p-6">
        
        <!-- SEÇÃO LANÇAMENTOS -->
        <div id="section-lancamentos" class="space-y-6">
            
            <!-- FERRAMENTAS INTELIGENTES -->
            <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
                <div class="bg-white border border-emerald-100 rounded-2xl p-4 shadow-sm hover:shadow-md transition-shadow">
                    <div class="flex items-center gap-3 mb-3">
                        <div class="bg-emerald-100 p-2 rounded-xl text-emerald-600"><i data-lucide="camera" class="w-5 h-5"></i></div>
                        <h4 class="text-xs font-black text-slate-700 uppercase">Scanner</h4>
                    </div>
                    <button onclick="startScanner()" class="w-full bg-emerald-600 text-white font-bold py-2.5 rounded-xl text-xs uppercase flex items-center justify-center gap-2 hover:bg-emerald-700 shadow-sm active:scale-95 transition-all">
                        <i data-lucide="maximize" class="w-3.5 h-3.5"></i> Abrir Câmera
                    </button>
                </div>

                <div class="bg-white border border-blue-100 rounded-2xl p-4 shadow-sm hover:shadow-md transition-shadow">
                    <div class="flex items-center gap-3 mb-3">
                        <div class="bg-blue-100 p-2 rounded-xl text-blue-600"><i data-lucide="files" class="w-5 h-5"></i></div>
                        <h4 class="text-xs font-black text-slate-700 uppercase">Upload (até 5 imagens)</h4>
                    </div>
                    <label class="w-full bg-blue-50 border-2 border-dashed border-blue-200 text-blue-700 rounded-xl text-xs font-bold py-2.5 flex items-center justify-center gap-2 cursor-pointer hover:bg-blue-100 transition-all">
                        <i data-lucide="image" class="w-3.5 h-3.5"></i> Carregar Imagens
                        <input type="file" id="file-upload" class="hidden" accept="image/*" multiple onchange="handleFileUpload(event)" />
                    </label>
                </div>

                <div class="bg-white border border-slate-200 rounded-2xl p-4 shadow-sm hover:shadow-md transition-shadow">
                    <div class="flex items-center gap-3 mb-3">
                        <div class="bg-slate-100 p-2 rounded-xl text-slate-600"><i data-lucide="scan-qr-code" class="w-5 h-5"></i></div>
                        <h4 class="text-xs font-black text-slate-700 uppercase">Link de Nota</h4>
                    </div>
                    <div class="flex gap-2">
                        <input 
                            type="text" 
                            id="scan-input"
                            placeholder="Link NFC-e ou Chave de Acesso (44 dígitos)..." 
                            class="flex-1 bg-slate-50 border border-slate-100 rounded-lg text-xs p-2 focus:ring-1 focus:ring-slate-300 outline-none"
                        />
                        <button onclick="handleLinkProcess()" class="bg-slate-800 text-white px-3 py-2 rounded-lg font-bold text-xs hover:bg-slate-700 active:scale-95 transition-all">OK</button>
                    </div>
                </div>
            </div>

            <!-- VALIDAÇÃO DE PENDENTES EXTRAÍDOS COM REGEX -->
            <div id="pending-section" class="hidden bg-amber-50 border border-amber-200 rounded-2xl overflow-hidden shadow-md animate-in fade-in duration-300">
                <div class="p-4 bg-amber-100/50 flex justify-between items-center border-b border-amber-200">
                    <h3 class="text-xs font-black text-amber-800 uppercase flex items-center gap-2">
                        <i data-lucide="file-check" class="w-4 h-4 animate-pulse text-amber-600"></i> Validando nota (<span id="pending-count">0</span>)
                    </h3>
                    <button onclick="confirmAllPending()" class="bg-amber-600 text-white px-3 py-1 rounded-lg text-[10px] font-bold uppercase flex items-center gap-1 hover:bg-amber-700 transition-colors">
                        <i data-lucide="save" class="w-3 h-3"></i> Salvar Todos
                    </button>
                </div>
                <div id="pending-list" class="p-3 space-y-2"></div>
            </div>

            <!-- FORMULÁRIO MANUAL -->
            <form id="manual-form" class="bg-white rounded-2xl shadow-sm border border-slate-200 p-6 flex flex-col md:grid md:grid-cols-5 gap-4 items-end" onsubmit="handleManualSubmit(event)">
                <div class="w-full space-y-1">
                    <label class="text-[10px] font-bold text-slate-400 uppercase">Usuário</label>
                    <select id="select-user" name="user" class="w-full bg-slate-50 border border-slate-200 rounded-lg text-sm p-2 outline-none focus:ring-1 focus:ring-emerald-500 transition-all"></select>
                </div>
                <div class="w-full space-y-1">
                    <label class="text-[10px] font-bold text-slate-400 uppercase">Categoria</label>
                    <select id="select-category" name="category" class="w-full bg-slate-50 border border-slate-200 rounded-lg text-sm p-2 outline-none focus:ring-1 focus:ring-emerald-500 transition-all"></select>
                </div>
                <div class="w-full space-y-1">
                    <label class="text-[10px] font-bold text-slate-400 uppercase">Local/Estabelecimento</label>
                    <input name="description" required type="text" placeholder="Ex: Feira livre" class="w-full bg-slate-50 border border-slate-200 rounded-lg text-sm p-2 outline-none focus:ring-1 focus:ring-emerald-500 transition-all" />
                </div>
                <div class="w-full space-y-1">
                    <label class="text-[10px] font-bold text-slate-400 uppercase">Valor R$</label>
                    <input name="amount" required step="0.01" type="number" placeholder="0,00" class="w-full border border-emerald-200 bg-emerald-50/30 rounded-lg text-sm p-2 font-bold outline-none focus:ring-1 focus:ring-emerald-500 transition-all" />
                </div>

                <!-- CAMPO DE RECORRÊNCIA INTEGRADO -->
                <div class="w-full md:col-span-5 flex flex-col md:flex-row items-start md:items-center justify-between gap-3 p-3.5 bg-slate-50 border border-slate-200/60 rounded-xl mt-2">
                    <div class="flex items-center gap-2">
                        <input type="checkbox" id="check-recurring" class="rounded border-slate-300 text-emerald-600 focus:ring-emerald-500 h-4 w-4" onchange="toggleRecurringPeriod(this.checked)" />
                        <label for="check-recurring" class="text-xs font-bold text-slate-700 flex items-center gap-1.5 cursor-pointer">
                            <i data-lucide="repeat" class="w-4 h-4 text-emerald-600"></i> Tornar este gasto recorrente?
                        </label>
                    </div>

                    <div id="recurring-period-panel" class="hidden flex-wrap items-center gap-3 w-full md:w-auto mt-2 md:mt-0 p-2.5 bg-white border border-slate-100 rounded-lg animate-in slide-in-from-top-1 duration-150">
                        <div class="flex items-center gap-1.5">
                            <span class="text-[10px] font-bold text-slate-400 uppercase">Frequência:</span>
                            <select id="recurring-frequency" class="bg-slate-50 border border-slate-200 rounded px-2 py-1 text-xs outline-none focus:border-emerald-500">
                                <option value="mensal">Mensal</option>
                                <option value="bimestral">Bimestral</option>
                                <option value="trimestral">Trimestral</option>
                                <option value="semestral">Semestral</option>
                                <option value="anual">Anual</option>
                            </select>
                        </div>
                        <div class="flex items-center gap-1.5">
                            <span class="text-[10px] font-bold text-slate-400 uppercase">Repetir por:</span>
                            <input type="number" id="recurring-months" min="2" max="120" value="12" class="w-14 bg-slate-50 border border-slate-200 rounded px-2 py-1 text-xs text-center font-bold outline-none focus:border-emerald-500" />
                            <span class="text-[10px] font-bold text-slate-400 uppercase">vezes</span>
                        </div>
                    </div>
                </div>

                <button type="submit" class="w-full md:w-auto md:col-start-5 bg-emerald-600 text-white font-bold py-2.5 px-6 rounded-lg text-sm uppercase shadow-md hover:bg-emerald-700 active:scale-95 transition-all">Salvar</button>
            </form>

            <!-- TABELA GASTOS -->
            <div class="bg-white rounded-2xl shadow-sm border border-slate-200 overflow-hidden">
                <div class="px-6 py-4 border-b border-slate-100 flex justify-between items-center">
                    <h3 class="text-xs font-black text-slate-500 uppercase tracking-widest">Registros do Mês</h3>
                    <span id="registered-count" class="text-[10px] bg-slate-100 text-slate-600 px-2 py-1 rounded-full font-bold">0 itens</span>
                </div>
                <div class="overflow-x-auto">
                    <table class="w-full text-left">
                        <thead class="bg-slate-50">
                            <tr>
                                <th class="px-6 py-3 text-[10px] font-bold text-slate-400 uppercase">Data</th>
                                <th class="px-6 py-3 text-[10px] font-bold text-slate-400 uppercase">Usuário</th>
                                <th class="px-6 py-3 text-[10px] font-bold text-slate-400 uppercase">Estabelecimento / Local</th>
                                <th class="px-6 py-3 text-[10px] font-bold text-slate-400 uppercase">Categoria</th>
                                <th class="px-6 py-3 text-[10px] font-bold text-slate-400 uppercase text-right">Valor</th>
                                <th class="px-6 py-3 w-20"></th>
                            </tr>
                        </thead>
                        <tbody id="expense-table-body" class="divide-y divide-slate-100"></tbody>
                    </table>
                </div>
            </div>
        </div>

        <!-- SEÇÃO DASHBOARD MENSAL -->
        <div id="section-dashboard" class="hidden space-y-6">
            <div class="flex flex-col md:flex-row gap-4">
                <div id="user-stats-list" class="flex-1 grid grid-cols-2 gap-4"></div>
                <div class="md:w-80 bg-emerald-50 border border-emerald-100 p-4 rounded-2xl shadow-sm flex items-center justify-between">
                    <div>
                        <h3 class="text-[9px] font-black text-emerald-600 uppercase tracking-widest mb-0.5">Gasto Mensal</h3>
                        <div id="monthly-total-value" class="text-xl font-black text-emerald-900 tracking-tighter">R$ 0,00</div>
                        <div id="monthly-note-total" class="text-[10px] text-emerald-700/80 font-bold mt-1.5 flex items-center gap-1">
                            <i data-lucide="file-check-2" class="w-3.5 h-3.5"></i> Notas: R$ 0,00
                        </div>
                    </div>
                    <div class="bg-emerald-600 p-3 rounded-xl text-white shadow-lg shadow-emerald-200"><i data-lucide="wallet" class="w-5 h-5"></i></div>
                </div>
            </div>

            <!-- PROPORCIONAL POR CATEGORIA -->
            <div class="bg-white rounded-2xl border border-slate-200 p-6 shadow-sm">
                <h3 class="font-black text-slate-700 text-xs uppercase tracking-widest flex items-center gap-2 mb-8">
                    <i data-lucide="trending-up" class="text-emerald-500 w-4 h-4"></i> Categorias do Mês
                </h3>
                <div id="category-progress-list" class="grid grid-cols-1 md:grid-cols-2 gap-x-12 gap-y-6"></div>
            </div>
        </div>

        <!-- SEÇÃO ANUAL -->
        <div id="section-anual" class="hidden space-y-6">
            <div class="flex flex-col md:flex-row gap-4">
                <div id="user-annual-stats" class="flex-1 grid grid-cols-2 gap-4"></div>
                <div class="md:w-80 bg-slate-900 text-white p-6 rounded-2xl shadow-xl flex items-center justify-between">
                    <div>
                        <h3 id="annual-total-title" class="text-[10px] font-black text-emerald-400 uppercase tracking-[0.2em] mb-1">Total 2026</h3>
                        <div id="annual-total-value" class="text-3xl font-black tracking-tighter">R$ 0,00</div>
                        <div id="annual-note-total" class="text-[11px] text-emerald-300 font-bold mt-1.5 flex items-center gap-1.5">
                            <i data-lucide="file-check-2" class="w-4 h-4"></i> Notas: R$ 0,00
                        </div>
                    </div>
                    <div class="bg-white/10 p-4 rounded-2xl backdrop-blur-md"><i data-lucide="calendar-days" class="w-6 h-6"></i></div>
                </div>
            </div>

            <!-- GRÁFICO ANUAL MENSAL -->
            <div class="bg-white rounded-2xl border border-slate-200 p-6 shadow-sm">
                <h3 class="font-black text-slate-700 text-xs uppercase tracking-widest flex items-center gap-2 mb-10">
                    <i data-lucide="bar-chart-3" class="text-emerald-500 w-4 h-4"></i> Evolução Mensal
                </h3>
                <div id="annual-bars-container" class="flex items-end justify-between h-48 gap-2 px-2"></div>
            </div>
        </div>
    </main>

    <!-- OVERLAY DO SCANNER AO VIVO -->
    <div id="scanner-modal" class="fixed inset-0 z-50 bg-black hidden flex-col items-center justify-center">
        <div class="relative w-full max-w-lg aspect-[3/4] bg-slate-900 overflow-hidden">
            <video id="scanner-video" autoplay playsinline class="w-full h-full object-cover"></video>
            <div class="absolute inset-0 border-2 border-emerald-500/30 m-12 pointer-events-none">
                <div class="absolute top-0 left-0 w-full h-1 bg-emerald-500/50 animate-scan-line shadow-[0_0_15px_rgba(16,185,129,0.5)]"></div>
            </div>
            <div class="absolute bottom-10 left-0 right-0 flex justify-around items-center px-10">
                <button onclick="stopScanner()" class="p-4 bg-white/10 rounded-full text-white backdrop-blur-md hover:bg-white/20 transition-colors">
                    <i data-lucide="x" class="w-6 h-6"></i>
                </button>
                <button onclick="capturePhoto()" class="w-20 h-20 bg-white rounded-full border-4 border-emerald-500 flex items-center justify-center shadow-2xl active:scale-90 transition-all">
                    <div class="w-16 h-16 rounded-full border-2 border-slate-200 bg-white"></div>
                </button>
                <div class="w-14"></div>
            </div>
        </div>
        <canvas id="scanner-canvas" class="hidden"></canvas>
        <p class="text-white/60 text-[10px] font-black uppercase tracking-[0.3em] mt-8 animate-pulse">Aponte para o Cupom ou QR Code</p>
    </div>

    <!-- MODAL CONFIGURAÇÕES (PAINEL DE CONTROLE) -->
    <div id="config-modal" class="fixed inset-0 z-50 bg-slate-900/80 backdrop-blur-sm hidden items-center justify-center p-4">
        <div class="bg-white w-full max-w-2xl rounded-[2.5rem] shadow-2xl overflow-hidden animate-in zoom-in-95 duration-200">
            <div class="p-8 border-b border-slate-100 flex justify-between items-center bg-slate-50">
                <h3 class="font-black text-slate-800 uppercase tracking-widest text-sm flex items-center gap-2">
                    <i data-lucide="settings" class="text-emerald-600 w-5 h-5"></i> Painel de Controle
                </h3>
                <button onclick="toggleConfig(false)" class="text-slate-400 hover:text-red-500 transition-colors">
                    <i data-lucide="x" class="w-6 h-6"></i>
                </button>
            </div>
            
            <div class="p-8 space-y-10 max-h-[70vh] overflow-y-auto no-scrollbar">
                
                <!-- GERENCIAR USUÁRIOS -->
                <section class="space-y-6">
                    <div class="flex items-center justify-between border-b border-emerald-100 pb-2">
                        <h4 class="text-[10px] font-black text-emerald-600 uppercase tracking-[0.2em] flex items-center gap-2">
                            <i data-lucide="users" class="w-3.5 h-3.5"></i> Gerenciar Usuários
                        </h4>
                    </div>
                    <div class="flex flex-col gap-3">
                        <div class="flex gap-2">
                            <input 
                                type="text" 
                                id="new-user-input"
                                placeholder="Nome do Usuário..." 
                                class="flex-1 bg-slate-50 border border-slate-200 rounded-xl text-xs p-3 outline-none focus:ring-2 focus:ring-emerald-500/20"
                            />
                            <input 
                                type="text" 
                                id="new-user-card"
                                placeholder="Cartões (ex: 1234, 5678, 9012)" 
                                class="w-60 bg-slate-50 border border-slate-200 rounded-xl text-xs p-3 outline-none focus:ring-2 focus:ring-emerald-500/20"
                            />
                            <button 
                                onclick="handleCreateUser()"
                                class="bg-emerald-600 text-white px-5 rounded-xl font-bold text-xs uppercase shadow-md hover:bg-emerald-700 active:scale-95 transition-all"
                            >
                                <i data-lucide="user-plus" class="w-4 h-4"></i>
                            </button>
                        </div>
                    </div>
                    <div id="users-config-list" class="grid grid-cols-1 gap-4"></div>
                </section>

                <!-- GERENCIAR CATEGORIAS -->
                <section class="space-y-6">
                    <div class="flex items-center justify-between border-b border-emerald-100 pb-2">
                        <h4 class="text-[10px] font-black text-emerald-600 uppercase tracking-[0.2em] flex items-center gap-2">
                            <i data-lucide="tag" class="w-3.5 h-3.5"></i> Gerenciar Categorias
                        </h4>
                    </div>
                    <div class="flex gap-2">
                        <input 
                            type="text" 
                            id="new-category-input"
                            placeholder="Nova Categoria..." 
                            class="flex-1 bg-slate-50 border border-slate-200 rounded-xl text-xs p-3 outline-none focus:ring-2 focus:ring-emerald-500/20"
                        />
                        <button 
                            onclick="handleCreateCategory()"
                            class="bg-emerald-600 text-white px-5 rounded-xl font-bold text-xs uppercase shadow-md hover:bg-emerald-700 active:scale-95 transition-all"
                        >
                            ADD
                        </button>
                    </div>
                    <div id="categories-config-list" class="flex flex-wrap gap-2 p-4 bg-slate-50 rounded-[1.5rem] border border-slate-100"></div>
                </section>

            </div>
            
            <div class="p-8 bg-slate-50 border-t border-slate-100 text-center">
                <button onclick="toggleConfig(false)" class="bg-slate-800 text-white px-10 py-3 rounded-2xl font-bold text-xs uppercase shadow-lg hover:bg-slate-700 transition-all active:scale-95">Fechar</button>
            </div>
        </div>
    </div>

    <!-- FEEDBACK TOAST -->
    <div id="toast" class="fixed bottom-24 left-1/2 -translate-x-1/2 z-[60] hidden bg-slate-900 text-white text-[10px] font-black px-6 py-3 rounded-full items-center gap-3 uppercase border border-white/10 shadow-2xl animate-in slide-in-from-bottom-5">
        <div id="toast-loader" class="hidden"><i data-lucide="loader-2" class="animate-spin text-emerald-400 w-4 h-4"></i></div>
        <div id="toast-success" class="hidden"><i data-lucide="check-circle-2" class="text-emerald-400 w-4 h-4"></i></div>
        <span id="toast-message">Feedback</span>
    </div>

    <script>
        // --- ESTADOS DO SISTEMA ---
        let activeTab = 'lancamentos';
        let currentMonth = new Date().getMonth();
        let currentYear = new Date().getFullYear();
        let isScanning = false;
        let stream = null;

        // Dados Locais e Fallbacks
        let expenses = JSON.parse(localStorage.getItem('expenses_v12')) || [];
        let users = JSON.parse(localStorage.getItem('users_v12')) || [
            { id: '2', name: 'Usuário Padrão', cards: ['1234'] }
        ];
        let categories = JSON.parse(localStorage.getItem('categories_v12')) || [
            'Aluguel', 'Beleza', 'Casa', 'Combustível', 'Energia', 'Estacionamento', 
            'Manutenção Carro', 'Feira', 'Fast Food', 'Delivery', 'Gás', 
            'Mercado', 'Farmácia', 'Internet', 'Celular', 'Padaria', 'Pets'
        ];
        let pendingExpenses = [];

        const monthNames = ["Janeiro", "Fevereiro", "Março", "Abril", "Maio", "Junho", "Julho", "Agosto", "Setembro", "Outubro", "Novembro", "Dezembro"];

        // Mapeamento de meses abreviados para numérico (Português)
        const monthsMap = {
            jan: '01', fev: '02', mar: '03', abr: '04', mai: '05', may: '05',
            jun: '06', jul: '07', ago: '08', set: '09', out: '10', nov: '11', dez: '12'
        };

        // --- INICIALIZAÇÃO ---
        window.addEventListener('DOMContentLoaded', () => {
            loadLibraryJsQR();
            updateUI();
        });

        // Carregamento dinâmico da biblioteca jsQR
        function loadLibraryJsQR() {
            const script = document.createElement('script');
            script.src = "https://cdn.jsdelivr.net/npm/jsqr@1.4.0/dist/jsQR.min.js";
            script.async = true;
            document.body.appendChild(script);
        }

        // --- FUNÇÕES DE PERSISTÊNCIA ---
        function saveLocal() {
            localStorage.setItem('expenses_v12', JSON.stringify(expenses));
            localStorage.setItem('users_v12', JSON.stringify(users));
            localStorage.setItem('categories_v12', JSON.stringify(categories));
        }

        // --- GESTÃO DE MENUS E NAVEGAÇÃO ---
        function switchTab(tab) {
            activeTab = tab;
            
            // Alteração de classes dos botões das abas
            ['lancamentos', 'dashboard', 'anual'].forEach(t => {
                const btn = document.getElementById(`btn-tab-${t}`);
                if (t === tab) {
                    btn.className = "px-4 py-1.5 rounded-lg text-xs font-black transition-all uppercase tab-active";
                } else {
                    btn.className = "px-4 py-1.5 rounded-lg text-xs font-black transition-all uppercase text-slate-400 hover:text-slate-600";
                }
            });

            // Alteração de visibilidade das seções
            document.getElementById('section-lancamentos').classList.toggle('hidden', tab !== 'lancamentos');
            document.getElementById('section-dashboard').classList.toggle('hidden', tab !== 'dashboard');
            document.getElementById('section-anual').classList.toggle('hidden', tab !== 'anual');

            updateUI();
        }

        function changePeriod(delta) {
            if (activeTab === 'anual') {
                currentYear += delta;
            } else {
                currentMonth += delta;
                if (currentMonth < 0) {
                    currentMonth = 11;
                    currentYear--;
                } else if (currentMonth > 11) {
                    currentMonth = 0;
                    currentYear++;
                }
            }
            updateUI();
        }

        function updateUI() {
            // Atualizar cabeçalho dinâmico
            document.getElementById('display-title').innerText = activeTab === 'anual' ? 'Visão Anual' : monthNames[currentMonth];
            document.getElementById('display-sub').innerText = currentYear;

            // Renderizar formulários manuais
            const uSelect = document.getElementById('select-user');
            const cSelect = document.getElementById('select-category');
            uSelect.innerHTML = users.map(u => `<option value="${u.name}">${u.name}</option>`).join('');
            cSelect.innerHTML = categories.map(c => `<option value="${c}">${c}</option>`).join('');

            // Renderizar o conteúdo da aba selecionada
            if (activeTab === 'lancamentos') {
                renderExpenses();
                renderPending();
            } else if (activeTab === 'dashboard') {
                renderDashboard();
            } else if (activeTab === 'anual') {
                renderAnnual();
            }
            lucide.createIcons();
        }

        // --- SISTEMA TOAST ---
        let toastTimeout = null;
        function showToast(message, loading = false) {
            const toast = document.getElementById('toast');
            const msgSpan = document.getElementById('toast-message');
            const loader = document.getElementById('toast-loader');
            const success = document.getElementById('toast-success');

            clearTimeout(toastTimeout);
            toast.classList.remove('hidden');
            toast.classList.add('flex');
            msgSpan.innerText = message;

            if (loading) {
                loader.classList.remove('hidden');
                success.classList.add('hidden');
            } else {
                loader.classList.add('hidden');
                success.classList.remove('hidden');
                toastTimeout = setTimeout(hideToast, 3500);
            }
            lucide.createIcons();
        }

        function hideToast() {
            const toast = document.getElementById('toast');
            toast.classList.add('hidden');
            toast.classList.remove('flex');
        }

        // --- COMPRESSOR DE IMAGENS LOCAL ---
        function compressImage(file) {
            return new Promise((resolve) => {
                const reader = new FileReader();
                reader.onload = function(e) {
                    const img = new Image();
                    img.onload = function() {
                        const canvas = document.createElement('canvas');
                        const max_width = 400; // Resolução reduzida para armazenamento ultra leve
                        const scale = max_width / img.width;
                        canvas.width = max_width;
                        canvas.height = img.height * scale;
                        const ctx = canvas.getContext('2d');
                        ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
                        resolve(canvas.toDataURL('image/jpeg', 0.6)); // Qualidade comprimida a 60%
                    };
                    img.src = e.target.result;
                };
                reader.readAsDataURL(file);
            });
        }

        function regexMappingParser(rawText) {
            if (!rawText) return null;
            const textLower = rawText.toLowerCase();
            const lines = rawText.split('\n').map(l => l.trim()).filter(l => l.length > 0);

            let estabelecimento = "Estabelecimento Desconhecido";
            let valor = 0.00;
            let usuario = users[0]?.name || "Usuário";
            let categoria = "Mercado"; 
            let dataCompra = new Date().toISOString().split('T')[0]; // Padrão: Hoje
            let cartaoDetectado = "";

            // --- 1. REGEX DE CARTÃO (Detecta múltiplos formatos de mascaramento ou finais de cartão) ---
            const asteriskCardRegex = /(?:\*{2,}|X{2,})\s*(\d{4})\b/i;
            const asteriskMatches = rawText.match(asteriskCardRegex);

            if (asteriskMatches && asteriskMatches[1]) {
                cartaoDetectado = asteriskMatches[1];
            } else {
                const fallbackCardRegex = /(?:cartao|cartão|final|via|credito|debito|mastercard|visa|elo|hipercard)\D{0,12}(\d{4})\b/i;
                const fallbackMatches = rawText.match(fallbackCardRegex);
                if (fallbackMatches && fallbackMatches[1]) {
                    cartaoDetectado = fallbackMatches[1];
                }
            }

            // Atribuição automática do usuário com base no cartão mapeado (Suporta múltiplos cartões por usuário)
            if (cartaoDetectado) {
                for (const u of users) {
                    if (u.cards && u.cards.some(c => c.trim().includes(cartaoDetectado) || cartaoDetectado.includes(c.trim()))) {
                        usuario = u.name;
                        break;
                    }
                }
            }

            // --- 2. REGEX DE DATA (Padrões de datas numéricos ou com meses textuais abreviados em português) ---
            const ptMonthDateRegex = /\b(\d{1,2})[\/\.-](jan|fev|mar|abr|mai|may|jun|jul|ago|set|out|nov|dez)[^\d\s]*[\/\.-](\d{2,4})\b/i;
            const ptMonthMatch = rawText.match(ptMonthDateRegex);

            if (ptMonthMatch) {
                let d = parseInt(ptMonthMatch[1]);
                let mStr = ptMonthMatch[2].toLowerCase();
                let y = ptMonthMatch[3];
                let m = monthsMap[mStr];

                if (d >= 1 && d <= 31 && m) {
                    if (y.length === 2) {
                        y = "20" + y;
                    }
                    const yearInt = parseInt(y);
                    if (yearInt >= 2000 && yearInt <= 2040) {
                        dataCompra = `${y}-${m}-${String(d).padStart(2, '0')}`;
                    }
                }
            } else {
                const dateRegex = /\b(\d{2})[\/\.-](\d{2})[\/\.-](\d{2,4})\b/;
                const dateMatch = rawText.match(dateRegex);
                if (dateMatch) {
                    let d = parseInt(dateMatch[1]);
                    let m = parseInt(dateMatch[2]);
                    let y = dateMatch[3];

                    if (d >= 1 && d <= 31 && m >= 1 && m <= 12) {
                        if (y.length === 2) {
                            y = "20" + y;
                        }
                        const yearInt = parseInt(y);
                        if (yearInt >= 2000 && yearInt <= 2040) {
                            dataCompra = `${y}-${String(m).padStart(2, '0')}-${String(d).padStart(2, '0')}`;
                        }
                    }
                }
            }

            // --- 3. REGEX DO ESTABELECIMENTO / LOCAL (Filtra dados fiscais e prioriza o nome limpo e legível) ---
            const blacklistRegex = /(cnpj|ie|im|telefone|tel|rua|av\.|avenida|bairro|cep|fone|data|hora|cupom|fiscal|extrato|comprovante|venda|original|via|coo|ccf|val|pago|danfe|documento|auxiliar|nfc|site|sefaz|www\.|emiss|operador|caixa|terminal|autentic|filiacao|adquirente|transacao|credito|debito|tef|pos|mastercard|visa|elo|hipercard|label|aid|nsu|snpos|arqc)/i;
            const phoneCnpjRegex = /(\d{2}\.\d{3}\.\d{3}\/\d{4}-\d{2}|\d{3}\.\d{3}\.\d{3}-\d{2}|\d{2}\.\d{4,5}-\d{4})/g;

            let candidateName = "";

            for (const line of lines) {
                const cleanLine = line.trim();
                // Procurar linhas em maiúsculo estruturadas (geralmente nomes fantasia ou de titulares)
                if (cleanLine.length > 3 && 
                    /^[A-Z0-9\s\.\-\&\(\)\/]+$/.test(cleanLine) && 
                    cleanLine.split(/\s+/).length >= 2 &&
                    !blacklistRegex.test(cleanLine) && 
                    !phoneCnpjRegex.test(cleanLine) &&
                    !/\d{2}\/\d{2}\/\d{4}/.test(cleanLine)) {
                    
                    candidateName = cleanLine;
                    break;
                }
            }

            if (!candidateName) {
                for (let i = 0; i < Math.min(lines.length, 12); i++) {
                    const currentLine = lines[i].trim();
                    if (currentLine.length > 2 && 
                        !/^\d+$/.test(currentLine) && 
                        !blacklistRegex.test(currentLine) && 
                        !phoneCnpjRegex.test(currentLine) &&
                        !/\d{2}\/\d{2}\/\d{4}/.test(currentLine)) {
                        
                        candidateName = currentLine;
                        break;
                    }
                }
            }

            if (candidateName) {
                candidateName = candidateName.replace(/[^\w\s\-\.\,\&\(\)\/\']/gi, '').trim();
                estabelecimento = candidateName.split(' ').map(w => w.charAt(0).toUpperCase() + w.slice(1).toLowerCase()).join(' ');
            }

            // --- 4. REGEX DE VALOR (Garante prioridade absoluta e infalível pós "R$") ---
            const rsRegex = /R\$\s*([\d\.,]+)/i;
            const rsMatch = rawText.match(rsRegex);
            if (rsMatch && rsMatch[1]) {
                const val = parseNumber(rsMatch[1]);
                if (val > 0 && val < 99999) {
                    valor = val;
                }
            }

            // Busca secundária por totalizadores tradicionais caso "R$" falhe
            if (valor === 0.00) {
                const priceRegex = /(?:r\$|rs\$|\$)?\s*(\d{1,3}(?:\.\d{3})*,\d{2}|\d+[\.,]\d{2})/gi;
                const totalKeywords = /(total|pago|pagar|subtotal|import|valor|liquido|debito|credito|vlr tot|vlr)/i;
                let potentialValues = [];

                for (const line of lines) {
                    if (totalKeywords.test(line)) {
                        if (line.replace(/[^\d]/g, '').length > 20) continue; 
                        
                        const matchedPrices = line.match(priceRegex);
                        if (matchedPrices) {
                            matchedPrices.forEach(str => {
                                const val = parseNumber(str);
                                if (val > 0 && val < 9999) {
                                    potentialValues.unshift(val);
                                }
                            });
                        }
                    }
                }

                if (potentialValues.length === 0) {
                    const allPrices = rawText.match(priceRegex);
                    if (allPrices) {
                        allPrices.forEach(str => {
                            const val = parseNumber(str);
                            if (val > 0 && val < 5000) {
                                potentialValues.push(val);
                            }
                        });
                    }
                }

                if (potentialValues.length > 0) {
                    valor = Math.max(...potentialValues);
                }
            }

            // --- 5. CATEGORIZAÇÃO INTELIGENTE POR LEXICO ---
            const categoryLexicon = {
                'Aluguel': ['aluguel', 'locacao', 'condominio', 'imobiliaria', 'locatario'],
                'Beleza': ['salao', 'cabeleireiro', 'barba', 'barbearia', 'estetica', 'esmalte', 'manicure', 'cosmeticos', 'boticario', 'natura', 'maquiagem', 'perfumaria'],
                'Casa': ['tok st延长k', 'leroy', 'decoracao', 'moveis', 'construcao', 'utensilios', 'multicoisas', 'casa', 'enxoval', 'cama', 'mesa', 'banho', 'ferragem'],
                'Combustível': ['posto', 'combustivel', 'gasolina', 'etanol', 'diesel', 'petrobras', 'ipiranga', 'shell', 'ale', 'lubrificante', 'br', 'auto posto', 'convenie'],
                'Energia': ['copel', 'enel', 'cemig', 'elektro', 'coelba', 'luz', 'energia eletrica', 'celpe', 'neoenergia'],
                'Estacionamento': ['estacionamento', 'rotativo', 'zona azul', 'pare', 'park', 'parking', 'garagem', 'estacione'],
                'Manutenção Carro': ['oficina', 'mecanico', 'pecas', 'pneu', 'auto', 'autocenters', 'troca de oleo', 'freio', 'alinhamento', 'amortecedor'],
                'Feira': ['feira', 'fruta', 'verdura', 'legume', 'hortifruti', 'pomar', 'sacolao', 'hortalica'],
                'Fast Food': ['mcdonalds', 'mc donalds', 'burger king', 'bk', 'subway', 'habibs', 'bobs', 'giraffas', 'kfc', 'milkshake'],
                'Delivery': ['ifood', 'rappi', 'ubereats', 'delivery', 'entrega', 'pedido online'],
                'Gás': ['gas de cozinha', 'gas', 'liquigas', 'ultragaz', 'copagaz', 'glp', 'entregas de gas'],
                'Mercado': ['mercado', 'supermercado', 'hipermercado', 'atacado', 'carrefour', 'pao de acucar', 'extra', 'muffato', 'assai', 'atacadao', 'condor', 'compras', 'alimento', 'emporio', 'mercearia', 'super'],
                'Farmácia': ['drogaria', 'farmacia', 'remedio', 'medicamento', 'pague menos', 'raia', 'drogasil', 'sao joao', 'ultrapopular', 'panvel', 'pilula', 'comprimido'],
                'Internet': ['internet', 'fibra', 'net', 'claro residencial', 'copel telecom', 'oi fibra', 'brisanet'],
                'Celular': ['vivo', 'tim', 'claro', 'oi', 'recarga', 'celular', 'chip', 'pre-pago', 'movel'],
                'Padaria': ['padaria', 'panificadora', 'pao', 'panificacao', 'confeitaria', 'bolo', 'pao de queijo', 'broa', 'biscoito'],
                'Pets': ['pet', 'petshop', 'veterinaria', 'racao', 'cachorro', 'gato', 'banho e tosa', 'agropecuaria', 'animal', 'veterinario']
            };

            let bestCategory = categories[0] || 'Mercado';
            let maxScore = -1;

            for (const cat of categories) {
                const lexicon = categoryLexicon[cat] || [cat.toLowerCase()];
                let score = 0;
                lexicon.forEach(term => {
                    const pattern = new RegExp('\\b' + term + '\\b', 'gi');
                    const matches = textLower.match(pattern);
                    if (matches) {
                        score += matches.length * 2.0; 
                    }
                    if (textLower.includes(term)) {
                        score += 0.5; 
                    }
                });

                if (score > maxScore) {
                    maxScore = score;
                    bestCategory = cat;
                }
            }

            if (maxScore > 0) {
                categoria = bestCategory;
            }

            return {
                estabelecimento: estabelecimento,
                valor: parseFloat(valor.toFixed(2)),
                usuario: usuario,
                categoria: categoria,
                data: dataCompra,
                cartao: cartaoDetectado
            };
        }

        // Auxiliar: Normalização e limpeza de string numérica brasileira ou internacional
        function parseNumber(str) {
            let cleaned = str.replace(/[^\d,\.]/g, '');
            if (cleaned.includes(',') && cleaned.includes('.')) {
                cleaned = cleaned.replace(/\./g, '').replace(',', '.');
            } else if (cleaned.includes(',')) {
                cleaned = cleaned.replace(',', '.');
            }
            return parseFloat(cleaned) || 0;
        }

        function toggleRecurringPeriod(checked) {
            const panel = document.getElementById('recurring-period-panel');
            if (checked) {
                panel.classList.remove('hidden');
                panel.classList.add('flex');
            } else {
                panel.classList.remove('flex');
                panel.classList.add('hidden');
            }
        }

        function addPendingItem(data) {
            if (!data) return;
            const newItem = {
                id: Date.now() + Math.random(),
                user: data.usuario || users[0].name,
                description: data.estabelecimento || "Nova Compra",
                category: categories.includes(data.categoria) ? data.categoria : categories[0],
                amount: data.valor || 0,
                date: data.data || new Date().toISOString().split('T')[0],
                cartao: data.cartao || "",
                fileData: data.fileData || "",
                fileName: data.fileName || "comprovante.jpg",
                isNote: true
            };
            pendingExpenses.unshift(newItem);
            renderPending();
            showToast("Mapeamento Regex concluído com sucesso!");
        }

        function renderPending() {
            const container = document.getElementById('pending-section');
            const list = document.getElementById('pending-list');
            const count = document.getElementById('pending-count');

            if (pendingExpenses.length === 0) {
                container.classList.add('hidden');
                return;
            }

            container.classList.remove('hidden');
            count.innerText = pendingExpenses.length;

            list.innerHTML = pendingExpenses.map(p => `
                <div class="bg-white p-3.5 rounded-xl border border-amber-200 grid grid-cols-1 md:grid-cols-7 gap-3 items-center text-slate-700">
                    <div class="md:col-span-1">
                        <label class="block text-[8px] font-bold text-slate-400 uppercase tracking-wider mb-1">Usuário</label>
                        <select 
                            class="w-full text-[10px] font-bold border border-slate-200 p-1.5 rounded bg-slate-50 uppercase shadow-inner outline-none focus:border-amber-400"
                            onchange="updatePendingField('${p.id}', 'user', this.value)"
                        >
                            ${users.map(u => `<option value="${u.name}" ${p.user === u.name ? 'selected' : ''}>${u.name}</option>`).join('')}
                        </select>
                    </div>
                    <div class="md:col-span-2">
                        <label class="block text-[8px] font-bold text-slate-400 uppercase tracking-wider mb-1">Local / Estabelecimento</label>
                        <input 
                          type="text"
                          class="w-full text-xs border border-slate-200 rounded p-1.5 font-medium outline-none focus:border-amber-400"
                          value="${p.description}"
                          onchange="updatePendingField('${p.id}', 'description', this.value)"
                        />
                    </div>
                    <div class="md:col-span-1">
                        <label class="block text-[8px] font-bold text-slate-400 uppercase tracking-wider mb-1">Categoria</label>
                        <select 
                          class="w-full text-[10px] border border-slate-200 rounded p-1.5 bg-slate-50 outline-none focus:border-amber-400"
                          onchange="updatePendingField('${p.id}', 'category', this.value)"
                        >
                            ${categories.map(c => `<option value="${c}" ${p.category === c ? 'selected' : ''}>${c}</option>`).join('')}
                        </select>
                    </div>
                    <div class="md:col-span-1">
                        <label class="block text-[8px] font-bold text-slate-400 uppercase tracking-wider mb-1">Data da Compra</label>
                        <input 
                          type="date"
                          class="w-full text-[10px] font-semibold border border-slate-200 rounded p-1.5 bg-slate-50 outline-none focus:border-amber-400"
                          value="${p.date}"
                          onchange="updatePendingField('${p.id}', 'date', this.value)"
                        />
                    </div>
                    <div class="md:col-span-1">
                        <label class="block text-[8px] font-bold text-slate-400 uppercase tracking-wider mb-1">Valor R$</label>
                        <input 
                          type="number" 
                          step="0.01" 
                          class="w-full text-xs font-bold border border-slate-200 rounded p-1.5 bg-amber-50/30 text-amber-900 outline-none focus:border-amber-400"
                          value="${p.amount}"
                          onchange="updatePendingField('${p.id}', 'amount', this.value)"
                        />
                    </div>
                    <div class="md:col-span-1 flex flex-col gap-1.5">
                        <input 
                            type="text" 
                            maxlength="4"
                            placeholder="💳 Cartão"
                            class="text-[10px] text-center font-bold border border-emerald-100 bg-emerald-50 text-emerald-800 px-2 py-1 rounded-lg outline-none focus:ring-1 focus:ring-emerald-400"
                            value="${p.cartao || ''}"
                            onchange="updatePendingField('${p.id}', 'cartao', this.value)"
                        />
                        <div class="flex gap-2">
                            <button onclick="approvePending('${p.id}')" class="flex-1 bg-emerald-600 text-white p-2 rounded-lg text-xs font-bold uppercase hover:bg-emerald-700 transition-all flex justify-center shadow-sm"><i data-lucide="check" class="w-4 h-4"></i></button>
                            <button onclick="discardPending('${p.id}')" class="bg-red-50 text-red-500 p-2 rounded-lg text-xs hover:bg-red-100 transition-all flex justify-center"><i data-lucide="trash-2" class="w-4 h-4"></i></button>
                        </div>
                    </div>
                </div>
            `).join('');
            lucide.createIcons();
        }

        function updatePendingField(id, field, value) {
            const item = pendingExpenses.find(p => p.id == id);
            if (item) {
                if (field === 'amount') {
                    item[field] = parseFloat(value) || 0;
                } else {
                    item[field] = value;
                }
            }
        }

        function approvePending(id) {
            const index = pendingExpenses.findIndex(p => p.id == id);
            if (index !== -1) {
                const item = pendingExpenses[index];
                item.isNote = true;
                expenses.push(item);
                pendingExpenses.splice(index, 1);
                saveLocal();
                
                // Mapeia automaticamente a viewport do app para o respectivo mês da nota aprovada
                const parsedDate = new Date(item.date + 'T00:00:00');
                if (!isNaN(parsedDate.getTime())) {
                    currentMonth = parsedDate.getMonth();
                    currentYear = parsedDate.getFullYear();
                }
                
                updateUI();
                showToast("Lançamento confirmado!");
            }
        }

        function confirmAllPending() {
            if (pendingExpenses.length === 0) return;
            pendingExpenses.forEach(p => {
                p.isNote = true;
                expenses.push(p);
            });
            pendingExpenses = [];
            saveLocal();
            updateUI();
            showToast("Todos confirmados com sucesso!");
        }

        function discardPending(id) {
            pendingExpenses = pendingExpenses.filter(p => p.id != id);
            renderPending();
        }

        // --- SUBMISSÃO MANUAL ---
        function handleManualSubmit(e) {
            e.preventDefault();
            const formData = new FormData(e.target);
            const user = formData.get('user');
            const category = formData.get('category');
            const description = formData.get('description');
            const amount = parseFloat(formData.get('amount')) || 0;
            const isRecurring = document.getElementById('check-recurring').checked;

            if (isRecurring) {
                const recurrenceMonths = parseInt(document.getElementById('recurring-months').value) || 12;
                const frequency = document.getElementById('recurring-frequency').value || 'mensal';
                
                let intervalMonths = 1;
                if (frequency === 'bimestral') intervalMonths = 2;
                else if (frequency === 'trimestral') intervalMonths = 3;
                else if (frequency === 'semestral') intervalMonths = 6;
                else if (frequency === 'anual') intervalMonths = 12;

                for (let i = 0; i < recurrenceMonths; i++) {
                    const totalMonthOffset = i * intervalMonths;
                    let targetMonth = currentMonth + totalMonthOffset;
                    let targetYear = currentYear + Math.floor(targetMonth / 12);
                    targetMonth = targetMonth % 12;

                    const newExpense = {
                        id: Date.now() + Math.random(),
                        user: user,
                        category: category,
                        description: `${description} (${i + 1}/${recurrenceMonths})`,
                        amount: amount,
                        date: `${targetYear}-${String(targetMonth + 1).padStart(2, '0')}-15`,
                        isNote: false
                    };
                    expenses.push(newExpense);
                }
                showToast(`Recorrência criada para ${recurrenceMonths} períodos!`);
            } else {
                const newExpense = {
                    id: Date.now(),
                    user: user,
                    category: category,
                    description: description,
                    amount: amount,
                    date: `${currentYear}-${String(currentMonth + 1).padStart(2, '0')}-15`,
                    isNote: false
                };
                expenses.push(newExpense);
                showToast("Lançamento efetuado!");
            }

            saveLocal();
            updateUI();
            e.target.reset();
            
            document.getElementById('check-recurring').checked = false;
            toggleRecurringPeriod(false);
        }

        function deleteExpense(id) {
            expenses = expenses.filter(exp => exp.id != id);
            saveLocal();
            updateUI();
            showToast("Item removido");
        }

        function renderExpenses() {
            const body = document.getElementById('expense-table-body');
            const count = document.getElementById('registered-count');

            // Filtragem para o mês e ano do cabeçalho
            const currentPeriodExpenses = expenses.filter(exp => {
                const date = new Date(exp.date + 'T00:00:00');
                return date.getMonth() === currentMonth && date.getFullYear() === currentYear;
            });

            count.innerText = `${currentPeriodExpenses.length} itens`;

            if (currentPeriodExpenses.length === 0) {
                body.innerHTML = `
                    <tr>
                        <td colspan="6" class="text-center py-8 text-slate-400 text-xs">Nenhuma despesa para este período.</td>
                    </tr>
                `;
                return;
            }

            body.innerHTML = currentPeriodExpenses.map(exp => {
                const formattedDate = new Date(exp.date + 'T00:00:00').toLocaleDateString('pt-BR', {day: '2-digit', month: '2-digit'});
                return `
                    <tr class="hover:bg-slate-50 transition-colors">
                        <td class="px-6 py-4 text-xs font-bold text-slate-400">${formattedDate}</td>
                        <td class="px-6 py-4 text-xs font-semibold text-slate-700">
                            ${exp.user}
                            ${exp.cartao ? `<span class="block text-[8px] text-slate-400 font-extrabold mt-0.5">💳 ${exp.cartao}</span>` : ''}
                        </td>
                        <td class="px-6 py-4 text-xs font-medium text-slate-800">
                            <div class="flex items-center gap-1.5 flex-wrap">
                                ${exp.isNote ? `<span class="bg-emerald-50 text-emerald-700 border border-emerald-100 rounded px-1 py-0.2 text-[8px] font-black uppercase flex items-center gap-0.5"><i data-lucide="file-check-2" class="w-2.5 h-2.5"></i> NOTA</span>` : ''}
                                <span>${exp.description}</span>
                            </div>
                        </td>
                        <td class="px-6 py-4"><span class="px-2 py-0.5 bg-slate-100 text-slate-600 rounded-full text-[9px] font-bold uppercase">${exp.category}</span></td>
                        <td class="px-6 py-4 text-xs font-bold text-slate-900 text-right">R$ ${exp.amount.toLocaleString('pt-BR', {minimumFractionDigits: 2, maximumFractionDigits: 2})}</td>
                        <td class="px-6 py-4 text-right">
                            <div class="flex items-center justify-end gap-2.5">
                                ${exp.fileData ? `
                                    <a href="${exp.fileData}" download="${exp.fileName || 'comprovante.jpg'}" class="text-emerald-600 hover:text-emerald-800 p-1 transition-colors" title="Baixar comprovante">
                                        <i data-lucide="download" class="w-4 h-4"></i>
                                    </a>
                                ` : ''}
                                <button onclick="deleteExpense('${exp.id}')" class="text-slate-400 hover:text-red-500 transition-colors">
                                    <i data-lucide="trash-2" class="w-4 h-4"></i>
                                </button>
                            </div>
                        </td>
                    </tr>
                `;
            }).join('');
        }

        // Tesseract local OCR integrado para uploads de imagens
        async function handleFileUpload(event) {
            const files = event.target.files;
            if (files.length === 0) return;

            showToast("Processando a nota...", true);

            for (let i = 0; i < Math.min(files.length, 5); i++) {
                const file = files[i];
                try {
                    const fileData = await compressImage(file);
                    const text = await performLocalOCR(file);
                    const parsedData = regexMappingParser(text);
                    parsedData.fileData = fileData;
                    parsedData.fileName = file.name;
                    addPendingItem(parsedData);
                } catch (err) {
                    console.error(err);
                    showToast("Falha ao ler imagem");
                }
            }
            hideToast();
        }

        function performLocalOCR(fileOrUrl) {
            return new Promise((resolve, reject) => {
                Tesseract.recognize(
                    fileOrUrl,
                    'por', // Idioma Português brasileiro
                    { logger: m => console.log(m.status + ": " + Math.round(m.progress * 100) + "%") }
                ).then(({ data: { text } }) => {
                    resolve(text);
                }).catch(err => reject(err));
            });
        }

        async function handleLinkProcess() {
            const input = document.getElementById('scan-input');
            const val = input.value.trim();
            if (!val) return;

            showToast("Mapeando Link do Documento...", true);

            setTimeout(() => {
                let key = "";
                let value = 45.90;
                let dateStr = new Date().toISOString().split('T')[0];
                let invoiceNumber = "";
                let cnpj = "";

                // Verifica se há uma chave de acesso de 44 dígitos no texto do link
                const keyRegex = /\b\d{44}\b/;
                let keyMatch = val.match(keyRegex);

                if (!keyMatch && val.includes("p=")) {
                    try {
                        const urlObj = new URL(val);
                        const pParam = urlObj.searchParams.get("p");
                        if (pParam) {
                            const parts = pParam.split('|');
                            if (parts[0] && parts[0].length === 44) {
                                key = parts[0];
                                if (parts.length >= 5) {
                                    const possibleVal = parseFloat(parts[4]);
                                    if (!isNaN(possibleVal) && possibleVal > 0) {
                                        value = possibleVal;
                                    }
                                }
                                if (parts.length >= 4 && parts[3].length === 2) {
                                    const day = parts[3];
                                    const year = "20" + key.substring(2, 4);
                                    const month = key.substring(4, 6);
                                    dateStr = `${year}-${month}-${day}`;
                                }
                            }
                        }
                    } catch (e) {}
                }

                if (keyMatch) {
                    key = keyMatch[0];
                } else if (!key && val.length === 44 && /^\d+$/.test(val)) {
                    key = val;
                }

                if (val.includes("http") && !key) {
                    try {
                        const urlObj = new URL(val);
                        const valueParams = ["vTo", "vVal", "vTotal", "vNF", "valor", "total", "vProd"];
                        for (let param of valueParams) {
                            const pVal = urlObj.searchParams.get(param);
                            if (pVal) {
                                const parsed = parseFloat(pVal.replace(',', '.'));
                                if (!isNaN(parsed) && parsed > 0) {
                                    value = parsed;
                                    break;
                                }
                            }
                        }

                        const dhEmi = urlObj.searchParams.get("dhEmi");
                        if (dhEmi) {
                            let decodedDate = dhEmi;
                            const dateMatch = decodedDate.match(/^(\d{4})-(\d{2})-(\d{2})/);
                            if (dateMatch) {
                                dateStr = `${dateMatch[1]}-${dateMatch[2]}-${dateMatch[3]}`;
                            }
                        }

                        const keyParams = ["chNFe", "chave", "ch", "id"];
                        for (let param of keyParams) {
                            const pk = urlObj.searchParams.get(param);
                            if (pk && pk.length === 44 && /^\d+$/.test(pk)) {
                                key = pk;
                                break;
                            }
                        }
                    } catch(e) {}
                }

                // Desmontagem de Chave de Acesso de 44 dígitos
                if (key && key.length === 44) {
                    const yy = key.substring(2, 4);
                    const mm = key.substring(4, 6);
                    const year = "20" + yy;
                    const rawCnpj = key.substring(6, 20);
                    cnpj = rawCnpj.replace(/^(\d{2})(\d{3})(\d{3})(\d{4})(\d{2})$/, "$1.$2.$3/$4-$5");
                    const rawNumber = key.substring(25, 34);
                    invoiceNumber = parseInt(rawNumber, 10).toString();

                    if (dateStr) {
                        const dateParts = dateStr.split('-');
                        if (dateParts[0] !== year || dateParts[1] !== mm) {
                            const day = dateParts[2] || "15";
                            dateStr = `${year}-${mm}-${day}`;
                        }
                    } else {
                        dateStr = `${year}-${mm}-15`;
                    }
                }

                let estabelecimento = "NFC-e Sefaz";
                if (invoiceNumber && cnpj) {
                    estabelecimento = `NFC-e Nº ${invoiceNumber} (${cnpj})`;
                } else if (cnpj) {
                    estabelecimento = `NFC-e (${cnpj})`;
                } else if (key) {
                    estabelecimento = `NFC-e Chave: ...${key.substring(36)}`;
                }

                const receiptText = `NFC-e Chave: ${key || val}\nEstabelecimento: ${estabelecimento}\nValor: R$ ${value}\nData: ${dateStr}`;
                const fileData = "data:text/plain;charset=utf-8," + encodeURIComponent(receiptText);

                const parsedData = {
                    estabelecimento: estabelecimento,
                    valor: parseFloat(value.toFixed(2)),
                    usuario: users[0]?.name || "Usuário Padrão",
                    categoria: "Mercado",
                    date: dateStr,
                    cartao: "",
                    fileData: fileData,
                    fileName: `NFCe_${key ? key.substring(0,8) : 'nota'}.txt`
                };

                addPendingItem(parsedData);
                input.value = "";
                hideToast();
            }, 1000);
        }

        // --- GERENCIAMENTO DE SCANNER / CÂMERA AO VIVO ---
        async function startScanner() {
            const modal = document.getElementById('scanner-modal');
            const video = document.getElementById('scanner-video');

            modal.classList.remove('hidden');
            modal.classList.add('flex');
            isScanning = true;

            try {
                stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } });
                video.srcObject = stream;
                tickQRScan();
            } catch (err) {
                console.error("Sem acesso à câmera", err);
                showToast("Erro ao abrir câmera");
                stopScanner();
            }
        }

        function stopScanner() {
            const modal = document.getElementById('scanner-modal');
            modal.classList.add('hidden');
            modal.classList.remove('flex');
            isScanning = false;

            if (stream) {
                stream.getTracks().forEach(track => track.stop());
                stream = null;
            }
        }

        function tickQRScan() {
            if (!isScanning) return;
            const video = document.getElementById('scanner-video');

            if (video.readyState === video.HAVE_ENOUGH_DATA) {
                const canvas = document.getElementById('scanner-canvas');
                const ctx = canvas.getContext('2d');
                canvas.width = video.videoWidth;
                canvas.height = video.videoHeight;
                ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

                const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
                if (window.jsQR) {
                    const code = jsQR(imageData.data, imageData.width, imageData.height);
                    if (code) {
                        document.getElementById('scan-input').value = code.data;
                        stopScanner();
                        handleLinkProcess();
                        return;
                    }
                }
            }
            requestAnimationFrame(tickQRScan);
        }

        async function capturePhoto() {
            const canvas = document.getElementById('scanner-canvas');
            const video = document.getElementById('scanner-video');
            const ctx = canvas.getContext('2d');

            canvas.width = video.videoWidth;
            canvas.height = video.videoHeight;
            ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

            stopScanner();
            showToast("Processando imagem da câmera...", true);

            try {
                const fileData = canvas.toDataURL('image/jpeg', 0.6); // Compactação instantânea
                const text = await performLocalOCR(fileData);
                const parsedData = regexMappingParser(text);
                parsedData.fileData = fileData;
                parsedData.fileName = `foto_${Date.now()}.jpg`;
                addPendingItem(parsedData);
            } catch (err) {
                console.error(err);
                showToast("Erro ao realizar OCR na foto");
            }
            hideToast();
        }

        // --- DASHBOARDS E RELATÓRIOS ---
        function renderDashboard() {
            const userListContainer = document.getElementById('user-stats-list');
            const progressContainer = document.getElementById('category-progress-list');
            const totalDisplay = document.getElementById('monthly-total-value');
            const monthlyNoteTotal = document.getElementById('monthly-note-total');

            const currentPeriodExpenses = expenses.filter(exp => {
                const d = new Date(exp.date + 'T00:00:00');
                return d.getMonth() === currentMonth && d.getFullYear() === currentYear;
            });

            const totalSum = currentPeriodExpenses.reduce((acc, curr) => acc + curr.amount, 0);
            totalDisplay.innerText = `R$ ${totalSum.toLocaleString('pt-BR', {minimumFractionDigits: 2, maximumFractionDigits: 2})}`;

            // Totalização dos gastos via notas comprovadas
            const totalNoteAmount = currentPeriodExpenses.filter(e => e.isNote).reduce((acc, curr) => acc + curr.amount, 0);
            monthlyNoteTotal.innerHTML = `<i data-lucide="file-check-2" class="w-3.5 h-3.5"></i> Notas: R$ ${totalNoteAmount.toLocaleString('pt-BR', {minimumFractionDigits: 2, maximumFractionDigits: 2})}`;

            // Consolidação por Usuário
            const userTotals = {};
            users.forEach(u => userTotals[u.name] = 0);
            currentPeriodExpenses.forEach(exp => {
                if (userTotals[exp.user] !== undefined) {
                    userTotals[exp.user] += exp.amount;
                }
            });

            userListContainer.innerHTML = Object.entries(userTotals).map(([name, sum]) => `
                <div class="bg-white p-4 rounded-2xl border border-slate-200 shadow-sm flex flex-col justify-between">
                    <span class="text-[9px] font-bold text-slate-400 uppercase tracking-wider">${name}</span>
                    <span class="text-sm font-extrabold text-slate-800">R$ ${sum.toLocaleString('pt-BR', {minimumFractionDigits: 2, maximumFractionDigits: 2})}</span>
                </div>
            `).join('');

            // Consolidação por Categoria
            const catTotals = {};
            categories.forEach(cat => catTotals[cat] = 0);
            currentPeriodExpenses.forEach(exp => {
                if (catTotals[exp.category] !== undefined) {
                    catTotals[exp.category] += exp.amount;
                }
            });

            const sortedCategories = Object.entries(catTotals).sort((a,b) => b[1] - a[1]).filter(item => item[1] > 0);

            if (sortedCategories.length === 0) {
                progressContainer.innerHTML = `<p class="col-span-full text-center text-xs text-slate-400">Sem estatísticas para este período.</p>`;
                return;
            }

            progressContainer.innerHTML = sortedCategories.map(([catName, amount]) => {
                const percentage = totalSum > 0 ? (amount / totalSum) * 100 : 0;
                return `
                    <div class="space-y-1.5">
                        <div class="flex justify-between items-center text-xs">
                            <span class="font-extrabold text-slate-600 uppercase text-[10px]">${catName}</span>
                            <span class="font-bold text-slate-900">R$ ${amount.toLocaleString('pt-BR', {minimumFractionDigits: 2, maximumFractionDigits: 2})} <span class="text-[9px] text-slate-400 font-medium">(${percentage.toFixed(0)}%)</span></span>
                        </div>
                        <div class="w-full bg-slate-100 h-2 rounded-full overflow-hidden">
                            <div class="bg-emerald-600 h-full rounded-full transition-all duration-500" style="width: ${percentage}%"></div>
                        </div>
                    </div>
                `;
            }).join('');
        }

        function renderAnnual() {
            const userAnnualContainer = document.getElementById('user-annual-stats');
            const totalAnnualDisplay = document.getElementById('annual-total-value');
            const annualNoteTotal = document.getElementById('annual-note-total');
            const barsContainer = document.getElementById('annual-bars-container');

            document.getElementById('annual-total-title').innerText = `Total ${currentYear}`;

            const annualExpenses = expenses.filter(exp => {
                const d = new Date(exp.date + 'T00:00:00');
                return d.getFullYear() === currentYear;
            });

            const annualSum = annualExpenses.reduce((acc, curr) => acc + curr.amount, 0);
            totalAnnualDisplay.innerText = `R$ ${annualSum.toLocaleString('pt-BR', {minimumFractionDigits: 2, maximumFractionDigits: 2})}`;

            // Valor anual comprovado por Notas
            const annualNoteAmount = annualExpenses.filter(e => e.isNote).reduce((acc, curr) => acc + curr.amount, 0);
            annualNoteTotal.innerHTML = `<i data-lucide="file-check-2" class="w-4 h-4"></i> Notas: R$ ${annualNoteAmount.toLocaleString('pt-BR', {minimumFractionDigits: 2, maximumFractionDigits: 2})}`;

            // Consolidado de usuários anual
            const userTotals = {};
            users.forEach(u => userTotals[u.name] = 0);
            annualExpenses.forEach(exp => {
                if (userTotals[exp.user] !== undefined) {
                    userTotals[exp.user] += exp.amount;
                }
            });

            userAnnualContainer.innerHTML = Object.entries(userTotals).map(([name, sum]) => `
                <div class="bg-white p-4 rounded-2xl border border-slate-200 shadow-sm flex flex-col justify-between">
                    <span class="text-[9px] font-bold text-slate-400 uppercase tracking-wider">${name}</span>
                    <span class="text-sm font-extrabold text-slate-800">R$ ${sum.toLocaleString('pt-BR', {minimumFractionDigits: 2, maximumFractionDigits: 2})}</span>
                </div>
            `).join('');

            // Gráfico Anual por Mês
            const monthlyTotals = Array(12).fill(0);
            annualExpenses.forEach(exp => {
                const d = new Date(exp.date + 'T00:00:00');
                monthlyTotals[d.getMonth()] += exp.amount;
            });

            const maxMonthVal = Math.max(...monthlyTotals, 1);

            barsContainer.innerHTML = monthlyTotals.map((val, idx) => {
                const heightPercentage = (val / maxMonthVal) * 100;
                return `
                    <div class="flex-1 flex flex-col items-center group relative h-full justify-end">
                        <div class="absolute -top-10 scale-0 group-hover:scale-100 transition-transform bg-slate-800 text-white font-bold text-[9px] py-1 px-2 rounded whitespace-nowrap shadow-md z-30 pointer-events-none">
                            R$ ${val.toLocaleString('pt-BR', {minimumFractionDigits: 2, maximumFractionDigits: 2})}
                        </div>
                        <div class="w-full bg-emerald-100 rounded-t-lg relative overflow-hidden transition-all duration-500 hover:bg-emerald-600/30" style="height: ${heightPercentage}%">
                            <div class="absolute bottom-0 left-0 right-0 bg-emerald-600 transition-all" style="height: 100%"></div>
                        </div>
                        <span class="text-[8px] font-black text-slate-400 uppercase tracking-wider mt-2">${monthNames[idx].substring(0,3)}</span>
                    </div>
                `;
            }).join('');
        }

        // --- GERENCIAMENTO DE CONFIGURAÇÕES ---
        function toggleConfig(show) {
            const modal = document.getElementById('config-modal');
            modal.classList.toggle('hidden', !show);
            modal.classList.toggle('flex', show);
            if (show) {
                renderConfigLists();
            }
        }

        function renderConfigLists() {
            const userList = document.getElementById('users-config-list');
            const catList = document.getElementById('categories-config-list');

            userList.innerHTML = users.map(u => {
                const cardsHtml = u.cards && u.cards.length > 0
                    ? u.cards.map(c => `
                        <span class="inline-flex items-center gap-1 bg-emerald-50 text-emerald-800 border border-emerald-100 rounded px-1.5 py-0.5 text-[10px] font-bold">
                            💳 ${c}
                            <button onclick="removeCardFromUser('${u.id}', '${c}')" class="text-emerald-400 hover:text-red-500 transition-colors ml-0.5 font-bold" title="Remover cartão">×</button>
                        </span>
                      `).join('')
                    : `<span class="text-[10px] text-slate-400 italic">Sem cartões associados</span>`;

                return `
                    <div class="bg-slate-50 p-4 rounded-2xl border border-slate-100 animate-in fade-in duration-200 space-y-3.5">
                        <div class="flex justify-between items-start">
                            <div>
                                <p class="text-xs font-black text-slate-700 uppercase tracking-wide">${u.name}</p>
                                <div class="flex flex-wrap gap-1.5 mt-2">
                                    ${cardsHtml}
                                </div>
                            </div>
                            <button onclick="deleteUser('${u.id}')" class="text-slate-300 hover:text-red-500 transition-colors p-1" title="Excluir Usuário">
                                <i data-lucide="trash-2" class="w-4 h-4"></i>
                            </button>
                        </div>
                        <div class="flex gap-1.5 pt-2 border-t border-slate-200/60">
                            <input 
                                type="text" 
                                id="add-card-input-${u.id}"
                                placeholder="Adicionar cartão (4 dígitos)..." 
                                class="flex-1 bg-white border border-slate-200 rounded-lg text-[10px] px-2.5 py-1.5 outline-none focus:ring-1 focus:ring-emerald-500/30"
                                maxlength="4"
                            />
                            <button 
                                onclick="addCardToUser('${u.id}')"
                                class="bg-slate-800 text-white px-2.5 py-1.5 rounded-lg text-[9px] font-black uppercase hover:bg-slate-700 active:scale-95 transition-all flex items-center gap-1"
                            >
                                <i data-lucide="plus" class="w-3 h-3"></i> Add
                            </button>
                        </div>
                    </div>
                `;
            }).join('');

            catList.innerHTML = categories.map(c => `
                <span class="inline-flex items-center gap-1.5 bg-white border border-slate-100 shadow-sm pl-3 pr-2 py-1 rounded-full text-[10px] font-black text-slate-600 uppercase animate-in zoom-in-95 duration-150">
                    ${c}
                    <button onclick="deleteCategory('${c}')" class="text-slate-300 hover:text-red-500 transition-colors p-0.5"><i data-lucide="x" class="w-3.5 h-3.5"></i></button>
                </span>
            `).join('');
            lucide.createIcons();
        }

        function addCardToUser(userId) {
            const input = document.getElementById(`add-card-input-${userId}`);
            const cardVal = input.value.trim();

            if (!cardVal) return;
            if (!/^\d{4}$/.test(cardVal)) {
                showToast("Digite exatamente os 4 dígitos finais");
                return;
            }

            const user = users.find(u => u.id === userId);
            if (user) {
                if (!user.cards) user.cards = [];
                if (user.cards.includes(cardVal)) {
                    showToast("Este cartão já está cadastrado");
                    return;
                }
                user.cards.push(cardVal);
                saveLocal();
                renderConfigLists();
                updateUI();
                showToast("Cartão adicionado com sucesso!");
            }
        }

        function removeCardFromUser(userId, card) {
            const user = users.find(u => u.id === userId);
            if (user && user.cards) {
                user.cards = user.cards.filter(c => c !== card);
                saveLocal();
                renderConfigLists();
                updateUI();
                showToast("Cartão removido");
            }
        }

        function handleCreateUser() {
            const inputName = document.getElementById('new-user-input');
            const inputCard = document.getElementById('new-user-card');
            const name = inputName.value.trim();
            const cardStr = inputCard.value.trim();

            if (!name) return;

            const cardArray = cardStr ? cardStr.split(',').map(s => s.trim()).filter(s => /^\d{4}$/.test(s)) : [];

            users.push({
                id: Date.now().toString(),
                name: name,
                cards: cardArray
            });

            saveLocal();
            renderConfigLists();
            updateUI();

            inputName.value = "";
            inputCard.value = "";
            showToast("Usuário criado!");
        }

        function deleteUser(id) {
            if (users.length <= 1) {
                showToast("Mantenha ao menos um usuário ativo");
                return;
            }
            users = users.filter(u => u.id !== id);
            saveLocal();
            renderConfigLists();
            updateUI();
            showToast("Usuário deletado");
        }

        function handleCreateCategory() {
            const input = document.getElementById('new-category-input');
            const cat = input.value.trim();

            if (!cat || categories.includes(cat)) return;

            categories.push(cat);
            saveLocal();
            renderConfigLists();
            updateUI();

            input.value = "";
            showToast("Categoria cadastrada!");
        }

        function deleteCategory(cat) {
            categories = categories.filter(c => c !== cat);
            saveLocal();
            renderConfigLists();
            updateUI();
            showToast("Categoria removida");
        }
    </script>
</body>
</html>
