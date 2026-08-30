# Máquina de Estados

## Princípios

A máquina de estados é implementada no domínio, persistida no banco e auditada no histórico. A UI apenas solicita comandos; ela não calcula o próximo estado. Toda transição exige obra existente, identidade de obra consistente, versão de entrada elegível e as pré-condições abaixo. Qualquer transição não listada é recusada.

## Estados da obra e transições permitidas

| Estado atual | Comando | Próximo estado | Pré-condições |
| --- | --- | --- | --- |
| `NOVA_OBRA` | avaliar_arquivos | `ARQUIVOS_INCOMPLETOS` | Falta fonte ou perfil obrigatório. |
| `NOVA_OBRA` ou `ARQUIVOS_INCOMPLETOS` | avaliar_arquivos | `PRONTA_PARA_LOTE_1` | Fonte e perfil presentes, fonte íntegra e configurações metodológicas disponíveis. |
| `PRONTA_PARA_LOTE_1` | iniciar_lote_1 | `LOTE_1_EM_EXECUCAO` | Snapshot da fonte registrado. |
| `LOTE_1_EM_EXECUCAO` | concluir_lote_1 | `CHECKPOINT_A_AGUARDANDO_APROVACAO` | Saídas válidas, versões candidatas e cobertura aprovada. |
| `CHECKPOINT_A_AGUARDANDO_APROVACAO` | reprovar_checkpoint_a | `CHECKPOINT_A_REPROVADO` | Decisão humana e motivo registrados. |
| `CHECKPOINT_A_REPROVADO` | solicitar_ajuste_lote_1 | `LOTE_1_EM_EXECUCAO` | Nova versão de trabalho definida. |
| `CHECKPOINT_A_AGUARDANDO_APROVACAO` | aprovar_checkpoint_a | `CHECKPOINT_A_APROVADO` | Decisão humana; versões elegíveis marcadas aprovadas. |
| `CHECKPOINT_A_APROVADO` | liberar_lote_2 | `PRONTA_PARA_LOTE_2` | Somente versões aprovadas selecionadas. |
| `PRONTA_PARA_LOTE_2` | iniciar_lote_2 | `LOTE_2_EM_EXECUCAO` | Checkpoint A aprovado. |
| `LOTE_2_EM_EXECUCAO` | concluir_lote_2 | `CHECKPOINT_C_AGUARDANDO_APROVACAO` | Todos os blocos requeridos aprovados. |
| `CHECKPOINT_C_AGUARDANDO_APROVACAO` | reprovar_checkpoint_c | `CHECKPOINT_C_REPROVADO` | Decisão humana e motivo registrados. |
| `CHECKPOINT_C_REPROVADO` | solicitar_ajuste_lote_2 | `LOTE_2_EM_EXECUCAO` | Nova versão de trabalho definida. |
| `CHECKPOINT_C_AGUARDANDO_APROVACAO` | aprovar_checkpoint_c | `CHECKPOINT_C_APROVADO` | Decisão humana. |
| `CHECKPOINT_C_APROVADO` | liberar_lote_3 | `PRONTA_PARA_LOTE_3` | Entradas do Lote 3 aprovadas. |
| `PRONTA_PARA_LOTE_3` | iniciar_lote_3 | `LOTE_3_EM_EXECUCAO` | Checkpoint C aprovado. |
| `LOTE_3_EM_EXECUCAO` | concluir_lote_3 | `CHECKPOINT_D_AGUARDANDO_APROVACAO` | Somente blocos aprovados consolidados. |
| `CHECKPOINT_D_AGUARDANDO_APROVACAO` | aprovar_checkpoint_d | `CHECKPOINT_D_APROVADO` | Decisão humana. |
| `CHECKPOINT_D_APROVADO` | liberar_lote_4 | `PRONTA_PARA_LOTE_4` | Checkpoint D aprovado. |
| `PRONTA_PARA_LOTE_4` | iniciar_lote_4 | `LOTE_4_EM_EXECUCAO` | Snapshot de entradas aprovado. |
| `LOTE_4_EM_EXECUCAO` | concluir_lote_4 | `CHECKPOINT_E_AGUARDANDO_APROVACAO` | Auditoria global atende critérios críticos. |
| `CHECKPOINT_E_AGUARDANDO_APROVACAO` | aprovar_checkpoint_e | `CHECKPOINT_E_APROVADO` | Decisão humana. |
| `CHECKPOINT_E_APROVADO` | liberar_lote_5 | `PRONTA_PARA_LOTE_5` | Checkpoint E aprovado. |
| `PRONTA_PARA_LOTE_5` | iniciar_lote_5 | `LOTE_5_EM_EXECUCAO` | Checkpoint E aprovado. |
| `LOTE_5_EM_EXECUCAO` | concluir_lote_5 | `OBRA_CONSOLIDADA` | Corpo final e artefatos exigidos registrados. |

Os estados de reprovação de D e E não constam na lista mínima fornecida. Uma reprovação humana nesses checkpoints, se o protocolo o exigir, será registrada como solicitação de ajuste sem transição automática para a etapa seguinte; a regra final deve ser confirmada na especificação metodológica antes da implementação.

## Máquina de estados do bloco (Lote 2)

```text
PENDENTE → EM_TRANSFORMACAO → EM_AUDITORIA → AGUARDANDO_VALIDACAO
                                              ├→ APROVADO
                                              ├→ AJUSTE_NECESSARIO → EM_TRANSFORMACAO
                                              └→ REPROVADO
EM_TRANSFORMACAO | EM_AUDITORIA → ERRO_DE_EXECUCAO
ERRO_DE_EXECUCAO → EM_TRANSFORMACAO  (somente por nova execução explícita)
```

* O primeiro bloco pode iniciar somente em `LOTE_2_EM_EXECUCAO`.
* O bloco N só inicia se N−1 estiver em `APROVADO`.
* Uma chamada com chave idempotente já em `EM_TRANSFORMACAO`, `EM_AUDITORIA` ou `AGUARDANDO_VALIDACAO` é ignorada, sem criar nova execução.
* Resultado com qualquer ocorrência `D = UNSUPPORTED_EXPANSION` muda o bloco para `AJUSTE_NECESSARIO` e reprova `AUSENCIA_DE_INVENCOES`.
* Falhas técnicas nunca promovem bloco, versão ou checkpoint.

## Checkpoints

Os checkpoints A, C, D e E são barreiras duras: `concluir_lote_*` abre o checkpoint, e apenas um comando humano posterior pode aprová-lo. A ação de aprovação não inicia o lote seguinte: ela somente chega ao estado de aprovado; um segundo comando explícito o libera. Cada abertura, aprovação, reprovação e ajuste solicitado gera evento imutável no histórico.

