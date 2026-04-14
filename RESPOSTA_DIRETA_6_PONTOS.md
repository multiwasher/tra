# ✅ RESPOSTA DIRETA AOS 6 PONTOS SOLICITADOS

## 1️⃣ Qual é o verdadeiro índice do TRA759, TRA759_R2 e TRA526_R3 em dbRecords?

A partir da análise do código, não há dados de teste embutidos no arquivo. Os dados são carregados via Google Sheets API. 

**No entanto, sabemos que:**

- **TRA759** tem um índice específico em `dbRecords` (vamos chamar `idx_TRA759`)
- **TRA759_R2** tem um índice específico em `dbRecords` (vamos chamar `idx_TRA759_R2`)
- **TRA526_R3** tem um índice específico em `dbRecords` (vamos chamar `idx_TRA526_R3`)

Estes índices são **encontrados dinamicamente** usando `findIndex()`:

```javascript
dbRecords.findIndex(r => r.codigo === 'TRA759')      // = idx_TRA759
dbRecords.findIndex(r => r.codigo === 'TRA759_R2')   // = idx_TRA759_R2
dbRecords.findIndex(r => r.codigo === 'TRA526_R3')   // = idx_TRA526_R3
```

**Eles NÃO são necessariamente consecutivos ou em qualquer ordem especial.**

Para encontrar os índices exatos, seria necessário:
- Executar a aplicação com dados reais
- Abrir a consola do navegador (F12)
- Executar: `dbRecords.findIndex(r => r.codigo === 'TRA759')`

---

## 2️⃣ Há registos com códigos duplicados ou similares em dbRecords?

**Procura realizada:** Grep search pelo arquivo `index.html`

### Resultados:

**Encontrados:**
- `TRA759` - mencionado 1 vez (linha 189 como placeholder de exemplo)
- `TRA759` - mencionado 1 vez (linha 332 como código de teste padrão)

**Não foram encontrados:**
- ❌ Registos literalmente duplicados (mesmo `codigo`)
- ⚠️ É possível que existam nos dados do Google Sheets (não embutidos no HTML)

### Padrões de Código Esperados:

O padrão de código segue esta convenção:
- `TRA### ` (ex: TRA759, TRA526)
- `TRA###_R# ` (ex: TRA759_R2, sufixo "_R2", "_R3" = revisões)
- `TRB###` (começam com TRB em vez de TRA)

Este padrão não causa duplicação automática, mas **codificações similares podem causar problemas em buscas por substring.**

### Exemplo:
Se pesquisar "TRA75", aparecem:
- TRA759
- TRA759_R2
- (qualquer outro que tenha "TRA75" no código)

Se houver `TRA750`, `TRA751`, `TRA752`, etc., todos aparecerão também!

---

## 3️⃣ Qual é a causa REAL do bug?

### ❌ NÃO É:
- ❌ Não é haver registos duplicados
- ❌ Não é falha na função `findIndex`
- ❌ Não é uma coincidência nos dados

### ✅ É EXATAMENTE:

**A INCONSISTÊNCIA DE ÍNDICES nas duas maneiras diferentes de chamar `loadFromDB()`**

| Elemento | Método de Passar Índice | Qual Índice | Resultado |
|----------|----------------------|------------|-----------|
| **tr.onclick (linha)** | `idx` do forEach | Índice no array **filtrado** (0, 1, 2, ...) | ❌ ERRADO |
| **Botões (buttons)** | `dbRecords.findIndex()` | Índice no array **dbRecords** completo | ✅ CORRETO |

### O Fluxo Exato do Bug:

```javascript
// 1. Utilizador pesquisa "TRA75"
filterDatabase() {
    const filtered = dbRecords.filter(rec => 
        rec.codigo.includes('TRA75')
    );
    renderDatabase(filtered);  // Passa array pequeno
}

// 2. renderDatabase recebe array filtrado
renderDatabase(filtered) {  // data = [TRA759, TRA759_R2]
    data.forEach((rec, idx) => {  // idx = 0, 1
        
        // ❌ PROBLEMA: Passa idx relativo a 'data'
        tr.onclick = () => loadFromDB(idx);  // passa 0 ou 1
        
        // ✅ CORRETO: Procura em dbRecords completo
        button.onclick = `loadFromDB(${dbRecords.findIndex(...)})`  // passa 150, 151
    });
}

// 3. loadFromDB recebe índices inconsistentes
loadFromDB(idx) {
    const r = dbRecords[idx];
    
    // Quando idx=0 (da linha): carrega dbRecords[0] ❌
    // Quando idx=150 (do botão): carrega dbRecords[150] ✅
}
```

---

## 4️⃣ É a linha `tr.onclick` que está a usar idx errado? Ou é o `findIndex`?

### Resposta: **SIM E NÃO**

1. **Não é o `findIndex` que está errado:**
   - `dbRecords.findIndex(r => r.codigo === rec.codigo)` funciona **perfeitamente**
   - Devolve o índice correto em `dbRecords`

2. **É a linha `tr.onclick` que está errada:**
   - `tr.onclick = () => loadFromDB(idx);` usa **o índice errado**
   - `idx` é relativo ao array `data` passado a `renderDatabase()`
   - Deveria ser relativo a `dbRecords`

### Identificação Exata:

**Localização:** [Linha 504 em index.html](index.html#L504)

```javascript
tr.onclick = () => loadFromDB(idx);  // ❌ ERRADO
```

**Deveria Ser:**

```javascript
tr.onclick = () => loadFromDB(dbRecords.findIndex(r => r.codigo === rec.codigo));  // ✅ CORRETO
```

---

## 5️⃣ Trechos de Código que Demonstram o Bug

### Trecho 1: Função renderDatabase - ANTES

[Linhas 486-512 em index.html](index.html#L486-L512)

```javascript
function renderDatabase(data) {
    const body = document.getElementById('db-body');
    body.innerHTML = '';
    data.forEach((rec, idx) => {  // data pode ser filtrado!
        const tr = document.createElement('tr');
        tr.className = 'hover:bg-blue-50/30 transition-colors group cursor-pointer';
        tr.innerHTML = `
            <td class="px-8 py-5 text-slate-400 font-mono text-xs">${rec.data_registo || '---'}</td>
            <td class="px-8 py-5 font-black text-slate-900 uppercase">${rec.codigo || '---'}</td>
            <td class="px-8 py-5 font-bold text-slate-600">${rec.titulo || 'N/A'}</td>
            <td class="px-8 py-5 text-center font-black text-blue-600">${rec.num_trolleys || 0}</td>
            <td class="px-8 py-5 flex justify-center gap-2">
                <button onclick="viewFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); event.stopPropagation();" ...>Ver</button>
                <button onclick="loadFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); event.stopPropagation();" ...>Editar</button>
                <button onclick="duplicateFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); event.stopPropagation();" ...>Dup</button>
                <button onclick="deleteFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); event.stopPropagation();" ...>Apag</button>
            </td>
        `;
        tr.onclick = () => loadFromDB(idx);  // ❌❌❌ PROBLEMA AQUI
        body.appendChild(tr);
    });
    lucide.createIcons();
}
```

### Trecho 2: Função filterDatabase - funcionamento correto

[Linhas 466-483 em index.html](index.html#L466-L483)

```javascript
function filterDatabase() {
    const searchTerm = document.getElementById('db-search').value.toLowerCase().trim();
    
    if (!searchTerm) {
        renderDatabase(dbRecords);
        return;
    }
    
    const filtered = dbRecords.filter(rec => {
        const codigo = (rec.codigo || '').toLowerCase();
        const titulo = (rec.titulo || '').toLowerCase();
        const data = (rec.data_registo || '').toLowerCase();
        
        return codigo.includes(searchTerm) ||  // ← "TRA75" encontra TRA759, TRA759_R2
               titulo.includes(searchTerm) || 
               data.includes(searchTerm);
    });
    
    renderDatabase(filtered);  // ← Passa array PEQUENO com índices 0,1,2...
}
```

### Trecho 3: Função loadFromDB - recebe índices inconsistentes

[Linhas 510-524 em index.html](index.html#L510-L524)

```javascript
function loadFromDB(idx) {
    const r = dbRecords[idx];  // ← ASSUME que idx é sempre de dbRecords
    projectData = {
        code: r.codigo || '', 
        title: r.titulo || '', 
        numTrolleys: parseInt(r.num_trolleys) || 1,
        groups: (r.materiais || []).map((g, i) => ({
            id: g.grupo || String.fromCharCode(65 + i),
            description: g.descricao || '',
            isCustom: !MATERIAL_OPTIONS.includes(g.descricao),
            items: (g.itens || []).map(it => ({ ...it, unit: 'mm' }))
        }))
    };
    updateInputFields();
    render();
    switchView('editor');
    showNotification('Projeto Carregado', 'Edite e grave para atualizar na BD.', 'success');
}
```

### Trecho 4: Comparação lado-a-lado

```javascript
// DOIS MÉTODOS DIFERENTES DE CHAMAR O MESMO CÓDIGO:

// ❌ MÉTODO 1 (linha 504) - ERRADO:
tr.onclick = () => loadFromDB(idx);  // idx = 0 (relativo a 'data')

// ✅ MÉTODO 2 (linhas 498-501) - CORRETO:
button.onclick = () => loadFromDB(dbRecords.findIndex(r => r.codigo === rec.codigo));  // findIndex = 150 (relativo a 'dbRecords')

// A MESMA FUNÇÃO RECEBE ÍNDICES DIFERENTES:
loadFromDB(0)    // ← de MÉTODO 1 (ERRADO)
loadFromDB(150)  // ← de MÉTODO 2 (CORRETO)
```

---

## 6️⃣ Descreva a Solução Necessária

### Solução 1: Uniformizar o Método (RECOMENDADO)

**Linha atual (504):**
```javascript
tr.onclick = () => loadFromDB(idx);
```

**Linha corrigida:**
```javascript
tr.onclick = () => loadFromDB(dbRecords.findIndex(r => r.codigo === rec.codigo));
```

**Por que funciona:**
- Usa **sempre** `dbRecords.findIndex()` para encontrar o índice verdadeiro
- Torna **consistente** com os botões (linhas 498-501)
- Funciona quer `data` seja o array completo ou filtrado
- Uma linha de mudança, sem alterações em outras funções

### Solução 2: Guardar índice original (PERFORMANCE)

Se tiver preocupações com performance (muitos findIndex):

```javascript
function renderDatabase(data) {
    const body = document.getElementById('db-body');
    body.innerHTML = '';
    data.forEach((rec, idx) => {
        // Guard true index once
        const trueIdx = dbRecords.findIndex(r => r.codigo === rec.codigo);
        
        const tr = document.createElement('tr');
        tr.className = 'hover:bg-blue-50/30 transition-colors group cursor-pointer';
        tr.innerHTML = `...`;
        tr.onclick = () => loadFromDB(trueIdx);  // ✅ Usa índice verdadeiro
        body.appendChild(tr);
    });
    lucide.createIcons();
}
```

### Validação da Fix

Depois de aplicar qualquer uma das soluções:

1. **Teste com Filtro:**
   - Pesquisar "TRA75"
   - Clicar em TRA759 (linha)
   - Verificar que carrega TRA759 ✅

2. **Teste sem Filtro:**
   - Limpar filtro
   - Clicar em TRA759 (linha)
   - Verificar que carrega TRA759 ✅

3. **Teste de Botões:**
   - Verificar que botões ainda funcionam ✅

---

## Resumo Final

| Pergunta | Resposta |
|----------|----------|
| **1. Índices?** | Dinâmicos, encontrados via `findIndex()` |
| **2. Duplicados?** | Não em código, possivelmente em dados do Sheets |
| **3. Causa?** | Inconsistência: `idx` vs `findIndex()` |
| **4. Qual está errado?** | `tr.onclick` (linha 504) está errado |
| **5. Trechos?** | Linhas 504, 498-501, 486-512, 466-483, 510-524 |
| **6. Solução?** | Mudar linha 504 para usar `dbRecords.findIndex()` |

## Implementação

**Arquivo:** `index.html`  
**Linha:** 504  
**Mudança:** 1 linha  
**Impacto:** Baixo, sem riscos

```diff
- tr.onclick = () => loadFromDB(idx);
+ tr.onclick = () => loadFromDB(dbRecords.findIndex(r => r.codigo === rec.codigo));
```

