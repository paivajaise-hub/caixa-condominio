Leia docs/ROTEIRO.md, docs/FORMA-DE-TRABALHO.md e docs/PERGUNTAS.md antes de qualquer coisa.

Sessão D1 — itens F1.1 a F1.4 do roteiro. Antes de executar, repita em três linhas o que vai fazer e o critério de aceite de cada item, e espere minha confirmação.

Contexto que você precisa saber:
- Projeto novo, nesta pasta. Next.js (App Router, TypeScript), Supabase, Vercel. Repositório público no GitHub.
- O Supabase e a Vercel já estão criados por mim. As variáveis estão em .env.local (não leia nem imprima o conteúdo; use só os nomes). Crie .env.example com os nomes.
- Migrações em supabase/migrations, aplicadas por mim com o CLI. Você escreve o SQL; eu reviso e aplico. Depois de eu aplicar, você pode consultar o banco para verificar, só leitura.
- Toda migração termina concedendo permissões explícitas (grant) ao que authenticated precisa. Views com security_invoker.
- Seed em supabase/seed.sql, totalmente fictício, conforme F1.3. Invente nomes brasileiros comuns; nenhum nome, chave Pix ou valor vem de mim.
- Não decida nada de negócio. Se o roteiro não cobrir, registre em docs/PERGUNTAS.md e me pergunte.

Ordem: F1.1 (estrutura do projeto, docs/ já dentro, primeiro commit, deploy na Vercel) → F1.2 (migração do esquema completo do §3 e §4 do roteiro) → F1.3 (seed) → F1.4 (auth com dois papéis via RLS e tela de login).

Ao terminar, ou se parar antes: atualize a seção ESTADO do roteiro com o que foi verificado por mim no ar, e escreva o PDCA da sessão no formato de docs/FORMA-DE-TRABALHO.md.
