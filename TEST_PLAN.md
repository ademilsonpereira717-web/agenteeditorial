# Estratégia de Testes

## Pirâmide de testes

1. **Unitários (`core/`)**: funções puras para transição, versão elegível, contagem, retenção/redução, enum A/B/C/D, cobertura e validação de caminho.
2. **Integração (`app/` + SQLite temporário + diretório temporário)**: comandos, transações, persistência de arquivo/metadata e histórico.
3. **Contrato de IA (`ai/`)**: respostas gravadas e inválidas contra esquemas; nenhum teste de domínio depende de modelo remoto.
4. **Fim a fim (`ui/`/API)**: fluxo humano mínimo, especialmente paradas nos checkpoints e dupla submissão.
5. **Propriedades e regressão**: Hypothesis para intervalos, estados e métricas; fixtures anônimas e explicitamente marcadas como teste, jamais disponíveis em produção.

## Casos obrigatórios de aceitação

| # | Cenário | Nível | Resultado esperado |
| --- | --- | --- | --- |
| 1 | Obra sem fonte ou perfil | Integração | Permanece/bloqueia em `ARQUIVOS_INCOMPLETOS`. |
| 2 | Duas obras | Integração | Caminho, IDs, consultas e versões não se cruzam. |
| 3 | Tentativa de alterar transcrição | Integração | Operação rejeitada; hash e bytes permanecem iguais. |
| 4 | Mapa parcial | Unitário/integração | Lacuna reduz cobertura e impede Checkpoint A. |
| 5 | Cascata após concluir lote | Integração | Abre checkpoint, sem iniciar lote seguinte. |
| 6 | Versão reprovada/obsoleta | Unitário | Não é elegível como entrada de lote seguinte. |
| 7 | Retenção | Unitário | Fórmula e arredondamento corretos. |
| 8 | Redução | Unitário | É exatamente `100 - retenção`. |
| 9 | Enum A/B/C/D | Unitário/contrato | Valores fixos; entrada com redefinição é recusada. |
| 10 | Ocorrência D | Integração | `AUSENCIA_DE_INVENCOES` reprovada e bloco em ajuste. |
| 11 | Clique duplo no bloco | Integração concorrente | Uma execução; a outra é ignorada e registrada. |
| 12 | Iniciar B2 antes de B1 aprovado | Integração | Comando recusado. |
| 13 | Conteúdo simulado em produção | Integração | Modo produção exige fonte existente, hash e snapshot reais; placeholder/cache é recusado. |
| 14 | Falha ao gravar/validar | Integração | Estado seguro, sem versão promovida, evento de erro. |
| 15 | Transições relevantes | Integração | Histórico append-only contém ator, momento, correlação e dados. |

## Testes adicionais recomendados

* Propriedade: intervalos válidos sem lacuna cobrem exatamente as palavras que o contador atribui; intervalos embaralhados, sobrepostos ou fora dos limites falham.
* Propriedade: nenhuma sequência aleatória de comandos alcança `OBRA_CONSOLIDADA` sem quatro aprovações humanas.
* Concorrência: duas requisições simultâneas para a mesma chave idempotente não criam duas linhas nem dois arquivos.
* Segurança: `obra_id` com `..`, separadores ou caminho absoluto é rejeitado; uma versão com `obra_id` divergente no cabeçalho falha na validação.
* Auditoria: `REMOVIDO_SUBSTANTIVO` impede aprovação automática; evidência vazia é inválida.
* Regressão: fixtures representativas somente após autorização e tratamento apropriado de conteúdo autoral.

## Critérios de saída para implementação futura

O Lote 1 não será considerado implementado até que todos os 15 casos obrigatórios estejam automatizados e verdes, cobertura de ramos do domínio crítico seja acordada (sugestão inicial: 90%+) e os arquivos metodológicos tenham uma matriz de rastreabilidade para testes/regras correspondentes.

