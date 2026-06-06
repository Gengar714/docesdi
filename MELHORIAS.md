# Guia de Melhorias - Docesdi

## 🔴 Bugs Críticos Encontrados

### 1. **Arquivo `produtos.json` Inválido**
**Problema:** Contém código de `package.json` ao invés de dados de produtos.

**Solução:**
```json
{
  "produtos": [
    {
      "id": 1,
      "nome": "Bolo de Chocolate",
      "descricao": "Bolo fofinho com cobertura de chocolate",
      "preco": 45.90,
      "estoque": 10,
      "categoria_id": 1,
      "imagem": "https://images.pexels.com/photos/291528/pexels-photo-291528.jpeg"
    }
  ],
  "categorias": [
    { "id": 1, "nome": "Bolos", "icone": "🍰" }
  ]
}
```

---

### 2. **Modal não abre (CSS bug)**
**Problema:** `.modal` está `display: none` com `justify-content`/`align-items`, que não funcionam sem `display: flex`.

**Solução - CSS:**
```css
.modal {
    display: none;
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0,0,0,0.5);
    justify-content: center;
    align-items: center;
}

.modal.ativo {
    display: flex; /* Adicionar classe .ativo quando abrir */
}
```

**Solução - JavaScript:**
```javascript
function abrirModal(modalId) {
    document.getElementById(modalId).classList.add('ativo');
}

function fecharModal() {
    document.querySelectorAll('.modal.ativo').forEach(m => {
        m.classList.remove('ativo');
    });
}
```

---

### 3. **Event Listeners Faltando**
**Problema:** Botões sem listeners causam erros quando clicados.

**Solução - Adicionar ao script:**
```javascript
// Busca e Filtros
document.getElementById('btnBuscar').addEventListener('click', buscarProdutos);
document.getElementById('btnOpenFilter').addEventListener('click', () => {
    abrirModal('modalFiltros');
});
document.getElementById('btnCloseFilter').addEventListener('click', fecharModal);
document.getElementById('btnApplyFilters').addEventListener('click', aplicarFiltros);
document.getElementById('btnResetFilters').addEventListener('click', resetarFiltros);

// Admin Tabs
document.querySelectorAll('.admin-tab').forEach(tab => {
    tab.addEventListener('click', function() {
        mudarAbaAdmin(this.dataset.tab);
    });
});

// Estoque Tabs
document.querySelectorAll('.estoque-tab').forEach(tab => {
    tab.addEventListener('click', function() {
        mudarAbaEstoque(this.dataset.estoquePanel);
    });
});

// Tema
document.getElementById('themeToggle').addEventListener('click', () => {
    document.body.classList.toggle('dark');
    localStorage.setItem('tema', document.body.classList.contains('dark') ? 'dark' : 'light');
});

// Login
document.getElementById('btnLogin').addEventListener('click', loginUsuario);
document.getElementById('btnCadastrar').addEventListener('click', cadastrarUsuario);
```

---

### 4. **Swiper Não Inicializa**
**Problema:** `swiperInstance` fica `null`, carrossel não funciona.

**Solução - Adicionar função:**
```javascript
function inicializarSwiper() {
    if (!swiperInstance) {
        swiperInstance = new Swiper('#productSwiper', {
            slidesPerView: 1,
            spaceBetween: 20,
            pagination: {
                el: '.swiper-pagination',
                clickable: true,
            },
            navigation: {
                nextEl: '.swiper-button-next',
                prevEl: '.swiper-button-prev',
            },
            breakpoints: {
                640: { slidesPerView: 2 },
                768: { slidesPerView: 3 },
                1024: { slidesPerView: 4 }
            }
        });
    }
}
```

---

### 5. **Falta Tratamento de Erros - Supabase**
**Problema:** Se Supabase falhar, app quebra sem fallback.

**Solução:**
```javascript
async function carregarDados() {
    try {
        // Tentar carregar do Supabase
        const { data: dataProdutos } = await supabaseClient
            .from('produtos')
            .select('*');
        
        const { data: dataCategorias } = await supabaseClient
            .from('categorias')
            .select('*');
        
        if (dataProdutos) produtos = dataProdutos;
        if (dataCategorias) categorias = dataCategorias;
        
    } catch (erro) {
        console.error('Erro ao carregar Supabase:', erro);
        // Usar dados padrão como fallback
        produtos = getDefaultProdutos();
        categorias = getDefaultCategorias();
        showToast('Usando dados padrão (offline)', 'warning');
    }
    
    renderizarProdutos();
}
```

---

### 6. **LocalStorage Sem Validação**
**Problema:** Carrinho corrompido causa crash ao fazer parse.

**Solução:**
```javascript
function carregarCarrinho() {
    try {
        const carrinhoSalvo = localStorage.getItem('carrinho');
        if (carrinhoSalvo) {
            carrinho = JSON.parse(carrinhoSalvo);
            if (!Array.isArray(carrinho)) {
                carrinho = [];
                throw new Error('Formato inválido');
            }
        }
    } catch (erro) {
        console.error('Erro ao carregar carrinho:', erro);
        carrinho = [];
        localStorage.removeItem('carrinho');
        showToast('Carrinho resetado', 'warning');
    }
    atualizarCarrinho();
}

function salvarCarrinho() {
    try {
        localStorage.setItem('carrinho', JSON.stringify(carrinho));
    } catch (erro) {
        console.error('Erro ao salvar carrinho:', erro);
        showToast('Não foi possível salvar carrinho', 'error');
    }
}
```

---

### 7. **Funções Faltando - Abas Admin**
**Problema:** `mudarAbaAdmin()` não existe, causando erro ao clicar nas abas.

**Solução:**
```javascript
function mudarAbaAdmin(nomeAba) {
    // Remover classe ativo de todas as abas
    document.querySelectorAll('.admin-tab').forEach(tab => {
        tab.classList.remove('ativo');
    });
    document.querySelectorAll('.admin-tab-content').forEach(content => {
        content.classList.remove('ativo');
    });
    
    // Adicionar classe ativo à aba selecionada
    document.querySelector(`[data-tab="${nomeAba}"]`).classList.add('ativo');
    document.getElementById(`tab${nomeAba.charAt(0).toUpperCase() + nomeAba.slice(1)}`).classList.add('ativo');
}

function mudarAbaEstoque(nomePainel) {
    document.querySelectorAll('.estoque-tab').forEach(tab => {
        tab.classList.remove('ativo');
    });
    document.querySelectorAll('.estoque-panel').forEach(panel => {
        panel.classList.remove('ativo');
    });
    
    document.querySelector(`[data-estoque-panel="${nomePainel}"]`).classList.add('ativo');
    document.getElementById(`estoquePainel${nomePainel.charAt(0).toUpperCase() + nomePainel.slice(1)}`).classList.add('ativo');
}
```

---

## ✅ Checklist de Correções

- [ ] Corrigir `produtos.json` com dados reais
- [ ] Adicionar classe CSS `.modal.ativo { display: flex; }`
- [ ] Implementar função `abrirModal()` e `fecharModal()`
- [ ] Adicionar todos os event listeners faltando
- [ ] Inicializar Swiper com `inicializarSwiper()`
- [ ] Adicionar try-catch em Supabase
- [ ] Adicionar try-catch em LocalStorage
- [ ] Implementar funções `mudarAbaAdmin()` e `mudarAbaEstoque()`
- [ ] Testar no console do navegador (F12)
- [ ] Usar `console.error()` para encontrar bugs rapidamente

---

## 🛠️ Ferramentas para Debug

1. **F12 → Console**: Ver erros em tempo real
2. **F12 → Network**: Ver requisições Supabase
3. **F12 → Application → Local Storage**: Ver dados salvos
4. **F12 → Debugger**: Pausar execução com breakpoints

