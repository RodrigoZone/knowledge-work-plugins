# SDR Autônomo — mockup web

Protótipo navegável, 100% web, do sistema descrito em `sdr-autonomo/` (Class Solutions):
empresa → contatos enriquecidos no Apollo → Dynamics 365 → validação oficial do LinkedIn
Sales Navigator → fila de cadência.

Arquivo único, sem build e sem dependências: **`index.html`**. Abra no navegador.

## O que dá para fazer

| Tela | O que demonstra |
|---|---|
| **Painel** | Funil por estado canônico, quotas Apollo/Sales Navigator com os limiares de R6.1/R6.2, e um simulador dos eventos que em produção chegam do Power Automate (R5.2, R5.3, R6.3, R6.4, R4.5). |
| **Prospecção** | O `cli run "Empresa"` passo a passo, cada passo etiquetado com a regra aplicada. Três caminhos: feliz (*Litoral Farma*), ambíguo (*Vale* → R1.1 pede confirmação de domínio), inexistente (*Sertão Mineração* → R1.2 encerra sem gastar crédito). |
| **Pipeline** | Todos os contatos, filtráveis por estado. Clique abre a gaveta com o registro no Dataverse, o gate R5.1 condição por condição e o histórico de decisões. |
| **Validação LinkedIn** | O `cli sync`: lê `li_notatcompany` / `li_crmbadge`, aplica R4.1–R4.4 e roda o gate R5.1. A fila mostra o prognóstico de cada contato antes de rodar. |
| **Cadência** | A fila que o Power Automate consome e, ao lado, quem saiu dela e por qual regra. |
| **Tarefas** | Fila humana gerada por `create_task` (R3.3, R4.1, R6.3 com SLA de 48h). |
| **Regras** | Catálogo R1–R6 renderizado do mesmo objeto que o motor interpreta, diagrama da máquina de estados e um **simulador**: monte os fatos e veja quais regras disparam. |
| **Configuração** | `config.yaml` editável — mudar `max_contacts_per_company` ou `validation_freshness_days` muda o comportamento das próximas execuções. Mais o mapeamento Apollo → Dataverse. |
| **Registro** | Log estruturado com `rule_id` e decisão, filtrável por nível. |

## Fidelidade à especificação

O mockup **não simula** as decisões: reimplementa em JS o que `rules_engine.py` faz.

- `RULES`, `STATE_MACHINE` e `ERROR_HANDLING` são transcrição de `rules/rules.yaml`; `CONFIG` é `config/config.yaml`.
- `evaluate(facts)` percorre as regras com os mesmos operadores (`eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `not_empty`), resolve `${config.*}`, avalia `priority: highest` primeiro e aplica o ramo `else` só quando os fatos referidos existem.
- Toda mudança de estado passa por `canTransition()`; uma transição fora de `state_machine.transitions` é recusada e registrada como erro.
- Nenhuma tela acessa o LinkedIn. Os campos de validação são tratados como somente leitura, escritos pela integração oficial (R6.4).

## Divergências deliberadas

1. **Casamento de cargo com flexão de gênero.** `pipeline.py::_title_matches` usa `target.lower() in title.lower()`, então "Diretora de Tecnologia" **não** casa com o alvo "Diretor de Tecnologia" e é descartada por R2.1. O mockup neutraliza a flexão antes de comparar (`degender`). Sem isso, o sistema descarta em silêncio boa parte das decisoras.
2. Dados fictícios (empresas e pessoas inventadas), sem chamadas de rede — os endpoints Apollo e Dataverse aparecem como rótulo nos passos, não são chamados.
3. O ciclo de 24h do Sales Navigator é disparado sob demanda pelo botão de sync, em vez de agendado.

## Pontos da especificação que valem revisão

- **R2.1, R2.2 e R2.3 estão hardcoded no `pipeline.py`** (`_title_matches`, `candidates[:max]`, `sort`), não passam pelo motor. Isso contraria o `CLAUDE.md` ("nunca hardcode uma regra no Python") e faz com que mudar o YAML não mude o comportamento da seleção. No mockup essas três regras passam pelo motor.
- **`sync_validation` fixa `days_since_validation = 0`** no laço dos contatos já `VALIDADO` ("acabou de validar neste ciclo"). Para quem foi validado em ciclos anteriores isso anula a checagem de frescor de R5.1 — a regra passa a aprovar validações de qualquer idade. O mockup calcula a idade real a partir de `validated_at`.
- **`read_validation_flags` faz um GET por contato** (N+1) e `list_contacts_in_status` não segue `@odata.nextLink`, então o sync silenciosamente ignora contatos além da primeira página quando o volume crescer.
- **`SEM_RESPOSTA` não tem quem o defina.** O estado existe no enum e na máquina de estados, mas nada no código transiciona para ele — provavelmente cabe ao Power Automate, e isso precisa estar escrito.
- **`EMPRESA_NAO_ENCONTRADA` é um estado de contato aplicado a uma empresa.** R1.2 define esse status quando ainda não existe contato algum. Vale um enum separado para o resultado da resolução de empresa.
