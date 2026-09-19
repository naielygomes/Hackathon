# Cadastro de técnicos: padronização e sincronização unidirecional

Problema: o mesmo técnico existe hoje em três bases (BTime, HubSpot, OPS) cadastrado de forma diferente, sem integração. Some a isso um segundo formato de prestador: cooperativas e oficinas, onde o vínculo contratual e fiscal é só com o representante, mas quem efetivamente vai a campo são os técnicos vinculados a ela.

Este documento propõe o modelo de dados, o fluxo de sincronização unidirecional e o procedimento manual para os casos que a sincronização não resolve sozinha.

## 1. Por que unidirecional, e por que OPS como origem

As três bases têm papéis diferentes e nenhuma delas deveria "vencer" as outras nos campos que não são dela:

| Base | Para que serve | Dono natural dos dados de identidade? |
|---|---|---|
| **OPS** | Onde Serviços gerencia a relação operacional com o prestador (contrato assinado, emite nota, técnicos de uma cooperativa) | **Sim** — é onde a tela "Técnicos" já registra `Contrato assinado` e `Emite nota` |
| **HubSpot** | CRM comercial (funil, negócio, card do cliente) | Não — precisa do técnico só como referência (Company/Contact) para associar ao card |
| **BTime** | Execução em campo (checklist, OS, disponibilidade) | Não — precisa saber *quem pode receber OS*, não decide quem é técnico |

Se qualquer uma das três pudesse escrever identidade nas outras, viraria disputa de verdade (exatamente o que gera hoje divergência de cadastro). Por isso o desenho é **unidirecional, sempre**: **OPS → HubSpot** e **OPS → BTime**, só para os campos de identidade/vínculo contratual. Dados de execução (histórico de OS, checklist) continuam vivendo na BTime; dados comerciais (etapa do negócio) continuam vivendo no HubSpot. Nenhum dos dois escreve de volta identidade no OPS automaticamente — isso é o item 4 (Divergências), tratado manualmente.

Isso confirma e reforça a proposta já colocada na rodada anterior: **OPS como fonte mestre, operação com o time de Serviços, código com quem herdar o domínio de técnico no Hermes.**

## 2. Modelo de dados: separar "quem assina" de "quem vai a campo"

O ponto que gera confusão hoje é tratar cooperativa/oficina como se fosse só "mais um técnico". São duas coisas diferentes:

```
EntidadeContratual (quem tem CNPJ/CPF, contrato assinado, emite nota)
 ├─ tipo: pessoa_fisica_autonoma
 │   └─ Tecnico (1:1 — a própria pessoa é quem vai a campo)
 └─ tipo: cooperativa_oficina
     ├─ representante (contato fiscal/contratual — é quem assina, não quem vai a campo)
     └─ Tecnico[] (membros — cada um dispachável individualmente)
```

- **`EntidadeContratual`** carrega tudo que hoje já aparece na tela "Técnicos" do OPS: CPF/CNPJ, `Contrato assinado`, `Emite nota`, status (`Ativo`/`Inativo`).
- **`Tecnico`** é a unidade despachável — a que entra no algoritmo de pontuação (distância, avaliação, histórico), recebe janela na agenda e OS na BTime. Todo `Tecnico` aponta para uma `EntidadeContratual` (a própria, se autônomo; a cooperativa, se membro).
- O representante da cooperativa **não é um `Tecnico`** — ele não deveria aparecer na agenda nem no ranking de pontuação, só nas telas de contrato/faturamento.

Essa separação evita o erro mais comum: cadastrar o representante da cooperativa como se ele fosse quem executa o serviço, ou vice-versa.

## 3. Fluxo de sincronização

Reaproveita o mesmo padrão já validado na tela "Depois do aceite" (assíncrono, idempotente por estágio, com fila e reprocessamento) — não é um mecanismo novo, é o mesmo aplicado a outro evento.

```
Cadastro/edição no OPS (EntidadeContratual ou Tecnico)
        │
        ▼
   Fila de eventos (outbox) — 1 evento por mudança de campo relevante
        │
        ├──► Adapter HubSpot ──► Company (cooperativa) ou Contact (autônomo)
        │        idempotente por (tecnico_id, campo) · retry automático
        │        falha ──► fila "Sincronização" · ação manual "Reprocessar"
        │
        └──► Adapter BTime ──► cadastro do técnico habilitado a receber OS
                 idempotente por (tecnico_id, campo) · retry automático
                 falha ──► fila "Sincronização" · ação manual "Reprocessar"
```

Tabela de correspondência (crosswalk), nova, mínima e a peça que falta hoje:

| Campo | Descrição |
|---|---|
| `tecnico_id` (OPS, chave) | ID interno, gerado no cadastro |
| `hubspot_id` | Contact ou Company ID retornado na primeira sincronização |
| `btime_id` | ID do técnico na BTime retornado na primeira sincronização |
| `entidade_contratual_id` | Para quem é membro de cooperativa |
| `ultima_sincronizacao` / `status` | `Sincronizado` · `Na fila` · `Falhou` |

Sem essa tabela, não tem como saber que "Rafael Moreira" no OPS é o mesmo "R. Moreira" no HubSpot e "MOREIRA, RAFAEL" na BTime — é a causa raiz da duplicidade atual.

## 4. Divergências pós-sync (por que continua unidirecional mesmo aí)

BTime e HubSpot podem mudar um dado no dia a dia (ex.: técnico atualiza telefone direto na BTime). Isso **não** volta sozinho para o OPS. Um job diário compara os três lados usando o crosswalk e, se achar diferença, cria um item na fila **"Divergências"** — visível para o time de Serviços decidir se atualiza o OPS manualmente. Isso preserva o unidirecional (nunca escrita automática de volta) sem perder a mudança real.

## 5. Procedimento manual — cadastro novo (daqui para frente)

1. Todo técnico novo — autônomo ou cooperativa — é cadastrado **primeiro e só no OPS**, nunca direto na BTime ou no HubSpot.
2. Campos obrigatórios para publicar: nome, CPF/CNPJ, telefone, tipo (`autônomo` ou `cooperativa/oficina`); se cooperativa, nome e contato do representante.
3. Ao salvar, o evento entra na fila de sincronização (seção 3) — a tela de cadastro deve mostrar o mesmo indicador de status que "Depois do aceite" já usa (`Sincronizado` / `Na fila` / `Falhou`), para quem cadastrou saber se o técnico já pode receber serviço.
4. Cadastro de membro de cooperativa: sempre vinculado a uma `EntidadeContratual` já existente — a tela deve impedir salvar um `Tecnico` sem `entidade_contratual_id`.
5. Desligamento: inativar no OPS propaga `Ativo → Inativo` para os dois destinos; nunca apagar (preserva histórico de OS já vinculado).

## 6. Procedimento manual — reconciliação do estoque atual (migração única)

O cadastro que já existe hoje nas três bases, divergente, não se resolve com o fluxo acima sozinho — precisa de uma migração única antes de ligar a sincronização automática:

1. Exportar das três bases: nome, telefone, CPF/CNPJ, cidade.
2. **Match forte** — por CPF/CNPJ igual nas três: concilia automaticamente, gera o `tecnico_id` e preenche o crosswalk.
3. **Match fraco** — telefone + nome semelhante (fuzzy), sem CPF/CNPJ batendo: cai numa fila de revisão manual, uma linha por candidato, para o time de Serviços confirmar ou rejeitar.
4. **Sem match** em nenhuma chave: tratado como cadastro novo — sobe direto pro OPS pelo fluxo da seção 5.
5. Cooperativas: mapear primeiro o representante (`EntidadeContratual`) e só depois os membros, para não cadastrar um membro órfão.

Esse item é de **migração de dados, não é 48h de hackathon** — cabe como próximo passo de roadmap pós-demo, na mesma lógica de honestidade que já funcionou para o portal do cliente: mostrar que o time pensou no que vem depois é parte do critério de viabilidade de negócio, não só código rodando no sábado.

## 7. Pendências que seguem em aberto

- **Número de conflitos em 90 dias**: não tenho acesso às bases (BTime/HubSpot/OPS) para rodar essa consulta a partir daqui — precisa de quem tem acesso aos dados rodar e trazer o número.
- **Dono do cadastro pós-hackathon**: este desenho assume e reforça a proposta já feita (OPS mestre / Serviços na operação / Hermes no código) — se isso mudar, os pontos 1 e 3 acima (fonte de verdade, adapters) mudam junto.
