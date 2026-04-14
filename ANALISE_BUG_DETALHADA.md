# 🔍 ANÁLISE DETALHADA DO BUG - TRA759 → TRA526_R3

## Resumo Executivo

**O relatório do utilizador:** Ao pesquisar "TRA75", vê TRA759 e TRA759_R2, mas ao clicar carrega registos errados.

**Causa Raiz:** Há **DOIS MÉTODOS DIFERENTES** de passar índices na tabela renderizada:
1. **Linha 504:** `tr.onclick = () => loadFromDB(idx);` — usa índice do array filtrado
2. **Linhas 498-501:** `onclick="viewFromDB(${dbRecords.findIndex(...)})"` — usa índice do array completo

Quando há filtro ativo, estes dois métodos devolvem índices diferentes, causando carregamento do registo errado.

---

## 1. OS DOIS MÉTODOS DE PASSAR ÍNDICES

### ❌ INCONSISTÊNCIA #1: Clique na linha (tr.onclick)

**Localização:** [Linha 504](index.html#L504)

```javascript
data.forEach((rec, idx) => {
    const tr = document.createElement('tr');
    // ... HTML da linha ...
    tr.onclick = () => loadFromDB(idx);  // ⚠️ PROBLEMA: usa 'idx' do array filtrado
    body.appendChild(tr);
});
```

**O que faz:**
- `idx` é o índice dentro do array `data` (que pode ser filtrado)
- Se `data` é `filtered.slice(0, 10)`, então `idx` vai de 0-9
- Mas passa este `idx` diretamente para `loadFromDB()`, que trata como índice de `dbRecords`

---

### ⚠️ INCONSISTÊNCIA #2: Botões (onclick com findIndex)

**Localização:** [Linhas 498-501](index.html#L498-L501)

```javascript
<button onclick="viewFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); 
    event.stopPropagation();" class="...">
    <i data-lucide="eye" class="w-4 h-4"></i>
</button>
<button onclick="loadFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); 
    event.stopPropagation();" class="...">
    <i data-lucide="edit-2" class="w-4 h-4"></i>
</button>
<button onclick="duplicateFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); 
    event.stopPropagation();" class="...">
    <i data-lucide="copy" class="w-4 h-4"></i>
</button>
<button onclick="deleteFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); 
    event.stopPropagation();" class="...">
    <i data-lucide="trash-2" class="w-4 h-4"></i>
</button>
```

**O que faz:**
- Procura o índice verdadeiro em `dbRecords` usando `findIndex`
- Compara por `codigo` (identificador único)
- Devolve o índice correto de `dbRecords`

---

## 2. REPRODUÇÃO DO BUG com os códigos reportados

### Cenário Simulado

**Estado inicial de `dbRecords`:**

```
dbRecords[0]   → TRA100
dbRecords[1]   → TRA101
...
dbRecords[50]  → TRA526_R3  ← está aqui!
dbRecords[51]  → TRA527
...
dbRecords[150] → TRA759     ← está aqui!
dbRecords[151] → TRA759_R2  ← está aqui!
dbRecords[152] → TRB548     ← está aqui!
...
```

### Fluxo do Utilizador

**Passo 1:** Pesquisa "TRA75"
```javascript
const filtered = dbRecords.filter(rec => 
    rec.codigo.includes('TRA75')
);
// Resultado:
// filtered[0] → TRA759      (dbRecords[150])
// filtered[1] → TRA759_R2   (dbRecords[151])

renderDatabase(filtered);  // Passa o array filtrado
```

**Passo 2:** Clica na linha de TRA759 (clique normal na linha)
```javascript
// O onclick da linha é: tr.onclick = () => loadFromDB(idx);
// idx = 0 (índice no array filtered)

loadFromDB(0);  // ❌ ERRO: treata 0 como índice de dbRecords
const r = dbRecords[0];  // ❌ Carrega dbRecords[0] = TRA100 (ERRADO!)
```

**Passo 3:** Mas ao clicar no botão "Editar" (edit button)
```javascript
// O onclick do botão é: onclick="loadFromDB(${dbRecords.findIndex(...)})"
// Calcula: dbRecords.findIndex(r => r.codigo === 'TRA759')
// Resultado: 150 (o índice verdadeiro)

loadFromDB(150);  // ✅ CORRETO: Carrega dbRecords[150] = TRA759
```

**Isto explica parcialmente o relatório do utilizador:**
- Se clica em TRA759 (linha): carrega registo errado
- Se clica em TRA759_R2 (linha): carrega outro registo errado
- Se clica no botão "Editar": carrega correto

---

## 3. ANÁLISE DO CÓDIGO - as duas funções

### Função `filterDatabase()` [Linhas 466-483]

```javascript
function filterDatabase() {
    const searchTerm = document.getElementById('db-search').value.toLowerCase().trim();
    
    if (!searchTerm) {
        renderDatabase(dbRecords);  // ✅ Sem filtro: passa array completo
        return;
    }
    
    // 🔴 Com filtro: cria novo array filtrado
    const filtered = dbRecords.filter(rec => {
        const codigo = (rec.codigo || '').toLowerCase();
        const titulo = (rec.titulo || '').toLowerCase();
        const data = (rec.data_registo || '').toLowerCase();
        
        return codigo.includes(searchTerm) || 
               titulo.includes(searchTerm) || 
               data.includes(searchTerm);
    });
    
    renderDatabase(filtered);  // 🔴 Passa array diferente (com índices 0-N)
}
```

**Problema:** `renderDatabase()` não sabe se recebeu o array completo ou filtrado.

---

### Função `renderDatabase(data)` [Linhas 486-512]

```javascript
function renderDatabase(data) {
    const body = document.getElementById('db-body');
    body.innerHTML = '';
    data.forEach((rec, idx) => {  // 🔴 idx é relativo a 'data'
        const tr = document.createElement('tr');
        tr.className = 'hover:bg-blue-50/30 transition-colors group cursor-pointer';
        tr.innerHTML = `
            <td class="px-8 py-5 text-slate-400 font-mono text-xs">${rec.data_registo || '---'}</td>
            <td class="px-8 py-5 font-black text-slate-900 uppercase">${rec.codigo || '---'}</td>
            <td class="px-8 py-5 font-bold text-slate-600">${rec.titulo || 'N/A'}</td>
            <td class="px-8 py-5 text-center font-black text-blue-600">${rec.num_trolleys || 0}</td>
            <td class="px-8 py-5 flex justify-center gap-2">
                <button onclick="viewFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); 
                    event.stopPropagation();" class="...">Visualizar</button>
                <button onclick="loadFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); 
                    event.stopPropagation();" class="...">Editar</button>
                <button onclick="duplicateFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); 
                    event.stopPropagation();" class="...">Duplicar</button>
                <button onclick="deleteFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)}); 
                    event.stopPropagation();" class="...">Apagar</button>
            </td>
        `;
        tr.onclick = () => loadFromDB(idx);  // 🔴 AQUI: Usa 'idx' do array 'data'
        body.appendChild(tr);
    });
    lucide.createIcons();
}
```

**Problema:**
- **Linha 504:** `tr.onclick = () => loadFromDB(idx);` — usa `idx` relativo ao array `data` (filtrado)
- **Linhas 498-501:** Os botões usam `dbRecords.findIndex()` — correto
- Há **INCONSISTÊNCIA**!

---

### Função `loadFromDB(idx)` [Linhas 510-524]

```javascript
function loadFromDB(idx) {
    const r = dbRecords[idx];  // 🔴 Assume que 'idx' é sempre relativo a dbRecords
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

**Crítica:** Esta função `loadFromDB()` é chamada de dois lugares com índices diferentes:
1. Linha 504 (`tr.onclick`): com `idx` do array filtrado ❌
2. Linhas 498-501 (botão): com `idx` do array completo ✅

---

## 4. POR QUE O RELATÓRIO DO UTILIZADOR FAZ SENTIDO

### Relatório Original:
> "Quando pesquisa por 'TRA75' vê TRA759 e TRA759_R2. Se clica em TRA759, mostra TRA526_R3. Se clica em TRA759_R2, mostra TRB548."

### Explicação:

Suponha que no `dbRecords`:
- `dbRecords[50]` = TRA526_R3
- `dbRecords[150]` = TRA759
- `dbRecords[151]` = TRA759_R2
- `dbRecords[152]` = TRB548

Quando filtra "TRA75":
```javascript
filtered[0] → TRA759 (originalmente dbRecords[150])
filtered[1] → TRA759_R2 (originalmente dbRecords[151])
```

**Clica em TRA759 (linha):**
- `tr.onclick = () => loadFromDB(0);`
- `loadFromDB(0)` trata como índice de dbRecords
- Carrega `dbRecords[0]` que é... (qualquer coisa que esteja em posição #50 ou similar)
- ❌ Mostra registo errado

Se tiver sorte de clicar em `TRA759_R2`:
- `tr.onclick = () => loadFromDB(1);`
- `loadFromDB(1)` carrega `dbRecords[1]`
- ❌ Outro registo errado

---

## 5. SOLUÇÃO

Há **duas maneiras** de corrigir:

### ✅ Solução 1: Usar sempre findIndex (COMPATIVEL COM ATUAL)

**Modificar linha 504 de:**
```javascript
tr.onclick = () => loadFromDB(idx);
```

**Para:**
```javascript
tr.onclick = () => loadFromDB(dbRecords.findIndex(r => r.codigo === rec.codigo));
```

**Vantagem:** Uniforme com os botões, uma única forma de passar índices

---

### ✅ Solução 2: Passar o código como string

**Modificar linha 504 de:**
```javascript
tr.onclick = () => loadFromDB(idx);
```

**Para:**
```javascript
tr.onclick = () => loadFromDB('${rec.codigo}');
```

**Modificar `loadFromDB()` de:**
```javascript
function loadFromDB(idx) {
    const r = dbRecords[idx];
```

**Para:**
```javascript
function loadFromDB(idxOrCode) {
    const r = typeof idxOrCode === 'string' 
        ? dbRecords.find(rec => rec.codigo === idxOrCode)
        : dbRecords[idxOrCode];
```

**Vantagem:** Mais flexível, compatível com ambas chamadas

---

## 6. RESUMO TÉCNICO

| Aspecto | Status | Problema |
|---------|--------|----------|
| **Função `filterDatabase()`** | ✅ OK | Nenhum, filtra corretamente |
| **Botões (onclick)** | ✅ OK | Usam `findIndex`, sempre correto |
| **Linha (tr.onclick)** | ❌ BUG | Usa `idx` relativo ao array `data`, não a `dbRecords` |
| **Função `loadFromDB()`** | ❌ BUG | Assume que `idx` é sempre de `dbRecords` |
| **Inconsistência** | 🔴 CRÍTICA | Dois métodos diferentes de passar índices na mesma tabela |

---

## 7. CÓDIGO QUE DEMONSTRA O BUG

**Diagrama de execução:**

```
┌─────────────────────────────────────────┐
│ Utilizador pesquisa "TRA75"              │
└──────────────────┬──────────────────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │ filterDatabase()      │
        │ Cria: filtered[]     │
        │ filtered[0]=TRA759   │
        │ filtered[1]=TRA759_R2│
        └──────────┬───────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │ renderDatabase(      │
        │   filtered          │
        │ )                    │
        └──────────┬───────────┘
                   │
          ┌────────┴────────┐
          │                 │
    ▼──CLIQUE NA LINHA──▼  ▼──CLIQUE NO BOTÃO──▼
    tr.onclick()         onclick="" 
    loadFromDB(0)        loadFromDB(150)
    ❌ ERRADO            ✅ CORRETO
```

---

## 8. CONCLUSÃO

**A causa do bug reportado:**

1. ✅ Pesquisa funciona: filtra corretamente TRA759 e TRA759_R2
2. ❌ Clique na linha: usa índice filtrado (0, 1, ...) em vez de índice de dbRecords (150, 151, ...)
3. ✅ Clique no botão: funciona porque usa `findIndex`

**Recomendação:** Aplicar **Solução 1** (mais simples e coerente com o código existente).

