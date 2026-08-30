# AGENTE EDITORIAL DE LIVRO — PRODUÇÃO ASSISTIDA v1.0

## Escopo desta primeira entrega

Esta entrega define a arquitetura do sistema; ela **não implementa o Lote 1** nem executa IA sobre qualquer conteúdo. Os quatro arquivos metodológicos estáveis — `03_REGRAS_EDITORIAIS.md`, `04_PROTOCOLO_MESTRE_v2.2.1.md`, `05_PROMPT_OPERACIONAL_WORK_v2.2.1.md` e `06_CHECKLIST_DE_VALIDACAO_DO_AGENTE_v2.2.1.md` — ainda não estão presentes neste repositório. Antes de qualquer implementação editorial, eles devem ser inseridos integralmente em `config/`, lidos e convertidos em requisitos rastreáveis. Até isso acontecer, a aplicação deve bloquear a execução de lotes.

## Decisões arquiteturais

O sistema será uma aplicação local monolítica, com uma API local e interface web local. A separação é lógica, não uma rede de microserviços:

| Camada | Responsabilidade | Não pode fazer |
| --- | --- | --- |
| `core/` | Regras determinísticas, máquina de estados, versões, métricas, arquivos, trilha de auditoria e bloqueios. | Tomar decisões editoriais por IA ou aceitar transições inválidas. |
| `ai/` | Adaptador de provedor, contratos de entrada/saída estruturada e registro de chamadas. | Alterar estados, calcular métricas oficiais, criar versões válidas ou redefinir regras. |
| `app/` | Casos de uso que orquestram `core` e `ai`. | Contornar validações de domínio. |
| `ui/` | Visualização e comandos humanos explícitos de aprovação/reprovação. | Avançar checkpoint automaticamente. |
| `storage/` | SQLite para metadados/transações e diretórios por obra para artefatos Markdown. | Compartilhar caminhos ou artefatos entre obras. |

## Stack proposta

* **Python 3.12**, por ser adequado a automação local, tipagem, processamento de texto e testes.
* **FastAPI + Pydantic v2** para API local, contratos de dados e validação de comandos.
* **SQLite** (via SQLAlchemy) como base local transacional; migrações com Alembic. A integridade entre obra, versão, bloco, execução e histórico deve ser imposta também no banco.
* **Typer** para uma CLI administrativa reprodutível (criar obra, importar fonte, abrir/aprovar checkpoint e consultar histórico).
* **React + TypeScript + Vite** para UI local, consumindo exclusivamente a API.
* **Pytest + Hypothesis** para testes unitários, integração, propriedades e regressões. `ruff`, `mypy` e Prettier/ESLint sustentam qualidade estática.
* Um **adaptador de IA** isolado e configurado por variável de ambiente. Chaves nunca entram no repositório, nos Markdown de obra ou no histórico.

Essa escolha mantém um único processo implantável localmente e permite futura troca de SQLite por PostgreSQL e de armazenamento local por objeto remoto, sem mudar as regras de domínio.

## Estrutura de diretórios proposta

```text
.
├── config/                         # especificação metodológica estável, versionada
│   ├── 03_REGRAS_EDITORIAIS.md
│   ├── 04_PROTOCOLO_MESTRE_v2.2.1.md
│   ├── 05_PROMPT_OPERACIONAL_WORK_v2.2.1.md
│   └── 06_CHECKLIST_DE_VALIDACAO_DO_AGENTE_v2.2.1.md
├── obras/
│   └── <obra_id>/
│       ├── fonte/
│       │   ├── 01_TRANSCRICAO_BRUTA.md
│       │   └── 02_PERFIL_DO_AUTOR.md
│       ├── lote_1/ ... lote_5/     # artefatos versionados de cada lote
│       ├── pendencias/
│       ├── historico/
│       ├── estado/
│       └── versoes/
├── core/                            # domínio puro e determinístico
├── app/                             # casos de uso/API/CLI
├── ai/                              # adaptadores de IA e esquemas de resposta
├── ui/                              # frontend local
├── tests/
├── migrations/
└── var/                             # banco SQLite e dados de execução locais; ignorado pelo Git
```

`obra_id` é um identificador imutável, normalizado e único. Todo caminho de artefato é resolvido a partir dele e validado contra travessia de diretório. Toda consulta e toda chave estrangeira carregam `obra_id`; nenhum caso de uso aceita um arquivo arbitrário de outra obra.

## Fonte, artefatos e rastreabilidade

A importação inicial copia a transcrição para `fonte/01_TRANSCRICAO_BRUTA.md` uma única vez, calcula SHA-256 e aplica permissão de somente leitura quando suportado. Alteração posterior é rejeitada; uma fonte corrigida exige nova obra ou um fluxo futuro explicitamente aprovado de nova fonte, sem apagar a anterior. O perfil do autor também é versionado e tem hash, mas não é a fonte soberana.

Antes de cada lote, o sistema persiste um *snapshot de contexto*: `obra_id`, caminho canônico da fonte, hash, versão ativa de entrada, trecho inicial, trecho final, versão do protocolo e identificador da execução. Se a fonte, o hash ou a versão ativa não forem localizáveis, o lote é bloqueado.

Cada saída é escrita primeiro como arquivo novo, por exemplo `lote_1/03_MAPA_EDITORIAL_v2.md`; então recebe um registro de versão. A gravação de arquivo e o registro de metadados ocorrem em operação controlada, com arquivo temporário e renomeação atômica. Artefatos nunca são substituídos silenciosamente.

## Limites entre determinismo e IA

O adaptador de IA recebe somente uma solicitação contextualizada com IDs, hashes, excertos autorizados e uma instrução de devolver JSON validável. Ele pode produzir diagnóstico, propostas de blocos, transformação, auditoria qualitativa, análise de voz e classificações A/B/C/D por item. A resposta é uma **proposta não confiável** até que validadores determinísticos confirmem esquema, `obra_id`, referências de origem, enumerações e pré-condições.

Somente `core` pode: contar palavras; calcular cobertura, retenção e redução; classificar a faixa de retenção; declarar A/B/C/D; bloquear D; escolher versão elegível; mudar estado; abrir checkpoint; ou registrar aprovação humana. A IA não recebe ferramenta para banco, sistema de arquivos, transição de estado ou aprovação.

## Controles críticos

* **Cobertura estrutural:** blocos guardam intervalos ordenados de offsets da fonte, além de contagem de palavras. Validador detecta lacunas, sobreposições e intervalos fora da fonte; calcula cobertura pelo texto efetivamente atribuído. O limiar de bloqueio e a definição de “significativamente abaixo” serão extraídos literalmente dos arquivos metodológicos antes da implementação.
* **Idempotência:** uma tabela de execuções possui chave única `(obra_id, bloco_id, versao_entrada)`. Estados em processamento recusam nova chamada e produzem evento `EXECUCAO_DUPLICADA_IGNORADA`.
* **Sequência:** o comando de transformar o bloco N exige todos os blocos anteriores aprovados e o estado da obra correto.
* **Aprovação humana:** comandos distintos e autenticados para aprovar/reprovar checkpoint e bloco; nenhuma conclusão de lote executa o próximo lote.
* **Falha segura:** exceções deixam a obra em estado estável, registram erro e não promovem versão parcialmente criada.

## Riscos técnicos e mitigação

| Risco | Mitigação arquitetural |
| --- | --- |
| Especificação metodológica ausente ou ambígua | Bloquear implementação editorial e manter matriz requisito-origem após importação de `config/`. |
| IA inventar ou reclassificar regras | Enum imutável no código, resposta estruturada, evidências de origem obrigatórias e bloqueio automático para D. |
| Perda de conteúdo por mapa parcial | Intervalos determinísticos, cobertura e testes de lacuna/sobreposição antes do Checkpoint A. |
| Corrupção por concorrência/double-click | Transação SQLite, chave de idempotência, lock lógico e histórico da tentativa ignorada. |
| Contaminação entre obras | Escopo obrigatório por `obra_id`, chaves compostas e validação de caminhos/hashes. |
| Arquivos parcialmente gravados | Escrita temporária, hash, renomeação atômica e promoção de versão apenas após validação. |
| Vazamento de segredo ou conteúdo | `.env`/`var/` ignorados, segredo apenas em ambiente local e logs sem credenciais. |
| Auditorias afirmativas sem prova | Evidência original/transformada por item e estados de cobertura prescritos. |

