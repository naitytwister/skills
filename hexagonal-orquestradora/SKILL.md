---
name: hexagonal-orquestradora
description: Use quando o usuário solicitar criar um projeto Python novo em Arquitetura Hexagonal ou modernizar/migrar um sistema legado existente via Strangler Fig. Coordena de ponta a ponta as sub-skills de diagnóstico, scaffold, testes de caracterização, migração incremental e guarda arquitetural.
---

# 1. Objetivo
Orquestrar de forma autônoma e determinística a criação de novos projetos ou a migração incremental de legados para Arquitetura Hexagonal em Python, garantindo integridade de código, portões de qualidade rígidos e isolamento git sem supervisão técnica do usuário.

---

# 2. Pré-condições
Antes de executar qualquer ação, o agente deve verificar obrigatoriamente:
1. **Ambiente Git inicializado:** O diretório de trabalho deve ser um repositório git válido.
2. **Árvore de trabalho limpa:** Não deve haver arquivos modificados ou não rastreados pendentes de commit na branch atual.
3. **Ambiente Python 3.10+ disponível:** Verificado via comando de versão.
4. **Permissões de escrita:** Acesso total para criar e editar pastas no diretório do projeto.

---

# 3. Procedimento Passo a Passo

### Passo 1: Inspeção de Estado e Classificação da Missão
Determinar se a missão é **Greenfield (Projeto Novo)** ou **Migração Strangler Fig (Projeto Legado)**.
- Regra de Decisão:
  - Se o diretório atual contiver apenas `.git` ou estiver vazio (ou tiver apenas README/licença) -> Classificar como `MODO_GREENFIELD`.
  - Se o diretório contiver arquivos de código de aplicação (`.py`, `requirements.txt`, `app.py`, `src/`, etc.) -> Classificar como `MODO_STRANGLER`.

*Comando de Verificação:*
```bash
git status --porcelain
```
*Resultado Esperado:* Saída vazia (código de saída 0). Se houver arquivos pendentes, o agente deve abortar imediatamente informando que a árvore de trabalho não está limpa.

### Passo 2: Ponto de Restauração Global e Congelamento de Segurança
Criar uma tag de segurança no git para garantir um ponto de retorno imutável antes de qualquer alteração.

*Comando de Execução:*
```bash
git tag base-seguranca-$(date +"%Y%m%d%H%M%S")
```
*Comando de Verificação:*
```bash
git tag -l "base-seguranca-*" | tail -n 1
```
*Resultado Esperado:* Exibição da tag recém-criada (código de saída 0).

### Passo 3: Execução do Ramo Greenfield (Se MODO_GREENFIELD)
1. Acionar a sub-skill hexagonal-scaffold-projeto-novo passando o nome do projeto e domínio inicial inferidos do prompt.
2. Acionar a sub-skill hexagonal-guarda-arquitetura para validar o scaffold gerado.
3. Se os portões passarem, consolidar o commit inicial na branch main.
4. Pular para o Passo 6.

*Comando de Verificação:*
```bash
bash scripts/run_gates.sh <project_name>
```
*Resultado Esperado:* Mensagem final === TODOS OS PORTÕES DE QUALIDADE FORAM APROVADOS COM SUCESSO === (código de saída 0).

### Passo 4: Diagnóstico e Planejamento da Migração (Se MODO_STRANGLER)
1. Acionar a sub-skill hexagonal-diagnostico-legado para analisar o monolito legado.
2. Inspecionar o artefato gerado migration_plan.json.
3. Ordenar a fila de casos de uso com base na regra de menor dependência externa primeiro.
4. Se o plano estiver vazio ou não contiver casos de uso identificáveis, registrar o motivo em decisoes_pendentes.md e interromper.

*Comando de Verificação:*
```bash
test -f migration_plan.json && jq '.use_cases | length' migration_plan.json
```
*Resultado Esperado:* Número inteiro maior que 0 (código de saída 0).

### Passo 5: Loop de Migração Incremental por Caso de Uso (Se MODO_STRANGLER)
Para cada caso de uso com status "pending" no migration_plan.json:
1. Atualizar o status do caso no migration_plan.json para "in_progress".
2. Criar e trocar para branch isolada: `git checkout -b migracao/<nome_do_caso>`
3. Acionar a sub-skill hexagonal-testes-caracterizacao para congelar o comportamento do caso de uso legado em tests/characterization/.
4. Acionar a sub-skill hexagonal-migrar-caso-de-uso para implementar domínio rico, portas, serviços, adaptadores e roteamento Strangler Fig.
5. Acionar a sub-skill hexagonal-guarda-arquitetura.
6. Se aprovado: commit, merge --ff-only na main, branch -d, status completed. Se reprovado: até 3 ciclos de correção, senão reversão segura.

*Comando de Verificação:*
```bash
git branch --show-current
```
*Resultado Esperado:* Retorno deve ser main após merge ou reversão (código de saída 0).

### Passo 6: Geração do Relatório Final para Leigo
Acionar a sub-skill hexagonal-relatorio-final para consolidar os logs de reports/, métricas do migration_plan.json e registrar o status final em linguagem clara e acessível para o usuário que não programa.

*Comando de Verificação:*
```bash
test -f reports/relatorio_final.md
```
*Resultado Esperado:* Arquivo reports/relatorio_final.md existente e com tamanho superior a zero bytes (código de saída 0).

---

# 4. Regras Invioláveis Aplicáveis
- O domínio (domain/) e a aplicação (application/) são protegidos contra importações de infraestrutura e frameworks web.
- Nenhuma alteração pode ser realizada diretamente na branch main sem passar pelos portões em branch isolada migracao/<caso>.
- O agente está estritamente proibido de executar git reset --hard ou git clean -fd na branch principal para evitar perda de dados legados não rastreados.
- Testes de caracterização existentes são imutáveis durante a fase de migração da regra de negócio.

---

# 5. Templates de Artefatos de Orquestração
## Template migration_plan.json [FORA DAS FONTES]
```json
{
  "project_name": "sistema_legado",
  "created_at": "2026-10-02T22:00:00Z",
  "current_use_case": "alocar_pedido",
  "use_cases": [
    {
      "name": "alocar_pedido",
      "status": "pending",
      "legacy_endpoints": ["POST /orders/allocate"],
      "characterization_tests": ["tests/characterization/test_allocate.py"],
      "attempts": 0
    }
  ]
}
```

---

# 6. Critérios de Parada e Reversão
1. **Falha Persistente nos Portões:** após 3 tentativas, run_gates.sh reprovado → `git checkout main && git branch -D migracao/<caso>`.
2. **Divergência Crítica de Negócio:** comportamento do legado inconsistente (decisoes_pendentes.md).
3. **Violação da Trava de Banco de Dados:** DATABASE_URL produtivo durante testes.
4. **Alteração Não Autorizada de Arquivos de Guarda.**

---

# 7. Saída Final
Em reports/relatorio_final.md: tipo de operação, casos concluídos/pendentes, qualidade técnica, riscos e próximo passo.
