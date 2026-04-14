# 🔧 SOLUÇÃO - Corrigindo o Bug dos Índices

## Sumário da Solução

**Problema:** Linha 504 usa `idx` do array filtrado, enquanto os botões usam `dbRecords.findIndex()`. Resultado: cliques na linha carregam registos errados quando há filtro ativo.

**Solução Recomendada:** Tornar consistentes os dois métodos de passar índices, usando sempre `dbRecords.findIndex()`.

---

## Opção 1: Corrigir apenas a linha 504 (MAIS SIMPLES)

### Código Atual (ERRADO) - Linha 504

```javascript
tr.onclick = () => loadFromDB(idx);
```

### Código Corrigido

```javascript
tr.onclick = () => loadFromDB(dbRecords.findIndex(r => r.codigo === rec.codigo));
```

### Contexto Completo da Função

**Localização:** [Linhas 486-512 em index.html](index.html#L486-L512)

**Código ANTES:**
```javascript
function renderDatabase(data) {
    const body = document.getElementById('db-body');
    body.innerHTML = '';
    data.forEach((rec, idx) => {
        const tr = document.createElement('tr');
        tr.className = 'hover:bg-blue-50/30 transition-colors group cursor-pointer';
        tr.innerHTML = `
            <td class="px-8 py-5 text-slate-400 font-mono text-xs">${rec.data_registo || '---'}</td>
            <td class="px-8 py-5 font-black text-slate-900 uppercase">${rec.codigo || '---'}</td>
            <td class="px-8 py-5 font-bold text-slate-600">${rec.titulo || 'N/A'}</td>
            <td class="px-8 py-5 text-center font-black text-blue-600">${rec.num_trolleys || 0}</td>
            <td class="px-8 py-5 flex justify-center gap-2">
                <button onclick="viewFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); event.stopPropagation();" class="p-2 bg-slate-600 text-white rounded-lg shadow-md hover:scale-110 transition-transform" title="Visualizar"><i data-lucide="eye" class="w-4 h-4"></i></button>
                <button onclick="loadFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); event.stopPropagation();" class="p-2 bg-blue-600 text-white rounded-lg shadow-md hover:scale-110 transition-transform" title="Editar"><i data-lucide="edit-2" class="w-4 h-4"></i></button>
                <button onclick="duplicateFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); event.stopPropagation();" class="p-2 bg-amber-600 text-white rounded-lg shadow-md hover:scale-110 transition-transform" title="Duplicar"><i data-lucide="copy" class="w-4 h-4"></i></button>
                <button onclick="deleteFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); event.stopPropagation();" class="p-2 bg-red-600 text-white rounded-lg shadow-md hover:scale-110 transition-transform" title="Apagar"><i data-lucide="trash-2" class="w-4 h-4"></i></button>
            </td>
        `;
        tr.onclick = () => loadFromDB(idx);  // ❌ PROBLEMA AQUI
        body.appendChild(tr);
    });
    lucide.createIcons();
}
```

**Código DEPOIS:**
```javascript
function renderDatabase(data) {
    const body = document.getElementById('db-body');
    body.innerHTML = '';
    data.forEach((rec, idx) => {
        const tr = document.createElement('tr');
        tr.className = 'hover:bg-blue-50/30 transition-colors group cursor-pointer';
        tr.innerHTML = `
            <td class="px-8 py-5 text-slate-400 font-mono text-xs">${rec.data_registo || '---'}</td>
            <td class="px-8 py-5 font-black text-slate-900 uppercase">${rec.codigo || '---'}</td>
            <td class="px-8 py-5 font-bold text-slate-600">${rec.titulo || 'N/A'}</td>
            <td class="px-8 py-5 text-center font-black text-blue-600">${rec.num_trolleys || 0}</td>
            <td class="px-8 py-5 flex justify-center gap-2">
                <button onclick="viewFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); event.stopPropagation();" class="p-2 bg-slate-600 text-white rounded-lg shadow-md hover:scale-110 transition-transform" title="Visualizar"><i data-lucide="eye" class="w-4 h-4"></i></button>
                <button onclick="loadFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); event.stopPropagation();" class="p-2 bg-blue-600 text-white rounded-lg shadow-md hover:scale-110 transition-transform" title="Editar"><i data-lucide="edit-2" class="w-4 h-4"></i></button>
                <button onclick="duplicateFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); event.stopPropagation();" class="p-2 bg-amber-600 text-white rounded-lg shadow-md hover:scale-110 transition-transform" title="Duplicar"><i data-lucide="copy" class="w-4 h-4"></i></button>
                <button onclick="deleteFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); event.stopPropagation();" class="p-2 bg-red-600 text-white rounded-lg shadow-md hover:scale-110 transition-transform" title="Apagar"><i data-lucide="trash-2" class="w-4 h-4"></i></button>
            </td>
        `;
        tr.onclick = () => loadFromDB(dbRecords.findIndex(r => r.codigo === rec.codigo));  // ✅ CORRIGIDO
        body.appendChild(tr);
    });
    lucide.createIcons();
}
```

### Vantagens
- ✅ Uma lina apenas
- ✅ Consistente com os botões
- ✅ Sem mudanças em outras funções
- ✅ Funciona com arrays filtrados ou não

### Desvantagens
- ⚠️ O `dbRecords.findIndex()` é executado a cada clique (performance mínima)

---

## Opção 2: Melhor Performance (Caching do índice)

Se houver preocupações com performance (muitos registos), pode-se guardar o índice no atributo de dados:

### Código Alternativo

**Código ANTES:**
```javascript
function renderDatabase(data) {
    const body = document.getElementById('db-body');
    body.innerHTML = '';
    data.forEach((rec, idx) => {
        const tr = document.createElement('tr');
        tr.className = 'hover:bg-blue-50/30 transition-colors group cursor-pointer';
        tr.innerHTML = `...`;
        tr.onclick = () => loadFromDB(idx);  // ❌ idx errado
        body.appendChild(tr);
    });
    lucide.createIcons();
}
```

**Código DEPOIS:**
```javascript
function renderDatabase(data) {
    const body = document.getElementById('db-body');
    body.innerHTML = '';
    data.forEach((rec, idx) => {
        // Guardar o índice verdadeiro em dbRecords
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

### Vantagens
- ✅ O `findIndex()` é executado uma vez por renderização
- ✅ Consistente
- ✅ Mais rápido em buscas com muitos resultados

---

## Implementação da Solução

### Passo 1: Fazer o backup
```bash
cp index.html index.html.backup
```

### Passo 2: Aplicar a fix (Opção 1 - recomendada)

Substituir apenas esta linha:

**De:**
```javascript
tr.onclick = () => loadFromDB(idx);
```

**Para:**
```javascript
tr.onclick = () => loadFromDB(dbRecords.findIndex(r => r.codigo === rec.codigo));
```

### Passo 3: Testar

1. Abrir a aplicação
2. Pesquisar por um código com múltiplos resultados (ex: "TRA75")
3. Clicar na linha (não no botão)
4. Verificar se carrega o registo correto

---

## Validação da Solução

### Antes da fix

```
Pesquisa "TRA75":
  result[0] → TRA759 (dbRecords[150])
  result[1] → TRA759_R2 (dbRecords[151])

Clica em TRA759 (linha):
  tr.onclick = () => loadFromDB(0)
  ❌ Carrega dbRecords[0] (ERRADO)

Clica em botão "Editar":
  loadFromDB(dbRecords.findIndex(r => r.codigo === 'TRA759'))
  ✅ Carrega dbRecords[150] (CORRETO)
```

### Depois da fix

```
Pesquisa "TRA75":
  result[0] → TRA759 (dbRecords[150])
  result[1] → TRA759_R2 (dbRecords[151])

Clica em TRA759 (linha):
  tr.onclick = () => loadFromDB(dbRecords.findIndex(...'TRA759'...))
  ✅ Carrega dbRecords[150] (CORRETO)

Clica em botão "Editar":
  loadFromDB(dbRecords.findIndex(r => r.codigo === 'TRA759'))
  ✅ Carrega dbRecords[150] (CORRETO)
```

---

## Nota Importante: Outras Funções Afetadas?

A mesma linha é usada para `viewFromDB`, `loadFromDB`, `duplicateFromDB`, e `deleteFromDB`. 

No entanto:
- ✅ Os **botões** já usam `dbRecords.findIndex()` (linha 498-501)
- ✅ Apenas a **linha 504** (`tr.onclick`) usa `idx` errado

Depois desta fix, todos os métodos estarão consistentes.

