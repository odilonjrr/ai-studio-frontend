# JK IA Foto Studio — Contexto do Projeto

> Cole este conteúdo no `CLAUDE.md` gerado pelo `/init` (ou substitua o arquivo
> inteiro por este). Ele dá ao Claude Code o contexto que normalmente eu
> teria que reexplicar em cada conversa.

## O que é o projeto

SaaS de ensaios fotográficos gerados por IA ("JK IA Foto Studio"), voltado ao
mercado brasileiro. Cliente envia uma foto, escolhe um tema, paga, e recebe um
ensaio fotográfico com o rosto preservado. Site em português do Brasil.

- **Frontend:** `index.html` — **arquivo único**, sem build step. HTML + CSS +
  JS tudo num arquivo só. Hospedado na Vercel.
  - URL produção: https://iafotostudio.vercel.app
  - Repo: `odilonjrr/ai-studio-frontend` (GitHub)
  - Local: `C:\Users\jr_od\ai-studio-frontend`
- **Backend:** Node.js separado, cuida da integração com MercadoPago.
  - Local: `C:\Users\jr_od\ai-studio-backend\backend_mercadopago.js`
  - URL produção: ai-studio-backend-five.vercel.app
- **Banco de dados / Auth / Storage:** Supabase
  - Project URL: `https://heebcqoduhvbwpkjwmtf.supabase.co`
  - Chave publicável: `sb_publishable_BCqrqQkvBnqzJFt85fdWpw_UqkBSIU8`
  - Tabelas: `pedidos`, `clientes`, `perfis`
  - Buckets de Storage: `amostras` (fotos com marca d'água), `fotos-prontas`
    (fotos finais sem marca d'água, organizadas por pasta = id do pedido)
- **Pagamento:** MercadoPago (modo de teste ainda em uso — verificar antes de
  ir para produção)
- **WhatsApp de contato:** 5521999850085

## ⚠️ A armadilha mais comum deste projeto: a tag `</script>`

O `index.html` tem **exatamente 2 blocos `<script>`**:

1. Um no `<head>` — contém funções chamadas via `onclick` inline no HTML, e
   por isso **precisa carregar antes do parse do body**. Várias funções aqui
   são expostas explicitamente via `window.nomeDaFuncao = nomeDaFuncao;` para
   garantir que os `onclick` as encontrem mesmo com qualquer particularidade
   de escopo.
2. Um antes do `</body>` — funções que só fazem sentido depois que o DOM
   existe.

**O bug mais recorrente em todo o histórico deste projeto** é uma tag
`</script>` (ou `<script>`) ficando mal posicionada no meio do código,
fechando um bloco antes da hora. Isso faz o JavaScript aparecer como **texto
solto na tela** (porque o navegador para de interpretar como script e passa a
renderizar como HTML), ou corta funções inteiras do meio.

**Sempre que for editar este arquivo, antes de considerar a tarefa concluída:**

```bash
python3 -c "
import re, subprocess
with open('index.html', 'r', encoding='utf-8') as f:
    content = f.read()
opens = len(re.findall(r'<script>', content))
closes = len(re.findall(r'</script>', content))
print(f'Aberturas: {opens}, Fechamentos: {closes}')  # devem ser IGUAIS (normalmente 2/2)
scripts = re.findall(r'<script>(.*?)</script>', content, re.DOTALL)
for i, s in enumerate(scripts):
    with open(f'/tmp/c{i}.js','w') as f: f.write(s)
    r = subprocess.run(['node','--check',f'/tmp/c{i}.js'], capture_output=True, text=True)
    print(f'Script {i}:', 'OK' if r.returncode == 0 else r.stdout)
"
```

Também vale checar duplicação de funções entre os dois blocos (uma função
definida duas vezes silenciosamente sobrescreve a primeira, e isso já causou
bugs difíceis de notar):

```bash
python3 -c "
import re
from collections import Counter
with open('index.html', 'r', encoding='utf-8') as f:
    content = f.read()
scripts = re.findall(r'<script>(.*?)</script>', content, re.DOTALL)
all_defs = []
for s in scripts:
    all_defs += re.findall(r'function\s+([a-zA-Z_]\w*)', s)
dups = [f for f, c in Counter(all_defs).items() if c > 1]
print('Duplicadas:', dups if dups else 'nenhuma')
"
```

## Convenções deste projeto

- Sem framework, sem bundler. É só abrir o `index.html` e editar direto.
- Funções chamadas por `onclick` inline no HTML **precisam** estar expostas em
  `window.nomeDaFuncao` quando definidas dentro de blocos que podem ter
  problema de escopo (isso já resolveu bugs de "botão não faz nada").
- `showToast(mensagem)` é o padrão para feedback ao usuário — **mas atenção
  ao z-index**: o toast já teve bug de ficar atrás de modais porque tinha
  z-index menor. Modais usam z-index 1000+; toast precisa ficar acima disso
  (atualmente 9999).
- Chamadas ao Supabase são `fetch` direto pra API REST (`/rest/v1/...`) e
  Storage (`/storage/v1/...`), sem usar a lib oficial `@supabase/supabase-js`
  (não tem build step pra importar pacotes npm).
- RLS (Row Level Security) no Supabase: as políticas já causaram erros 500 em
  loop quando ficavam complexas demais (ex: `auth.uid() = id` em cascata).
  Preferir políticas simples (`USING (true)`) quando o controle de acesso real
  já é feito client-side ou não é crítico.

## Fluxo de deploy

```powershell
cd C:\Users\jr_od\ai-studio-frontend
git pull origin main
# editar index.html
git add index.html
git commit -m "descrição da mudança"
git push
```

A Vercel faz redeploy automático a cada push na branch `main`. Considerar
~1-2 minutos pra propagar, e pedir para recarregar com **Ctrl+Shift+R** pra
evitar cache do navegador mostrando a versão antiga.

## Sistema de autenticação (Supabase Auth)

Dois tipos de usuário, mesma tabela `perfis` com campo `role`:

- **Admin** (você): `odilon.cg@gmail.com`. Único admin do sistema. Acesso ao
  painel `/admin` só aparece no menu depois de logar — por padrão fica
  `display:none`.
- **Cliente**: cadastro próprio (nome, email, WhatsApp, senha mínima 6
  caracteres) via tela de login/cadastro. Painel mostra status do pedido,
  preview com marca d'água, e download das fotos finais quando liberadas.

Pontos sensíveis:
- O Supabase Auth retém emails mesmo depois de deletados da tabela
  `auth.users` em alguns casos — gerando erro "User already registered" ao
  tentar recriar a conta. O cadastro já trata isso tentando login automático
  quando recebe esse erro específico.
- Cadastro de cliente grava em **duas tabelas**: `perfis` (controle de
  role/auth) e `clientes` (pra aparecer na aba Clientes do admin), usando
  `Prefer: resolution=merge-duplicates` pra evitar erro 409 de duplicata.
- Sessão é mantida via `localStorage` (token + dados do usuário) pra
  sobreviver a reload de página sem precisar logar de novo.

## Pendências conhecidas (verificar status atual com o usuário)

- [ ] Download de fotos prontas pelo cliente — fluxo de upload do admin pra
  o bucket `fotos-prontas` foi implementado, mas o último teste reportado
  ainda não confirmava o download funcionando ponta a ponta. Vale conferir:
  se o `pedidoId` usado no upload é exatamente o mesmo usado na listagem, e
  se a política RLS de Storage cobre a operação de `list` (não só `select`).
- [ ] MercadoPago em modo de produção (ainda em modo teste).

## Preços atuais dos pacotes

| Fotos | Preço |
|-------|-------|
| 2     | R$17,00 |
| 5     | R$37,00 |
| 10    | R$57,00 (destaque "Mais Vendido") |
| 20    | R$97,00 |

Prazo de entrega exibido: "até 24 horas".