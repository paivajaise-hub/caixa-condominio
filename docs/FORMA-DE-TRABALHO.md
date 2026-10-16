# FORMA DE TRABALHO — como este projeto é construído com IA

Dois papéis. **Planejadora** (Claude no app): lê o roteiro, escreve o prompt da sessão, revisa o resultado, registra decisões e atritos. **Executor** (Claude Code no repositório): um passo do roteiro por sessão, começa lendo `docs/ROTEIRO.md` e termina atualizando a seção ESTADO. Jaíse dirige os dois e é a única que decide o que entra.

## Regras que a IA não contorna
1. A IA só **lê** o banco. Toda escrita de esquema vira migração em `supabase/migrations`, revisada por Jaíse antes de aplicar. Dados: via seed versionado.
2. Segredos nunca passam pelo chat nem pelo repositório. `.env.local` fica fora do git; `.env.example` documenta as variáveis.
3. "Funciona" = teste passando (onde houver teste) **e** Jaíse viu no endereço publicado. "Deve funcionar" não existe.
4. Gasto com API ou serviço pago: só com autorização prévia e valor previsto.
5. Toda migração concede permissões explicitamente (`grant` para `authenticated`/`anon` no que precisar). Tabela nova sem grant = bug.
6. Acesso é RLS. Nenhum `if (user.role === ...)` decide o que alguém vê.
7. Escopo: o que está em "Fora de escopo" no roteiro não é reaberto em sessão. Para reabrir, muda o roteiro primeiro.

## Critério de decisão técnica delegada
Parcimônia, eficiência, efetividade, eficácia. A IA decide o técnico sob esse critério e justifica em uma linha no roteiro (§8). Decisões de negócio e irreversíveis (apagar dados, mudar modelo, gastar dinheiro) são de Jaíse.

## Formato de sessão
- **Abertura**: executor lê `docs/ROTEIRO.md` e `docs/FORMA-DE-TRABALHO.md`, repete em três linhas o que vai fazer e o critério de aceite, e espera confirmação.
- **Execução**: branch por funcionalidade (`feat/f1-2-esquema`), commits por assunto, PR para `main`. Nada direto em `main`.
- **Fechamento (PDCA)**, no fim de toda sessão longa, anexado ao roteiro em §9 ou §10:
  - Planejado: …
  - Saiu: …
  - Atritos (até três, de cada lado): …
  - Uma única mudança para a próxima sessão: …

## Dúvidas entre agentes
Perguntas do executor que a planejadora precisa responder vão para `docs/PERGUNTAS.md` (data, pergunta, resposta, decisão resultante). Nada fica só na memória de uma conversa.
