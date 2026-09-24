# Process Log — Linha do tempo das iterações

Registro de como a análise evoluiu, onde a IA (Claude, no Cowork) entrou e onde eu corrigi o rumo. O chat completo está em `chat-export/`.

## Fase 1 — Entender o problema antes de codar

- **Pedido ao Claude:** ler o desafio do G4 e as soluções públicas desse dataset (Kaggle/GitHub), se colocar no lugar do CEO e **não escrever código**.
- **Resultado:** mapa da referência a superar. As soluções públicas concluem "DevTools churna mais, plano não importa, uso não prediz" e recomendam migrar clientes para o anual.
- **Decisão minha:** pensar em receita (MRR) e em como reconciliar as três falas do CEO (churn subiu, satisfação ok, uso cresceu), e não só em correlações.

## Fase 2 — Análise bivariada (5 tabelas)

| Iteração | O que fiz | O que a revisão apontou | Correção |
|---|---|---|---|
| 1 | Histogramas e boxplots contra `churn_flag` em cada tabela | Merge tickets × assinaturas multiplicou 2.000 tickets para ~20 mil linhas, com 63% nos dois grupos. Boxplots de uso feitos sobre colunas de texto. Gráficos de colunas de ID (13 MB de notebook). | Merge com `df_accounts`, `select_dtypes('number')`, IDs fora dos loops |
| 2 | Primeiro teste qui-quadrado | Rodado sobre linhas duplicadas (748 em vez de 500): industry saiu de p = 0,07 para p = 0,0002. `account_name` deu "significativo". Qui-quadrado aplicado a variáveis contínuas e datas. | 1 linha por entidade, Mann-Whitney para numéricas |
| 3 | Função de teste com Mann-Whitney | Faltava `continue`: o qui-quadrado continuava rodando nas numéricas e mantinha o falso positivo de `first_response_time` | Corrigido; todos os p-valores passaram a ser impressos |
| 4 | Markdowns das seções | Os textos contradiziam os testes (ainda citavam DevTools como causa) | Reescritos com os p-valores |

## Fase 3 — O "quando" em vez do "quem"

- Gráfico de churn por coorte sugeria que 2023 era pior. **A revisão apontou viés de exposição** (coortes antigas tiveram mais tempo para cancelar).
- Criei `days_to_churn` e comparei as coortes em **janela fixa** (30/60/90 dias). O churn em 30 dias saltou de ~1% para 6% em 2024Q4.
- Li errado o `size` do gráfico como número de churns (era o número de assinaturas novas). Corrigido na revisão.
- **Pergunta de dono que mudou a manchete:** "os US$ 157 mil de churn precoce são quanto da perda total do Q4?" → **75% da MRR perdida no Q4 veio de assinaturas com até 90 dias.**

## Fase 4 — Quem são os que saem cedo

- Tabela analítica com 1 linha por assinatura; uso e tickets restritos aos primeiros 30 dias.
- `seats_acct` apareceu com p = 0,006. **Antes de aceitar:** refiz com 1 linha por conta (p = 0,001), apliquei Bonferroni, descartei viés de exposição (assinaturas por conta parecidas entre as faixas) e mostrei dose-resposta (30% → 16%).
- O código sugerido pelo Claude ordenou as faixas alfabeticamente (`astype(str)` num `qcut`) e escondeu a dose-resposta. Corrigido com rótulos ordenados.
- 80% das assinaturas não têm uso registrado nos primeiros 30 dias; ~80% de quem sai cedo nunca abriu ticket → churn silencioso.

## Fase 5 — Contas em risco e modelagem

- Lista de risco por regra (probabilidade da faixa de seats × MRR). A primeira versão usava o histórico inteiro e **subestimava a perda esperada em ~60%**. Recalibrei com as coortes de 2024H2 (US$ 50 mil → US$ 81 mil/mês).
- Corte de Pareto: 40 contas concentram 50% da perda esperada.
- Df de treino só com informação do dia 1 (removi `churn_flag`, flags de conta sem data e uso/tickets da janela). Validação temporal: treino até 2024Q3, teste em 2024Q4.
- XGBoost, regressão logística e a regra: ROC-AUC ≈ 0,5. A acurácia de 93% escondia o problema. Troquei por Average Precision contra a linha de base, lift no top 10% e **IC 95% por bootstrap** (todos incluem o acaso).
- **Decisão minha:** não forçar um modelo. Transformei o resultado em argumento estratégico: se não dá para prever quem sai, o problema é o processo de entrada, e o conserto vale para todos.

## Fase 6 — Relatório

- Rascunho do README gerado pelo Claude a partir dos meus números; revisado e ajustado por mim.

## Como usei a IA (em resumo)

- **Modo conselheiro:** a partir da Fase 2 pedi que o Claude fizesse perguntas e revisasse, em vez de entregar respostas prontas.
- **Revisão crítica a cada rodada:** o notebook era relido inteiro após cada mudança, o que encontrou bugs de merge, de tipo e de teste.
- **Nada foi aceito sem checagem:** os números do relatório vêm dos outputs do notebook, e o código sugerido pela IA também teve bugs que eu corrigi.
