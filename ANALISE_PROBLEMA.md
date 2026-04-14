# Análise do Problema - Duplicação Incorreta

## 1. Função `duplicateFromDB()` - Código Completo

**Localização:** Linhas ~950-974

```javascript
async function duplicateFromDB(idx) {
    const r = dbRecords[idx];  // ⚠️ PROBLEMA: Usa idx diretamente em dbRecords
    const codigoOriginal = r.codigo;
    const novoCodigoBase = codigoOriginal + '_1';
    
    // Carregar os dados do registro original
    projectData = {
        code: novoCodigoBase, 
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
    showNotification('Projeto Duplicado', `Código alterado para "${novoCodigoBase}". Grave para criar a cópia.`, 'success');
}
```

---

## 2. Como a Tabela é Renderizada

**Função `renderDatabase(data)`** - Linhas ~380-410

```javascript
function renderDatabase(data) {
    const body = document.getElementById('db-body');
    body.innerHTML = '';
    data.forEach((rec, idx) => {  // 🔴 idx é o índice dentro de 'data'
        const tr = document.createElement('tr');
        tr.className = 'hover:bg-blue-50/30 transition-colors group cursor-pointer';
        tr.innerHTML = `
            <td class="px-8 py-5 text-slate-400 font-mono text-xs">${rec.data_registo || '---'}</td>
            <td class="px-8 py-5 font-black text-slate-900 uppercase">${rec.codigo || '---'}</td>
            <td class="px-8 py-5 font-bold text-slate-600">${rec.titulo || 'N/A'}</td>
            <td class="px-8 py-5 text-center font-black text-blue-600">${rec.num_trolleys || 0}</td>
            <td class="px-8 py-5 flex justify-center gap-2">
                <button onclick="viewFromDB(${idx}); event.stopPropagation();" ...>
                <button onclick="loadFromDB(${idx}); event.stopPropagation();" ...>
                <button onclick="duplicateFromDB(${idx}); event.stopPropagation();" ...>  🔴 PROBLEMA AQUI
                <button onclick="deleteFromDB(${idx}); event.stopPropagation();" ...>
            </td>
        `;
        body.appendChild(tr);
    });
    lucide.createIcons();
}
```

---

## 3. Função de Filtro/Pesquisa

**Função `filterDatabase()`** - Linhas ~360-376

```javascript
function filterDatabase() {
    const searchTerm = document.getElementById('db-search').value.toLowerCase().trim();
    
    if (!searchTerm) {
        // Se não há termo de busca, mostra todos os registos
        renderDatabase(dbRecords);
        return;
    }
    
    // Filtra por código, título ou data
    const filtered = dbRecords.filter(rec => {  // 🔴 Cria um NOVO array filtrado
        const codigo = (rec.codigo || '').toLowerCase();
        const titulo = (rec.titulo || '').toLowerCase();
        const data = (rec.data_registo || '').toLowerCase();
        
        return codigo.includes(searchTerm) || 
               titulo.includes(searchTerm) || 
               data.includes(searchTerm);
    });
    
    renderDatabase(filtered);  // 🔴 Passa o array filtrado, NÃO o original
}
```

---

## 4. O PROBLEMA - Índice Incorreto

### Fluxo do Erro:

1. **Estado inicial:** `dbRecords` tem 500 registos (índices 0-499)
   - TRA399 está em `dbRecords[150]`
   - TRA398 está em `dbRecords[149]`
   - TRA397 está em `dbRecords[148]`

2. **Utilizador pesquisa "TRA39":** `filterDatabase()` cria um novo array `filtered`
   - `filtered[0]` → TRA399 (originalmente `dbRecords[150]`)
   - `filtered[1]` → TRA398 (originalmente `dbRecords[149]`)
   - `filtered[2]` → TRA397 (originalmente `dbRecords[148]`)

3. **Utilizador clica "Duplicar" em TRA397:**
   - Passa `idx=2` para `duplicateFromDB()`
   - Código faz: `const r = dbRecords[2]`
   - `dbRecords[2]` é **OUTRO projeto COMPLETAMENTE diferente!**
   - Duplica o projeto errado ❌

### Exemplo Visual:

```
ARRAY FILTRADO (o que vê):    ARRAY ORIGINAL (o que busca):
idx=0 → TRA399                dbRecords[0] → TRA100
idx=1 → TRA398                dbRecords[1] → TRA101
idx=2 → TRA397  ← clica aqui  dbRecords[2] → TRA102 ← DUPLICA ESTE!
```

---

## 5. Raiz do Problema

A função `renderDatabase()` recebe um array pode ser:
- ✅ O array completo `dbRecords` (quando sem filtro)
- ❌ Um array filtrado com índices 0-N (quando há filtro)

Mas `duplicateFromDB()` sempre trata o `idx` como se fosse um índice em `dbRecords`.

Isso causa duplicação do registo errado quando há filtro ativo.

---

## Solução Recomendada

**Opção 1 (Simples):** Guardar o objeto completo na renderização
```javascript
// Em renderDatabase, guardar referência ao objeto:
<button onclick="duplicateFromDB('${rec.codigo}'); ...>
// Em duplicateFromDB, procurar pelo código:
function duplicateFromDB(codigo) {
    const r = dbRecords.find(rec => rec.codigo === codigo);
    // ... resto do código
}
```

**Opção 2 (Melhor):** Guardar índice original
```javascript
// Guardar o índice original do dbRecords
const originalData = dbRecords.map((rec, originalIdx) => ({...rec, _originalIdx: originalIdx}));
// Depois guardar isso ao renderizar
<button onclick="duplicateFromDB(${rec._originalIdx}); ...>
```
