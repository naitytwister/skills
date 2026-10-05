---
name: hexagonal-diagnostico-legado
description: Use quando for necessário analisar uma base de código legada (Big Ball of Mud) para planejar a migração para Arquitetura Hexagonal via Strangler Fig. Identifica pontos de entrada, acoplamentos de persistência, efeitos colaterais e fatia os casos de uso ordenados por dependência no migration_plan.json.
---
# 1. Objetivo
Analisar deterministicamente o sistema legado, mapear acoplamentos de infraestrutura e gerar o plano formal de migração incremental (`migration_plan.json`) ordenado por menor dependência externa.
---
# 2. Pré-condições
O agente deve verificar obrigatoriamente antes de iniciar:
1. **Repositório Git Válido e Limpo:** Sem alterações não commitadas na branch principal.
2. **Código Legado Presente:** Existência de arquivos de código da aplicação anterior no diretório raiz ou subdiretórios.
3. **Branch Atual é a Principal:** O diagnóstico deve rodar na branch `main` (ou `master`).
4. **Estabelecimento de Guarda:** A tag imutável `$GUARD_REF` deve ser criada e exportada no ambiente.
*Comando de Verificação:*
```bash
git diff --quiet && git diff --cached --quiet && test -z "$(git status --porcelain)"
```
*Resultado Esperado:* Código de saída `0` (nenhuma modificação pendente).
---
# 3. Procedimento Passo a Passo
### Passo 1: Estabelecimento da Referência de Guarda Global ($GUARD_REF)
Criar uma tag de segurança imutável no git que servirá como base para as verificações estruturais de guarda.
*Comando de Execução:*
```bash
GUARD_TAG="base-seguranca-$(date +"%Y%m%d%H%M%S")"
git tag "$GUARD_TAG"
export GUARD_REF="$GUARD_TAG"
```
*Comando de Verificação:*
```bash
git rev-parse "$GUARD_REF" > /dev/null 2>&1
```
*Resultado Esperado:* Código de saída `0`.
---
### Passo 2: Varredura de Pontos de Entrada e Rotas do Legado
Identificar todos os controladores, rotas HTTP (FastAPI, Flask, Django) ou comandos CLI do código legado através de análise estática.
*Comando de Execução:*
```bash
mkdir -p reports
python3 scripts/scan_legacy_entrypoints.py > reports/legacy_entrypoints.json
```
*Comando de Verificação:*
```bash
test -s reports/legacy_entrypoints.json
```
*Resultado Esperado:* Código de saída `0` (arquivo gerado e não vazio).
---
### Passo 3: Análise de Acoplamento de Banco e Efeitos Colaterais
Mapear chamadas diretas de banco de dados (`session.commit`, SQL bruto, chamadas ORM ativas) e serviços externos (e-mail, APIs de terceiros, filas) associados a cada rota.
*Comando de Execução:*
```bash
python3 scripts/analyze_legacy_coupling.py reports/legacy_entrypoints.json > reports/legacy_dependencies.json
```
*Comando de Verificação:*
```bash
test -s reports/legacy_dependencies.json
```
*Resultado Esperado:* Código de saída `0`.
---
### Passo 4: Fatiamento e Ordenação dos Casos de Uso (Strangler Fig)
Estruturar os casos de uso no arquivo `migration_plan.json`, ordenando-os com base na regra de menor acoplamento externo primeiro.
Todos os casos iniciam estritamente com status `"pending"`. Os únicos status permitidos pelo contrato são: `pending`, `in_progress`, `completed`, `failed`, `blocked`.
*Comando de Execução:*
```bash
python3 scripts/generate_migration_plan.py reports/legacy_dependencies.json > migration_plan.json
```
*Comando de Verificação:*
```bash
python3 -c "
import json
with open('migration_plan.json') as f:
    data = json.load(f)
assert data['guard_ref'] != ''
assert len(data['use_cases']) > 0
valid_statuses = {'pending', 'in_progress', 'completed', 'failed', 'blocked'}
for uc in data['use_cases']:
    assert uc['status'] in valid_statuses, f'Status inválido: {uc[\"status\"]}'
"
```
*Resultado Esperado:* Código de saída `0`.
---
### Passo 5: Inicialização do Quadro de Decisões Pendentes
Criar o arquivo `decisoes_pendentes.md` para registro de ambiguidades técnicas ou de regras de negócio encontradas no legado.
*Comando de Execução:*
```bash
cat << 'EOF' > decisoes_pendentes.md
# Decisões Pendentes e Bloqueios de Negócio
Este arquivo registra comportamentos ambíguos ou inconsistências encontradas no código legado.
EOF
```
*Comando de Verificação:*
```bash
test -f decisoes_pendentes.md
```
*Resultado Esperado:* Código de saída `0`.
---
### Passo 6: Consolidação do Diagnóstico no Git
Commitar os arquivos de planejamento na branch `main` para compor o estado de controle do projeto.
*Comando de Execução:*
```bash
git add migration_plan.json decisoes_pendentes.md reports/
git commit -m "chore(hex): diagnostico do legado e definicao do migration_plan.json"
```
*Comando de Verificação:*
```bash
git diff --quiet HEAD~1 HEAD -- migration_plan.json
```
*Resultado Esperado:* Código de saída `1` (comprova que o arquivo foi comitado no commit mais recente) e `git status --porcelain` retornando vazio (código `0`).
---
# 4. Regras Invioláveis Aplicáveis
* **Fonte (Architecture Patterns with Python - Epílogo / Martin Fowler):**
1. **Migração Incremental Vertical:** É proibido reescrever o sistema de forma monolítica ("Big Bang"). A migração ocorre estritamente caso de uso a caso de uso.
2. **Priorização por Menor Dependência:** Casos de uso de leitura pura ou com dependências autocontidas devem ser migrados antes de casos com transações distribuídas e múltiplos efeitos colaterais.
3. **Contrato de Status Rigoroso:** O status de cada caso de uso em `migration_plan.json` só pode ser `pending`, `in_progress`, `completed`, `failed` ou `blocked`.
4. **Preservação Absoluta do Legado:** Durante a fase de diagnóstico, nenhuma linha do código legado pode ser modificada ou removida.
5. **Verificação Estrita por Código de Saída:** Todas as asserções e validações devem ser feitas com base no código de retorno do processo (`0` para sucesso), sendo proibido o parsing de texto.
---
# 5. Templates de Código Prontos (Python e JSON)
### 5.1 Script de Varredura de Rotas: `scripts/scan_legacy_entrypoints.py` [FORA DAS FONTES]
```python
#!/usr/bin/env python3
"""Localiza rotas HTTP em arquivos Python do projeto legado via AST."""
import ast
import json
from pathlib import Path
import sys
def find_routes() -> list[dict[str, str]]:
  endpoints = []
  for path in Path(".").rglob("*.py"):
    if (
        "tests" in path.parts
        or "scripts" in path.parts
        or ".venv" in path.parts
    ):
      continue
    try:
      tree = ast.parse(path.read_text(encoding="utf-8"), filename=str(path))
    except Exception:
      continue
    for node in ast.walk(tree):
      if isinstance(node, ast.FunctionDef):
        for dec in node.decorator_list:
          if isinstance(dec, ast.Call) and isinstance(dec.func, ast.Attribute):
            method = dec.func.attr.upper()
            if method in {"GET", "POST", "PUT", "DELETE", "PATCH"}:
              route_path = "/"
              if dec.args and isinstance(dec.args[0], ast.Constant):
                route_path = str(dec.args[0].value)
              endpoints.append({
                  "file": str(path),
                  "line": node.lineno,
                  "function": node.name,
                  "method": method,
                  "path": route_path,
                  "suggested_name": f"{method.lower()}_{node.name}",
              })
  return endpoints
if __name__ == "__main__":
  routes = find_routes()
  print(json.dumps({"endpoints": routes}, indent=2, ensure_ascii=False))
  sys.exit(0)
```
### 5.2 Script de Análise de Acoplamento: `scripts/analyze_legacy_coupling.py` [FORA DAS FONTES]
```python
#!/usr/bin/env python3
"""Mapeia tabelas de banco e efeitos colaterais associados a cada endpoint legado."""
import ast
import json
from pathlib import Path
import sys
def analyze_file_coupling(file_path: str, func_name: str) -> dict[str, list[str]]:
  tree = ast.parse(Path(file_path).read_text(encoding="utf-8"))
  tables = set()
  side_effects = set()
  for node in ast.walk(tree):
    if isinstance(node, ast.FunctionDef) and node.name == func_name:
      for subnode in ast.walk(node):
        if isinstance(subnode, ast.Call):
          if isinstance(subnode.func, ast.Attribute):
            attr = subnode.func.attr
            if attr in {"commit", "execute", "query", "filter_by"}:
              tables.add("db_transaction")
            if "email" in attr or "mail" in attr or "send" in attr:
              side_effects.add("notifications")
  return {"db_tables": list(tables), "side_effects": list(side_effects)}
def main() -> int:
  if len(sys.argv) < 2:
    return 1
  with open(sys.argv[1], encoding="utf-8") as f:
    data = json.load(f)
  for ep in data.get("endpoints", []):
    coupling = analyze_file_coupling(ep["file"], ep["function"])
    ep["db_tables"] = coupling["db_tables"]
    ep["side_effects"] = coupling["side_effects"]
  data["project_name"] = Path(".").resolve().name.replace("-", "_")
  print(json.dumps(data, indent=2, ensure_ascii=False))
  return 0
if __name__ == "__main__":
  sys.exit(main())
```
### 5.3 Script de Geração do Plano: `scripts/generate_migration_plan.py` [FORA DAS FONTES]
```python
#!/usr/bin/env python3
"""Gera o migration_plan.json estruturado conforme o contrato comum."""
import json
import os
import sys
def main() -> int:
  if len(sys.argv) < 2:
    return 1
  dep_file = sys.argv[1]
  guard_ref = os.environ.get("GUARD_REF", "base-seguranca")
  with open(dep_file, encoding="utf-8") as f:
    legacy_data = json.load(f)
  # Ordena: menos acoplamentos de banco e efeitos colaterais primeiro
  sorted_endpoints = sorted(
      legacy_data.get("endpoints", []),
      key=lambda x: (len(x.get("db_tables", [])), len(x.get("side_effects", []))),
  )
  use_cases = []
  for ep in sorted_endpoints:
    use_cases.append({
        "name": ep["suggested_name"],
        "status": "pending",
        "legacy_endpoints": [f"{ep['method']} {ep['path']}"],
        "target_service": f"src/{legacy_data.get('project_name', 'core_app')}/application/services.py:{ep['suggested_name']}",
        "characterization_tests": [
            f"tests/characterization/test_{ep['suggested_name']}.py"
        ],
        "dependencies": ep.get("db_tables", []),
        "side_effects": ep.get("side_effects", []),
        "attempts": 0,
    })
  plan = {
      "project_name": legacy_data.get("project_name", "core_app"),
      "guard_ref": guard_ref,
      "total_use_cases": len(use_cases),
      "use_cases": use_cases,
  }
  print(json.dumps(plan, indent=2, ensure_ascii=False))
  return 0
if __name__ == "__main__":
  sys.exit(main())
```
---
# 6. Erros Comuns e Como Corrigir
* **Erro:** Começar a migração por fluxos complexos com múltiplas gravações no banco.
* *Correção:* Seguir estritamente a ordenação de `scripts/generate_migration_plan.py`, priorizando operações de leitura ou com menor dependência de banco.
* **Erro:** Usar status fora do contrato no `migration_plan.json` (ex.: `"done"`, `"waiting"`, `"error"`).
* *Correção:* Aplicar asserção de schema que garanta que os únicos status permitidos sejam `pending`, `in_progress`, `completed`, `failed` e `blocked`.
* **Erro:** Modificar ou tentar refatorar arquivos legados durante o diagnóstico.
* *Correção:* Manter o diagnóstico como operação estritamente passiva (somente leitura do código legado e gravação de artefatos de controle).
---
# 7. Critérios de Parada
O agente deve parar a execução imediatamente se:
1. **Erros de Sintaxe Impeditivos:** O código legado possuir erros de compilação ou sintaxe que impeçam o parsing pelo AST do Python.
2. **Nenhum Ponto de Entrada Mapeável:** Não forem localizados endpoints HTTP ou comandos CLI executáveis.
3. **Impossibilidade de Git Limpo:** O repositório git apresentar alterações pendentes de commit na branch `main`.
---
# 8. Saída Final (Modelo de Relatório para Leigo)
Ao concluir o diagnóstico, a skill gera e exibe o resumo em `reports/diagnostico_legado.md`:
```markdown
# Diagnóstico do Sistema Legado e Plano de Migração
## 1. O Que Foi Realizado
- Foi realizada a varredura completa do sistema legado para identificar suas funcionalidades e acoplamentos.
- Foram detectados **[X]** pontos de entrada e rotas operacionais.
- Foi gerado o plano formal de migração incremental (`migration_plan.json`) dividido em **[X]** casos de uso independentes.
- Foi criada a marcação de segurança no Git (`[GUARD_REF]`) para garantir retorno seguro em caso de falha.
## 2. O Que Foi Testado e Validado
- A estrutura do plano de migração foi validada contra o contrato oficial com código de saída 0.
- Todos os casos de uso foram configurados com status inicial `pending`.
- O código original não sofreu nenhuma alteração estrutural.
## 3. Estado Atual dos Portões
- Varredura de Controladores Legados: **Aprovado**
- Análise de Dependências de Banco de Dados: **Aprovado**
- Validação Estrutural do Plano (`migration_plan.json`): **Aprovado**
## 4. Riscos Restantes
- O sistema legado ainda está operando 100% no modelo antigo.
- A migração ainda não começou; os riscos de regressão serão mitigados na próxima etapa através dos testes de caracterização.
## 5. Próximo Passo Recomendado
- Iniciar a captura de testes de caracterização para o primeiro caso da fila: `[NOME_DO_PRIMEIRO_CASO]`.
```
