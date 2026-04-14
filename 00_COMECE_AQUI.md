# 📋 RESUMO EXECUTIVO - Bug TRA759

## O Problema Reportado
```
Utilizador: Pesquiso \"TRA75\" e vejo TRA759 e TRA759_R2.
Se clico em TRA759, mostra TRA526_R3.
Se clico em TRA759_R2, mostra TRB548.
❌ Registos carregam errados!
```

---

## A Causa Raiz

### ❌ INCONSISTÊNCIA: Dois Métodos Diferentes de Passar Índices

```
┌─────────────────────────────────────────────────────────────┐
│                     renderDatabase(data)                    │
│                                                              │
│  data.forEach((rec, idx) => {                              │
│                                                              │
│    // ✅ CORRETO: Botões usam findIndex                    │
│    <button onclick=\"loadFromDB(                            │
│        dbRecords.findIndex(r => r.codigo === rec.codigo)  │
│    );\">Editar</button>                                    │
│        ↑                                                    │
│        Procura em dbRecords completo                      │
│        Resultado para TRA759: índice 150 ✅                │
│                                                              │
│    // ❌ ERRADO: Linha usa idx                             │
│    tr.onclick = () => loadFromDB(idx);                    │
│                      ↑                                      │
│                      Usa índice relativo a 'data'          │
│                      Resultado para TRA759: índice 0 ❌    │
│  });                                                         │
│}                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Ficheiros de Análise Criados

| Ficheiro | Conteúdo | Para Quem |
|----------|----------|-----------|
| **RESPOSTA_DIRETA_6_PONTOS.md** | Respostas diretas aos 6 pontos solicitados | 📌 COMECE AQUI |
| **ANALISE_BUG_DETALHADA.md** | Análise técnica profunda com exemplos | Desenvolvedores |
| **ANALISE_VISUAL.md** | Diagramas, simulações, fluxogramas | Aprendizagem visual |
| **SOLUCAO_FIX.md** | Guia passo-a-passo para implementar a fix | Implementação |

---

## A Fix em 30 Segundos

### Localização
- **Arquivo:** `index.html`
- **Linha:** 504

### Antes ❌
```javascript
tr.onclick = () => loadFromDB(idx);
```

### Depois ✅
```javascript
tr.onclick = () => loadFromDB(dbRecords.findIndex(r => r.codigo === rec.codigo));
```

### Efeito
Cliques na linha passam a funcionar como os botões: carregam o registo correto mesmo com filtro ativo.

---

## Validação Rápida

### Teste 1: Com Filtro
```
1. Pesquisar "TRA75"
2. Clicar em TRA759 (na linha)
3. Esperado: Carrega TRA759
   ANTES (❌): Carrega outro código
   DEPOIS (✅): Carrega TRA759
```

### Teste 2: Sem Filtro
```
1. Limpar pesquisa
2. Clicar em TRA759
3. Esperado: Carrega TRA759 (funcionava antes, continua)
```

### Teste 3: Botões
```
1. Clicar em botão "Editar"
2. Esperado: Carrega correto (funcionava antes, continua)
```

---

## Análise Técnica Rápida

### Por que acontece?

```
Estado: dbRecords tem 500 registos [TRA100...TRB999]

Pesquisa "TRA75":
  filtered = [TRA759, TRA759_R2]  ← array pequeno
  
Índices em 'filtered': 0, 1
Índices em 'dbRecords': 150, 151 (verdadeiros)

Clica TRA759 (linha):
  Passa: idx = 0        (relativo a filtered)
  Recebe: dbRecords[0]  (interpretado como dbRecords)
  Carrega: TRA100       (ERRADO!)

Clica TRA759 (botão):
  Passa: findIndex = 150 (procura em dbRecords)
  Recebe: dbRecords[150] (correto)
  Carrega: TRA759       (CORRETO!)
```

---

## Padrão de Código

O código segue um padrão mas tem inconsistência:

```javascript
// ✅ Padrão CORRETO (para botões):
function renderDatabase(data) {
    data.forEach((rec, idx) => {
        tr.innerHTML = `
            <button onclick="loadFromDB(${dbRecords.findIndex(r => r.codigo === rec.codigo)});">
        `;
    });
}

// ❌ Padrão ERRADO (para linha):
function renderDatabase(data) {
    data.forEach((rec, idx) => {
        tr.onclick = () => loadFromDB(idx);  // ← Usa idx do forEach
    });
}

// ✨ Deveria ser (consistente):
function renderDatabase(data) {
    data.forEach((rec, idx) => {
        tr.onclick = () => loadFromDB(dbRecords.findIndex(r => r.codigo === rec.codigo));
    });
}
```

---

## Checklist de Fixes

- [ ] Ler `RESPOSTA_DIRETA_6_PONTOS.md` para entender
- [ ] Ler `ANALISE_VISUAL.md` para visualizar o bug
- [ ] Ler `SOLUCAO_FIX.md` para implementar
- [ ] Fazer backup do `index.html`
- [ ] Editar linha 504 conforme indicado
- [ ] Testar com filtro ativo
- [ ] Testar sem filtro
- [ ] Testar cliques em botões
- [ ] Commit das alterações

---

## Impacto da Fix

| Métrica | Antes | Depois |
|---------|-------|--------|
| Performance | OK | Mesmo (uma chamada findIndex por render) |
| Compatibilidade | N/A | 100% (HTML/JS padrão) |
| Risco | N/A | Muito baixo (1 linha) |
| Efeito colateral | Sim ❌ | Não ✅ |
| Testes necessários | Sim | Sim (validação básica) |

---

## Próximos Passos

1. **Compreensão:** Ler os documentos na ordem:
   1. RESPOSTA_DIRETA_6_PONTOS.md
   2. ANALISE_VISUAL.md
   3. SOLUCAO_FIX.md

2. **Implementação:** Aplicar a mudança de 1 linha

3. **Validação:** Testar conforme checklist acima

4. **Documentação:** Comentar no código (opcional):
   ```javascript
   // NOTE: O índice vem de dbRecords.findIndex para funcionar
   // corretamente quando renderDatabase() recebe um array filtrado
   tr.onclick = () => loadFromDB(dbRecords.findIndex(r => r.codigo === rec.codigo));
   ```

---

**Documentação criada em:** 2026-04-14  
**Status:** Análise Completa ✅  
**Pronto para Implementação:** Sim ✅  

