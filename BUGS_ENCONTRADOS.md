# 🐛 Bugs Encontrados e Soluções

## ❌ BUG 1: Categorias não carregam na página principal

### Problema
- As categorias aparecem como "Carregando categorias..." mas não renderizam
- O arquivo `MELHORIAS.md` mostrou que `fecharModal()` está usando `display: none` sem classe `.ativo`

### Causa Raiz
```javascript
// ERRADO - Não funciona com display: none
.modal {
    display: none; // Estes atributos não fazem efeito em display: none
    justify-content: center;
    align-items: center;
}

// Função antiga que não abre modais corretamente
function fecharModal() { 
    document.querySelectorAll('.modal').forEach(m => m.style.display = 'none'); 
}
```

### Solução Implementada
```javascript
// CORRETO - CSS deve ter classe .ativo para display: flex
.modal {
    display: none; /* Padrão: escondido */
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0,0,0,0.5);
    z-index: 2000;
}

.modal.ativo {
    display: flex; /* Quando .ativo é adicionado */
    justify-content: center;
    align-items: center;
}

// Funções corretas
function abrirModal(modalId) {
    const modal = document.getElementById(modalId);
    if(modal) modal.classList.add('ativo');
}

function fecharModal(modalId) {
    if(modalId) {
        const modal = document.getElementById(modalId);
        if(modal) modal.classList.remove('ativo');
    } else {
        document.querySelectorAll('.modal.ativo').forEach(m => m.classList.remove('ativo'));
    }
}
```

---

## ❌ BUG 2: Categorias não rendem após carregar do Supabase

### Problema
- Função `renderizarCategoriasFront()` é chamada mas nada aparece
- `categoriaAtiva` fica `null`

### Causa Raiz
```javascript
// Função antiga está incompleta
async function carregarCategorias() {
    try {
        const { data, error } = await supabaseClient.from('categorias').select('*').order('ordem');
        if (!error && data && data.length > 0) {
            categorias = data;
        } else {
            categorias = getDefaultCategorias();
            // Não salva no Supabase
        }
    } catch(e) { 
        console.error("Erro Supabase categorias:", e);
        categorias = getDefaultCategorias(); 
    }
    // renderizarCategoriasFront(); <- FALTAVA CHAMAR
    // renderizarAdminCategorias();
}
```

### Solução
```javascript
async function carregarCategorias() {
    try {
        const { data, error } = await supabaseClient
            .from('categorias')
            .select('*')
            .order('ordem');
        
        if (!error && data && data.length > 0) {
            categorias = data;
        } else {
            categorias = getDefaultCategorias();
            // Tenta salvar os padrões
            for (let cat of categorias) {
                await supabaseClient.from('categorias').upsert(cat).catch(e=>console.log);
            }
        }
    } catch(e) { 
        console.error("Erro Supabase categorias:", e);
        categorias = getDefaultCategorias(); 
    }
    
    // IMPORTANTE: Renderizar APÓS carregar
    localStorage.setItem('docesdi_categorias', JSON.stringify(categorias));
    renderizarCategoriasFront();      // ← ADICIONADO
    renderizarAdminCategorias();       // ← ADICIONADO
    popularSelectCategorias();         // ← ADICIONADO
}
```

---

## ❌ BUG 3: Produtos não carregam por categoria

### Problema
- Ao clicar em uma categoria, o Swiper fica vazio
- Função `carregarProdutosPorCategoria()` não renderiza nada

### Causa Raiz
```javascript
// Função antiga
function carregarProdutosPorCategoria() {
    if (!categoriaAtiva) return;
    let filtrados = produtos.filter(p => p.categoria_id === categoriaAtiva);
    const searchTerm = document.getElementById('searchInput')?.value.toLowerCase() || '';
    if (searchTerm) {
        filtrados = filtrados.filter(p => 
            p.nome.toLowerCase().includes(searchTerm) || 
            (p.descricao || '').toLowerCase().includes(searchTerm)
        );
    }
    renderizarProdutosSwiper(filtrados);
    // PROBLEMA: Swiper não reinicializa se já estava criado
}
```

### Solução
```javascript
function carregarProdutosPorCategoria() {
    if (!categoriaAtiva) return;
    
    let filtrados = produtos.filter(p => p.categoria_id === categoriaAtiva);
    
    const searchTerm = document.getElementById('searchInput')?.value.toLowerCase() || '';
    if (searchTerm) {
        filtrados = filtrados.filter(p => 
            p.nome.toLowerCase().includes(searchTerm) || 
            (p.descricao || '').toLowerCase().includes(searchTerm)
        );
    }
    
    renderizarProdutosSwiper(filtrados);
    
    // IMPORTANTE: Atualizar o Swiper após renderizar
    if (swiperInstance) {
        swiperInstance.update();
    } else {
        inicializarSwiper();
    }
}

function inicializarSwiper() {
    const wrapper = document.getElementById('swiperProdutosWrapper');
    if (!wrapper || wrapper.children.length === 0) return;
    
    // Destruir instância antiga se existir
    if (swiperInstance) {
        swiperInstance.destroy(true, true);
    }
    
    // Criar nova instância
    swiperInstance = new Swiper('#productSwiper', {
        slidesPerView: 1,
        spaceBetween: 20,
        pagination: { el: '.swiper-pagination', clickable: true },
        navigation: { 
            nextEl: '.swiper-button-next', 
            prevEl: '.swiper-button-prev' 
        },
        breakpoints: { 
            640: { slidesPerView: 2 }, 
            768: { slidesPerView: 3 }, 
            1024: { slidesPerView: 4 } 
        }
    });
}
```

---

## ❌ BUG 4: Adicionar categoria não grava

### Problema
- Clica no botão "➕ Adicionar" em Categorias
- Nada acontece, categoria não aparece na lista

### Causa Raiz
```javascript
// Função antiga incompleta
async function adicionarCategoria() {
    const nome = document.getElementById('categoriaNome').value.trim();
    const icone = document.getElementById('categoriaIcone').value.trim() || '📁';
    if (!nome) { showToast("Nome obrigatório!",'error'); return; }
    
    const novoId = Date.now();
    const nova = { id: novoId, nome, icone, ordem: categorias.length+1 };
    categorias.push(nova);
    
    showToast(`Categoria "${nome}" adicionada!`);
    localStorage.setItem('docesdi_categorias', JSON.stringify(categorias));
    
    // PROBLEMA: Não limpa os inputs e não renderiza corretamente
    // PROBLEMA: Botão "Salvar Categorias" não faz nada
    
    await supabaseClient.from('categorias').insert(nova).catch(e=>console.log);
    // Não estava renderizando após inserir
}
```

### Solução
```javascript
async function adicionarCategoria() {
    const nome = document.getElementById('categoriaNome').value.trim();
    const icone = document.getElementById('categoriaIcone').value.trim() || '📁';
    
    if (!nome) { 
        showToast("Nome da categoria obrigatório!",'error'); 
        return; 
    }
    
    const novoId = Date.now();
    const nova = { id: novoId, nome, icone, ordem: categorias.length + 1 };
    
    categorias.push(nova);
    
    // Salvar localmente
    localStorage.setItem('docesdi_categorias', JSON.stringify(categorias));
    
    // Salvar no Supabase
    const { error } = await supabaseClient
        .from('categorias')
        .insert(nova);
    
    if(error) {
        showToast(`Erro ao salvar: ${error.message}`,'error');
        // Remover da lista local se falhar
        categorias = categorias.filter(c => c.id !== novoId);
        return;
    }
    
    showToast(`Categoria "${nome}" adicionada com sucesso!`, 'success');
    
    // IMPORTANTE: Limpar inputs
    document.getElementById('categoriaNome').value = '';
    document.getElementById('categoriaIcone').value = '';
    
    // IMPORTANTE: Renderizar para refletir a nova categoria
    renderizarCategoriasFront();
    renderizarAdminCategorias();
    popularSelectCategorias();
}
```

---

## ❌ BUG 5: Botão "Salvar Categorias" não faz nada

### Problema
```javascript
// Código antigo - apenas mostra mensagem
document.getElementById('btnSalvarCategorias')?.addEventListener('click', 
    () => showToast("Categorias salvas!")
);
```

### Solução
```javascript
async function salvarCategorias() {
    try {
        for (let cat of categorias) {
            const { error } = await supabaseClient
                .from('categorias')
                .upsert(cat);
            
            if(error) {
                showToast(`Erro ao salvar ${cat.nome}: ${error.message}`,'error');
                return;
            }
        }
        
        localStorage.setItem('docesdi_categorias', JSON.stringify(categorias));
        showToast("Todas as categorias salvas com sucesso!", 'success');
        renderizarCategoriasFront();
    } catch(e) {
        showToast(`Erro: ${e.message}`,'error');
    }
}

// Adicionar event listener correto
document.getElementById('btnSalvarCategorias')?.addEventListener('click', salvarCategorias);
```

---

## ✅ Checklist de Implementação

- [ ] **Passo 1**: Adicionar CSS correto para `.modal.ativo`
- [ ] **Passo 2**: Implementar `abrirModal()` e `fecharModal()` corrigidos
- [ ] **Passo 3**: Chamar `renderizarCategoriasFront()` em `carregarCategorias()`
- [ ] **Passo 4**: Implementar `inicializarSwiper()` corrigido
- [ ] **Passo 5**: Chamar `swiperInstance.update()` em `carregarProdutosPorCategoria()`
- [ ] **Passo 6**: Implementar `adicionarCategoria()` corrigido com renderização
- [ ] **Passo 7**: Implementar `salvarCategorias()` completo
- [ ] **Passo 8**: Testar adicionar categoria nova
- [ ] **Passo 9**: Testar clique em categoria para carregar produtos
- [ ] **Passo 10**: Verificar no console (F12) se há erros

---

## 🧪 Como Testar

1. **Abrir Console** (F12)
2. **Ir para Admin** → Categorias
3. **Adicionar categoria teste**: Nome = "Test", Ícone = "🧪"
4. **Clicar em "➕ Adicionar"**
5. **Clicar em "💾 Salvar Categorias"**
6. **Voltar para página principal**
7. **Verificar se "Test 🧪" aparece no carrossel**
8. **Clicar na categoria**
9. **Verificar se produtos aparecem no carrossel principal**

---

## 📋 Resumo dos Erros

| Bug | Causa | Solução |
|-----|-------|---------|
| Categorias não rendem | `renderizarCategoriasFront()` não era chamado | Adicionar chamada em `carregarCategorias()` |
| Produtos não carregam | Swiper não atualizava após renderizar | Chamar `swiperInstance.update()` |
| Adicionar categoria não grava | Falta de renderização após inserir | Chamar `renderizarCategoriasFront()` |
| Salvar categorias não faz nada | Função vazia | Implementar com loop de upsert |
| Modal não abre | CSS sem `.ativo` | Usar `classList.add('ativo')` |

