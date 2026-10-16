# ROTEIRO — Caixa do Condomínio

> Fonte única da verdade sobre o estado do projeto. Toda sessão de Claude Code começa lendo este arquivo e termina atualizando a seção ESTADO.
> Nome provisório: **Caixa do Condomínio**. Repositório público. Condomínio e pessoas são fictícios; o modelo de negócio é real (cinco casas, administração informal).

## 1. Objetivo

Colocar no ar, até **19/10/2026**, um app que substitua a planilha de controle do caixa de um condomínio pequeno, mostrando o método de trabalho com IA em uso (este roteiro, FORMA-DE-TRABALHO.md, decisões e atritos registrados, PDCA por sessão). O link entra no formulário da vaga Desenvolvedor Vibe Coding (ibe.IA) no dia 20/10.

Stack: Next.js (App Router) + TypeScript, Supabase (Postgres, Auth, Storage, RLS), Vercel.

## 2. O problema (real, nomes fictícios)

Condomínio de 5 casas (101 a 105), sem síndico profissional. Uma moradora (casa 105) administra o caixa voluntariamente. O dinheiro circula pela conta pessoal dela, misturado a transferências que nada têm a ver com taxa. A taxa é mensal (R$ 50,00 até mai/2025; R$ 60,00 a partir de jun/2025). Alguns moradores pagam serviços do condomínio do próprio bolso (cerca elétrica, portão, dedetização) e ficam com crédito que vai consumindo a taxa. Moradores entram e saem; meses de casa vazia não são cobrados. O controle hoje é uma planilha com uma aba de lançamentos e abas de cálculo; a prestação de contas é um recorte mensal dela. Objetivo declarado: um contador consegue revisar sem perguntar nada.

## 3. Modelo de domínio (espelha a planilha v3)

### Entidades
- **condominio** — id, nome, chave_pix (texto, configurável pela administradora, nunca commitada), nome_recebedor, cidade. Uma linha. Existe para que o app não precise ser reescrito se um dia virar multi-condomínio.
- **unidade** — id, condominio_id, numero (101–105), observacao.
- **responsavel** — id, unidade_id, nome, nome_no_pix, user_id (nullable; vincula ao login), deve_desde (date, 1º dia do mês), deve_ate (date, nullable = em aberto). Uma unidade pode ter vários responsáveis em períodos diferentes; períodos sem responsável = casa vazia, sem cobrança.
- **taxa** — id, condominio_id, valor, vigente_desde (date). Histórico; a taxa de um mês é a última com vigente_desde <= mês.
- **lancamento** — id, condominio_id, data, tipo (enum: receita | despesa | servico_pago_por_morador | reembolso_a_morador), unidade_id (nullable; obrigatório para receita de taxa, servico_pago_por_morador e reembolso), descricao, categoria (enum abaixo), valor (numeric 12,2 > 0), mes_ref (date, 1º dia do mês), fonte (enum abaixo), observacao, comprovante_path (Storage, nullable), criado_por (user_id), criado_em, atualizado_por, atualizado_em. Lançamentos não são apagados: campo `anulado_em` + `motivo_anulacao` (trilha de auditoria).
- **pendencia** — id, condominio_id, data, unidade_id (nullable), item, valor (nullable), o_que_confirmar, resolvido (bool), resolvido_em, resolucao.

### Enums
- categoria: taxa_mensal | saldo_inicial | rendimento | energia_area_comum | jardinagem_e_poda | limpeza_e_conservacao | portoes_cerca_e_iluminacao | dedetizacao | outros
- fonte: extrato_bancario | comprovante | prestacao_de_contas | informado_pela_administradora | contribuicao_da_administradora

### Regras de negócio (copiadas do LEIA-ME da planilha)
1. Tudo é lançamento. Quadros, resumo e prestação são **calculados** (views SQL), nunca digitados.
2. Os quatro tipos:
   - **receita** — Pix de taxa recebido de uma casa (unidade preenchida) ou outra entrada (rendimento, saldo inicial). Aumenta o caixa.
   - **despesa** — gasto do condomínio pago com dinheiro do caixa. Reduz o caixa.
   - **servico_pago_por_morador** — gasto do condomínio pago do bolso de um morador. É despesa do condomínio E crédito para a casa dele. O caixa não se mexe.
   - **reembolso_a_morador** — dinheiro do caixa devolvido a um morador para quitar crédito. Reduz o caixa e reduz o crédito.
3. mes_ref = 1º dia do mês a que a taxa se refere (para despesas, o mês em que entra na prestação).
4. A casa da administradora não transfere dinheiro; a taxa dela é lançada todo mês como receita (fonte contribuicao_da_administradora) para o quadro por casa ser comparável.
5. O caixa é contábil: o "saldo do caixa" é quanto do dinheiro na conta pessoal pertence ao condomínio.
6. Casa vazia = período sem responsável ativo = devido 0 naquele mês.
7. Situação da casa no mês M: saldo_acumulado = Σ devido(≤M) − Σ receitas de taxa(≤M) − Σ serviços pagos(≤M) + Σ reembolsos(≤M). Positivo = deve; negativo = crédito; |x| ≤ 0,50 = em dia.

### Views (SQL, com security_invoker)
- **v_quadro_por_casa** — por (mês, unidade): devido, pago, saldo_acumulado, situacao.
- **v_resumo_mensal** — por mês: taxas_recebidas, outras_receitas, despesas_pagas_pelo_caixa, reembolsos_pagos, servicos_pagos_por_moradores, resultado_do_mes, saldo_do_caixa (acumulado).
- **v_prestacao_mes** — tudo que a aba PRESTAÇÃO DO MÊS mostra, para um mês: saldo inicial, receitas por casa com situação, despesas por categoria (pago pelo caixa / pago por morador), reembolsos, total de saídas, resultado, saldo final.

## 4. Papéis e acesso
- **administradora** — vê e lança tudo; configura chave Pix, taxa, unidades, responsáveis; resolve pendências.
- **responsavel** — vê só a própria unidade: devido/pago/saldo por mês, comprovantes das despesas do condomínio (leitura), prestação de contas do mês, Pix copia-e-cola e QR com o valor da taxa do mês e identificador "casa-mês".
- Papel guardado em `perfil(user_id, condominio_id, papel, unidade_id)`; **toda** regra de acesso é RLS no banco. A aplicação não decide acesso.
- Primeiro acesso do responsável: convite por e-mail (Supabase Auth), troca de senha obrigatória.

## 5. Pix
Sem provedor de pagamento. O app gera o **BR Code estático** (EMV "Pix copia e cola") com a chave do condomínio, o valor da taxa do mês e um txid "U101-2026-10", e mostra o QR gerado no cliente. A administradora continua conferindo o extrato e lançando a receita à mão (ou por importação de CSV, Fase 2). Isto é decisão, não limitação: receber Pix dentro do app exige contrato com provedor e CNPJ do condomínio, que uma administração informal não tem.

## 6. Fases e critérios de aceite

"Pronto" = teste passando onde houver teste, e verificado por mim no endereço publicado na Vercel.

### Fase 1 — no ar até 19/10 (obrigatória)
- [ ] F1.1 Repositório público no GitHub com este `docs/` desde o primeiro commit. Deploy na Vercel ligado à branch `main`.
- [ ] F1.2 Esquema Supabase: tabelas, enums, views, RLS, políticas de Storage (bucket `comprovantes`, privado, leitura por policy). Migrações versionadas em `supabase/migrations`. **Toda migração concede permissões explicitamente** (grant à role `authenticated` nas tabelas/views necessárias).
- [ ] F1.3 Seed fictício: 5 casas, responsáveis com períodos (uma casa com troca de morador e dois meses vazios; uma com crédito por cerca elétrica; uma inadimplente), taxa 50→60 em jun/25, lançamentos de mar/24 a set/26, 4 pendências de "Pix de valor baixo". Nenhum nome, chave Pix ou valor real.
- [ ] F1.4 Auth com dois papéis via RLS; usuário administradora e dois responsáveis de teste no seed (credenciais no README para o avaliador, só do ambiente fictício).
- [ ] F1.5 Administradora: lista e cadastro de lançamentos com os 4 tipos, upload de comprovante, anulação com motivo; cadastro de unidades, responsáveis (períodos) e taxa.
- [ ] F1.6 Administradora: quadro por casa (mês × casa, cores: em dia / deve / crédito) e resumo mensal com saldo do caixa.
- [ ] F1.7 Administradora: prestação de contas do mês (mesmo layout da aba PRESTAÇÃO DO MÊS), com link para os lançamentos do mês.
- [ ] F1.8 Responsável: painel da própria unidade (devido/pago/saldo por mês), Pix copia-e-cola + QR do mês, comprovantes das despesas do condomínio, prestação do mês.
- [ ] F1.9 Pendências: lista, criar, resolver (com texto da resolução).
- [ ] F1.10 Teste automatizado das regras de cálculo (pelo menos: situação por casa com crédito, com vacância e com mudança de taxa) rodando contra o banco local ou o esquema.
- [ ] F1.11 README: o que é, como rodar, credenciais de teste, link para docs/. Sem segredo no repositório (`.env.example` apenas).

### Fase 2 — só depois do 20/10
- [ ] Importar CSV de extrato bancário e sugerir lançamentos (casa por nome no Pix) → cria pendência quando não bate.
- [ ] Prestação de contas em PDF.
- [ ] Página de informações: regras internas, calendário, o que diz a lei (Código Civil arts. 1.331–1.358; art. 1.348, VIII: dever de prestar contas).
- [ ] Multi-condomínio real (hoje condominio_id existe, mas há um só).

### Fora de escopo, por escrito
- Receber Pix dentro do app (provedor, webhook, conciliação automática).
- Assembleia, votação, chat, multas, boletos, reservas de área comum.
- App nativo. É web responsiva.

## 7. Plano por dia (16 a 20/10)
- **16/10 (D1)** — F1.1, F1.2, F1.3, F1.4. Fim do dia: login funcionando no endereço da Vercel, banco com seed.
- **17/10 (D2)** — F1.5, F1.6.
- **18/10 (D3)** — F1.7, F1.8, F1.9.
- **19/10 (D4)** — F1.10, F1.11, revisão no celular, PDCA final, atualizar este roteiro.
- **20/10 (D5)** — enviar formulário. Nada de código novo.

## 8. Decisões (uma linha de justificativa cada)
- D-01 (15/10) Views SQL em vez de cálculo no front: a planilha já é "uma aba de dados + abas de fórmula"; o banco faz o mesmo papel e um contador pode consultar direto. Parcimônia.
- D-02 (15/10) Lançamento nunca é apagado, só anulado com motivo: trilha de auditoria é o que dá confiança a quem revisa. Efetividade.
- D-03 (15/10) Pix por BR Code estático, sem provedor: ver §5. Parcimônia.
- D-04 (15/10) condominio_id em tudo desde o início, mesmo com um só condomínio: custa uma coluna agora e evita reescrever RLS depois. Eficiência.
- D-05 (15/10) Papel e acesso só por RLS, nunca por `if` na aplicação: regra já usada nas duas plataformas anteriores. Eficácia.

## 9. Atritos (registrar quando acontecer)
_(formato: data — o que travou — o que foi feito — regra nova, se houver)_

## 10. ESTADO
Atualizado em: 15/10/2026, antes da primeira sessão.
- Fase atual: **Fase 1, D1 não iniciado.**
- Última entrega verificada no ar: nenhuma.
- Próximo passo: prompt `prompts/01-setup.md` no Claude Code.
- Bloqueios: nenhum.
