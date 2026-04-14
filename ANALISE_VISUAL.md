# 📊 COMPARAÇÃO VISUAL - O Bug em Detalhe

## Contexto: A Tabela Renderizada

```html
<!-- Exemplo de linha renderizada quando filtra "TRA75" -->

<!-- LINHA RENDERIZADA -->
<tr class="hover:bg-blue-50/30 transition-colors group cursor-pointer">
    <td class="px-8 py-5 text-slate-400 font-mono text-xs">14-04-2026</td>
    <td class="px-8 py-5 font-black text-slate-900 uppercase">TRA759</td>
    <td class="px-8 py-5 font-bold text-slate-600">Projeto A</td>
    <td class="px-8 py-5 text-center font-black text-blue-600">2</td>
    <td class="px-8 py-5 flex justify-center gap-2">
        <!-- BOTÃO 1: Visualizar -->
        <button onclick="viewFromDB(150); event.stopPropagation();">
            <i data-lucide="eye" class="w-4 h-4"></i>
        </button>
        
        <!-- BOTÃO 2: Editar -->
        <button onclick="loadFromDB(150); event.stopPropagation();">
            <i data-lucide="edit-2" class="w-4 h-4"></i>
        </button>
        
        <!-- BOTÃO 3: Duplicar -->
        <button onclick="duplicateFromDB(150); event.stopPropagation();">
            <i data-lucide="copy" class="w-4 h-4"></i>
        </button>
        
        <!-- BOTÃO 4: Apagar -->
        <button onclick="deleteFromDB(150); event.stopPropagation();">
            <i data-lucide="trash-2" class="w-4 h-4"></i>
        </button>
    </td>
</tr>
<!-- TAMBÉM TEM ESTE: -->
<tr onclick="loadFromDB(0)">  <!-- ❌ PROBLEMA: passa 0, não 150! -->
```

---

## O Problema em 3 Passos

### 1️⃣ RENDERIZAÇÃO - Como a função cria as linhas

```javascript
function renderDatabase(data) {  // data = filtered array [TRA759, TRA759_R2]
    const body = document.getElementById('db-body');
    body.innerHTML = '';
    data.forEach((rec, idx) => {  // ⬅️ idx vai de 0 a N-1 do array passado
        // idx = 0  quando rec = TRA759     (no array data)
        // idx = 1  quando rec = TRA759_R2  (no array data)
        
        const tr = document.createElement('tr');
        
        // ✅ BOTÕES: Calculam índice correto em dbRecords
        tr.innerHTML = `
            <button onclick="viewFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)});">
                                           ↑
                Para TRA759: dbRecords.findIndex(...) = 150 ✅
        `;
        
        // ❌ CLIQUE NA LINHA: Usa idx do array data
        tr.onclick = () => loadFromDB(idx);
                                     ↑
            Para TRA759: idx = 0  ❌
            Mas precisa: 150  ✅
            
        body.appendChild(tr);
    });
}
```

---

### 2️⃣ FUNÇÃO loadFromDB - O que espera

```javascript
function loadFromDB(idx) {
    const r = dbRecords[idx];  // ⬅️ Trata 'idx' como índice de dbRecords
    
    // Se idx = 0:    r = dbRecords[0] = TRA100 (ERRADO!)
    // Se idx = 150:  r = dbRecords[150] = TRA759 (CORRETO!)
}
```

---

### 3️⃣ CHAMADAS - A Inconsistência

| Quem chama | O que passa | Índice esperado | Índice recebido | Resultado |
|-----------|------------|----------------|-----------------|-----------|
| Botão "Visualizar" | `viewFromDB(150)` | 150 em dbRecords | 150 em dbRecords | ✅ FUNCIONA |
| Botão "Editar" | `loadFromDB(150)` | 150 em dbRecords | 150 em dbRecords | ✅ FUNCIONA |
| Botão "Duplicar" | `duplicateFromDB(150)` | 150 em dbRecords | 150 em dbRecords | ✅ FUNCIONA |
| Botão "Apagar" | `deleteFromDB(150)` | 150 em dbRecords | 150 em dbRecords | ✅ FUNCIONA |
| **Clique na Linha** | `loadFromDB(0)` | 150 em dbRecords | **0 em dbRecords** | ❌ **FALHA** |

---

## Simulação do Bug

### Estado de Dados

```javascript
dbRecords = [
    { codigo: 'TRA100', titulo: 'Projeto 100' },    // índice 0
    { codigo: 'TRA101', titulo: 'Projeto 101' },    // índice 1
    // ...
    { codigo: 'TRA526_R3', titulo: 'Proj 526-R3' },  // índice 50 ← AQUI
    { codigo: 'TRA527', titulo: 'Projeto 527' },    // índice 51
    // ...
    { codigo: 'TRA759', titulo: 'Projeto 759' },    // índice 150 ← AQUI
    { codigo: 'TRA759_R2', titulo: 'Proj 759-R2' }, // índice 151 ← AQUI
    { codigo: 'TRB548', titulo: 'Projeto B548' },   // índice 152 ← AQUI
    // ... total 500 registos
];
```

### Ação do Utilizador: Pesquisar "TRA75"

```javascript
// filterDatabase() executa:
const filtered = dbRecords.filter(rec => 
    rec.codigo.includes('TRA75')
);

// Resultado:
filtered = [
    { codigo: 'TRA759', titulo: 'Projeto 759' },    // índice 0 em filtered
    { codigo: 'TRA759_R2', titulo: 'Proj 759-R2' }, // índice 1 em filtered
];

renderDatabase(filtered);
```

### Ação do Utilizador: Clica em "TRA759" (a LINHA, não o botão)

```javascript
// A linha tem: tr.onclick = () => loadFromDB(idx);
// Quando idx = 0 (posição de TRA759 no array filtered)

loadFromDB(0);

// Função executa:
const r = dbRecords[0];  // ❌ ERRADO!
console.log(r.codigo);   // "TRA100" (nem era para aparecer!)
```

### Ação do Utilizador: Clica em "TRA759_R2" (a LINHA)

```javascript
// A linha tem: tr.onclick = () => loadFromDB(idx);
// Quando idx = 1 (posição de TRA759_R2 no array filtered)

loadFromDB(1);

// Função executa:
const r = dbRecords[1];  // ❌ ERRADO!
console.log(r.codigo);   // "TRA101" (nem era para aparecer!)
```

### Ação do Utilizador: Clica no Botão "Editar" em "TRA759"

```javascript
// O inline onclick do botão tem:
// onclick="loadFromDB(${dbRecords.findIndex(r => r.codigo === 'TRA759')})"
// dbRecords.findIndex(...) calcula = 150

loadFromDB(150);

// Função executa:
const r = dbRecords[150];  // ✅ CORRETO!
console.log(r.codigo);     // "TRA759" (como esperado)
```

---

## Por que o Relatório do Utilizador Faz Sentido

> "Se clica em TRA759, mostra TRA526_R3. Se clica em TRA759_R2, mostra TRB548."

Se em `dbRecords`:
- `dbRecords[50]` = TRA526_R3

Quando o utilizador clica em TRA759 (linha):
1. **Idx passado:** 0 (posição no array filtrado)
2. **Índice usado:** dbRecords[0]
3. **Registo carregado:** Qualquer coisa que esteja em dbRecords[0]
4. **Coincidência:** Se dbRecords[0..49] = [TRA100..TRA149], então **não é TRA526_R3**

Mas se o índice coincidisse com a posição onde **TRA526_R3 realmente está** (~índice 50), poderia acontecer de clicar em TRA759 e mostrar TRA526_R3!

---

## Fluxo de Execução - Diagrama

```
┌──────────────────────────────────────┐
│ HTML: <input id="db-search">          │
│  Input: "TRA75"                      │
└────────────┬─────────────────────────┘
             │
             ▼ (user types)
┌──────────────────────────────────────┐
│ filterDatabase()                      │
│ filtered = dbRecords.filter(...)     │
│ filtered = [TRA759, TRA759_R2]       │
└────────────┬─────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│ renderDatabase(filtered)              │
│  idx=0: TRA759                       │
│  idx=1: TRA759_R2                    │
└────────────┬─────────────────────────┘
             │
     ┌───────┴───────┐
     │               │
     ▼ (user clicks) ▼ (user clicks)
  LINE CLICK      BUTTON CLICK
  tr.onclick          onclick=""
  loadFromDB(0)   loadFromDB(150)
     │               │
     ▼               ▼
  CARREGA        CARREGA
  dbRecords[0]   dbRecords[150]
  ❌ ERRADO      ✅ CORRETO
  TRA100         TRA759
```

---

## Verificação

### Como confirmar que o bug existe

```javascript
// Console do navegador durante teste:

// 1. Pesquisar "TRA75"
// 2. Abrir console (F12)
// 3. Clicar em TRA759 (na linha)
// 4. Verificar que carregou outro código

// Se ver:
// projectData.code = "TRA100"  (ou outro)
// ❌ = Confirma o bug

// Se clicasse no botão:
// projectData.code = "TRA759"
// ✅ = Funcionaria corretamente
```

---

## Fix Resumido

**Mudar esta linha:**
```javascript
tr.onclick = () => loadFromDB(idx);
```

**Para esta linha:**
```javascript
tr.onclick = () => loadFromDB(dbRecords.findIndex(r => r.codigo === rec.codigo));
```

**Resultado:**
- ✅ Clique na linha funciona
- ✅ Clique no botão funciona
- ✅ Tudo consistente

