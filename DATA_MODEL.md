# Modelo de Dados e Versionamento

## Entidades principais

| Entidade | Campos essenciais | Regras de integridade |
| --- | --- | --- |
| `obra` | `obra_id`, `estado`, `criada_em`, `atualizada_em` | `obra_id` único e imutável. |
| `arquivo_fonte` | `obra_id`, `tipo`, `caminho`, `sha256`, `importado_em`, `imutavel` | `tipo=TRANSCRICAO_BRUTA` é único por obra e imutável. |
| `snapshot_execucao` | `id`, `obra_id`, `lote`, `fonte_sha256`, `versao_entrada_id`, `trecho_inicial`, `trecho_final`, `protocolo_hash` | Obrigatório antes de iniciar lote. |
| `versao_artefato` | `id`, `obra_id`, `arquivo_tipo`, `numero`, `status`, `caminho`, `sha256`, `criada_em`, `origem`, `checkpoint_associado` | Único em `(obra_id, arquivo_tipo, numero)`; não pode apontar fora da obra. |
| `bloco` | `id`, `obra_id`, `ordem`, `inicio_offset`, `fim_offset`, `palavras_atribuidas`, `estado` | Único em `(obra_id, ordem)`; intervalos ordenados e dentro da fonte. |
| `execucao_bloco` | `id`, `obra_id`, `bloco_id`, `versao_entrada`, `estado`, `iniciada_em`, `terminada_em` | Chave de idempotência única `(obra_id, bloco_id, versao_entrada)`. |
| `metrica` | `obra_id`, `escopo`, `versao_id`, `palavras_original`, `palavras_transformado`, `retencao`, `reducao`, `faixa` | Calculada pelo código e armazenada com precisão decimal. |
| `evidencia_auditoria` | `id`, `obra_id`, `bloco_id`, `categoria`, `evidencia_original`, `evidencia_transformada`, `status` | Evidências devem referenciar a mesma obra/bloco. |
| `pendencia` | `id`, `obra_id`, `origem`, `descricao`, `trecho_fonte`, `status` | Nunca substitui nem “corrige” a fonte. |
| `evento_historico` | `id`, `obra_id`, `tipo`, `ocorrido_em`, `ator`, `dados_json`, `correlacao_id` | Somente inserção; registra toda ação relevante. |

## Enumerações imutáveis

Estas enumerações pertencem ao código de domínio e a uma migração versionada; não são configuráveis por prompt, interface ou resposta de IA.

```text
Intervencao:
  A = SOURCE_EXPLICIT
  B = DIRECT_REFORMULATION
  C = NECESSARY_CONNECTION
  D = UNSUPPORTED_EXPANSION

StatusVersao:
  ATIVA | APROVADA | REPROVADA | OBSOLETA | EM_AJUSTE

StatusEvidencia:
  PRESERVADO | CONDENSADO | REMOVIDO_ORALIDADE |
  REMOVIDO_SUBSTANTIVO | ALTERADO | NAO_LOCALIZADO
```

`D` é proibido. Se a auditoria validada registrar contagem de D maior que zero, a regra de domínio define `AUSENCIA_DE_INVENCOES=REPROVADA` e o bloco fica em `AJUSTE_NECESSARIO`. Listas especiais de ressalvas editoriais serão implementadas como detectores determinísticos e poderão aumentar a exigência de evidência, mas não substituirão a auditoria humana/qualitativa.

## Contrato de versão

Todo Markdown gerado começa com front matter ou cabeçalho de metadados contendo obrigatoriamente:

```yaml
obra_id: <obra_id>
arquivo_tipo: 03_MAPA_EDITORIAL
versao: 2
status: EM_AJUSTE
data_criacao: 2026-08-30T00:00:00Z
origem: lote_1/execucao/<id>
checkpoint_associado: A
```

O banco é a fonte de autoridade para seleção de versões; o cabeçalho serve para inspeção humana e verificação de consistência. Uma fase seguinte só pode selecionar versão com `status=APROVADA` (ou `ATIVA` quando a regra da fase permitir explicitamente), hash íntegro e pertencente à mesma obra. `REPROVADA` e `OBSOLETA` são inelegíveis. Ao aprovar substituta, a versão anteriormente ativa do mesmo artefato torna-se `OBSOLETA`, preservando arquivo e histórico.

## Métricas determinísticas

Para uma mesma política de tokenização/contagem, aplicada à fonte e à saída:

```text
retencao = palavras_transformado / palavras_original * 100
reducao = 100 - retencao
```

Classificação: `FAIXA_NORMAL` para retenção >= 60%; `ALERTA_AMARELO` de 45% a 59%; `ALERTA_VERMELHO` de 30% a 44%; e `NAO_APROVAR_AUTOMATICAMENTE` abaixo de 30%. A retenção é alarme diagnóstico, nunca meta de escrita e jamais autorização isolada de aprovação.

## Cobertura estrutural

O mapa persiste intervalos de caracteres/linhas da fonte e a contagem de palavras resultante deles. O validador ordena intervalos, verifica início/fim, lacunas e sobreposições, e calcula:

```text
cobertura = palavras_cobertas_blocos / palavras_fonte * 100
```

Ele também produz relatório de trechos não atribuídos. Checkpoint A não pode abrir enquanto houver cobertura abaixo do limiar metodológico ou lacuna substantiva. A faixa de 3.000–5.000 palavras é uma preferência de planejamento para textos longos, não uma restrição do validador.

