# Plano de Implementação

## Situação atual: parada deliberada

Esta primeira entrega está limitada a documentos de arquitetura. Não há Lote 1 implementado, não há chamadas de IA, não há fonte simulada e não há dados editoriais produzidos. A próxima etapa depende de aprovação humana e da disponibilização em `config/` dos quatro arquivos metodológicos íntegros.

## Fase 0 — Congelamento da especificação

1. Adicionar os arquivos `03` a `06` em `config/`, preservar seus nomes e calcular hashes.
2. Ler integralmente seu conteúdo e criar matriz `requisito metodológico → regra de domínio → teste`.
3. Resolver explicitamente ambiguidades: limiar de cobertura “significativamente abaixo”, comportamento de reprovação nos checkpoints D/E, política de tokenização e requisitos precisos de auditoria.
4. Versionar a matriz e impedir execução se os hashes da especificação não forem os aprovados.

## Fase 1 — Fundação determinística (antes do Lote 1)

1. Inicializar projeto Python/TypeScript, lint, tipos, testes, `.gitignore` para `var/` e `.env.example` sem segredos.
2. Criar esquema SQLite/migrações para obra, fonte, versões, eventos, blocos, execuções e snapshots.
3. Implementar isolamento por `obra_id`, importação imutável da fonte, hashes e escrita atômica de artefatos.
4. Implementar máquina de estados, autorizações humanas, histórico append-only e seleção de versão elegível.
5. Implementar métricas, cobertura por intervalos e enum A/B/C/D imutável.
6. Escrever e aprovar os testes obrigatórios desta fundação antes de qualquer chamada de IA.

## Fase 2 — Contrato de IA e observabilidade

1. Definir interfaces de adaptador e modelos Pydantic para cada tarefa editorial futura.
2. Forçar resposta estruturada com IDs da obra, hashes de entrada, evidências e classificações permitidas.
3. Registrar metadados de chamada (provedor, modelo, latência, hashes de entrada/saída e correlação), sem segredo e sem permitir que a resposta execute ações.
4. Criar dublês determinísticos para testes e um modo produção que rejeite placeholders, fixtures e cache de outra obra.

## Fase 3 — Lote 1 (somente após as fases anteriores e aprovação)

1. Validar entradas e criar snapshot real da fonte.
2. Orquestrar diagnóstico e proposta de mapa por IA sob contrato.
3. Validar deterministicamente cobertura, ordem, intervalos, versões e pendências.
4. Emitir versões de `01_RELATORIO_DE_INGESTAO`, `02_DIAGNOSTICO_AUTORAL`, `03_MAPA_EDITORIAL`, `04_ORIENTACOES_DE_TRANSFORMACAO` e, quando necessário, `12_RELATORIO_DE_PENDENCIAS`.
5. Abrir — sem aprovar — o Checkpoint A, apresentando evidência, cobertura e histórico à pessoa revisora.

## Fases posteriores (não iniciar nesta entrega)

* **Lote 2:** transformação sequencial e auditoria por bloco, com idempotência e bloqueio de D.
* **Lote 3:** consolidação apenas de blocos aprovados; abertura do Checkpoint D.
* **Lote 4:** auditoria global e correções locais; abertura do Checkpoint E.
* **Lote 5:** corpo final e materiais permitidos, sem inventar elementos autorais.

Cada fase só começa mediante aprovação humana do checkpoint precedente e ampliação correspondente da matriz de testes.

