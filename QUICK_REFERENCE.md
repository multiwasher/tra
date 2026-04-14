# 🎯 QUICK REFERENCE - Bug TRA759

## Problema
```
Pesquisa \"TRA75\" → TRA759 ← Clic → Carrega TRA526_R3 ❌
```

## Causa
**Linha 504 usa `idx` do array filtrado em vez do índice verdadeiro em `dbRecords`**

## Fix
```diff
- tr.onclick = () => loadFromDB(idx);
+ tr.onclick = () => loadFromDB(dbRecords.findIndex(r => r.codigo === rec.codigo));
```

## Impacto
- 1 linha
- 0 testes quebrados
- Alto risco reduzido

## Documentação Completa
1. `00_COMECE_AQUI.md` ← Sumário executivo
2. `RESPOSTA_DIRETA_6_PONTOS.md` ← Respostas aos 6 pontos
3. `ANALISE_BUG_DETALHADA.md` ← Análise técnica
4. `ANALISE_VISUAL.md` ← Diagramas e simulações
5. `SOLUCAO_FIX.md` ← Guia de implementação
6. `ANALISE_PROBLEMA.md` ← Análise anterior (referência)

---

## O Bug Explicado em 10 Linhas

```javascript
// 1. Pesquisa filtra dados
const filtered = dbRecords.filter(rec => rec.codigo.includes('TRA75'));
// filtered = [TRA759, TRA759_R2]  ← array pequeno

// 2. Renderiza com índices do array pequeno
data.forEach((rec, idx) => {  // idx = 0, 1
    // ❌ ERRADO: passa 0 ou 1
    tr.onclick = () => loadFromDB(idx);
});

// 3. loadFromDB trata como índice de dbRecords
const r = dbRecords[0];  // ❌ Carrega dbRecords[0], não TRA759!

// ✅ CORRETO: passar 150 (índice verdadeiro)
const r = dbRecords[150];  // Carrega TRA759
```

---

## Estado Atual vs. Depois

### Comportamento ANTES ❌
```
┌─────────────────────────────────────┐
│ Pesquisa: \"TRA75\"                   │
└────────────┬────────────────────────┘
             │
      ┌──────┴──────┐
      │             │
      ▼             ▼
  Clica Linha   Clica Botão
  ❌ Carrega    ✅ Carrega
   Errado       Certo
```

### Comportamento DEPOIS ✅
```
┌─────────────────────────────────────┐
│ Pesquisa: \"TRA75\"                   │
└────────────┬────────────────────────┘
             │
      ┌──────┴──────┐
      │             │
      ▼             ▼
  Clica Linha   Clica Botão
  ✅ Carrega    ✅ Carrega
   Certo        Certo
```

---

## Ficheiros Modificados
- `index.html` - Linha 504 (1 mudança)

## Ficheiros Criados (Documentação)
- `00_COMECE_AQUI.md`
- `RESPOSTA_DIRETA_6_PONTOS.md`
- `ANALISE_BUG_DETALHADA.md`
- `ANALISE_VISUAL.md`
- `SOLUCAO_FIX.md`
- `QUICK_REFERENCE.md` (este ficheiro)

## Teste Rápido
```bash
# 1. Pesquisar \"TRA75\"
# 2. Clicar em TRA759 (linha)
# Expected: Abre o projeto TRA759
# Resultado ANTES: ❌ Abre outro projeto
# Resultado DEPOIS: ✅ Abre TRA759
```

---

## Respostas aos 6 Pontos

### 1. Índices de TRA759, TRA759_R2, TRA526_R3?
👉 Dinâmicos, obtidos via `dbRecords.findIndex()`

### 2. Registos duplicados?
👉 Não (em código). Verificar dados do Sheets.

### 3. Causa real?
👉 Inconsistência: `idx` vs `findIndex()`

### 4. É tr.onclick ou findIndex?
👉 É `tr.onclick` (linha 504)

### 5. Trechos?
👉 Linhas 486-512, especialmente 504

### 6. Solução?
👉 Usar `findIndex()` em vez de `idx`

---

Leia `00_COMECE_AQUI.md` para análise completa.

