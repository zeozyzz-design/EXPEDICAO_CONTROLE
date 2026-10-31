# Painel de Controle de Expedição — Iveco Group

Controle das solicitações de emissão entre CEVA, Celere e transportadoras:
fila em tempo real, tempo medido por etapa, ocorrências com responsável e
histórico consultável.

**Stack:** Next.js 15 · React 19 · TypeScript · Tailwind · Supabase
(Postgres, Auth, Realtime, Storage).

---

## Por que a medição é confiável

| Garantia | Como é feita |
| --- | --- |
| A hora não pode ser forjada | Todo evento é gravado por função no banco com `now()` do Postgres. O navegador nunca envia data nem hora. |
| Checkpoint não pode ser editado | `UPDATE` e `DELETE` em `eventos` disparam exceção. Uma correção é um evento de estorno; o original fica. |
| Alteração no banco fica visível | Cada evento guarda o hash SHA-256 do anterior. `verificar_integridade()` recalcula a cadeia e aponta o que não confere. |
| Transportadora não vê a concorrente | Row Level Security no Postgres filtra por transportadora. Não é filtro de tela: a linha não sai do banco. |
| Permissão por etapa | A função `registrar_checkpoint` recusa quem não é dono da etapa. Só a Celere registra em nome de terceiro, e isso fica marcado como `por_master`. |

---

## Implantação, do zero ao ar

### 1. Criar o projeto no Supabase

1. Entre em [supabase.com](https://supabase.com) e crie um projeto.
   Região **South America (São Paulo)** deixa o painel mais rápido no Brasil.
2. Guarde a senha do banco que ele pedir.
3. Espere o provisionamento terminar (cerca de dois minutos).

### 2. Rodar o schema

1. No projeto, abra **SQL Editor → New query**.
2. Cole o conteúdo inteiro de `supabase/schema.sql` e clique em **Run**.
3. Deve terminar com `Success`. Isso cria tabelas, políticas de acesso,
   funções, triggers de imutabilidade, o bucket de anexos e liga o realtime.

### 3. Pegar as chaves

Em **Project Settings → API**, copie:

| Campo no Supabase | Variável |
| --- | --- |
| Project URL | `NEXT_PUBLIC_SUPABASE_URL` |
| `anon` `public` | `NEXT_PUBLIC_SUPABASE_ANON_KEY` |
| `service_role` `secret` | `SUPABASE_SERVICE_ROLE_KEY` |

> A chave `service_role` ignora todas as regras de acesso. Ela só existe no
> servidor, para criar logins. Nunca a coloque em variável com prefixo
> `NEXT_PUBLIC`, nunca a mande por e-mail ou WhatsApp.

### 4. Desligar o cadastro aberto

Em **Authentication → Providers → Email**, desligue **Enable email signups**.
Assim ninguém cria conta sozinho: só a Celere cria logins, pelo painel.

### 5. Publicar na Vercel

1. Suba esta pasta para um repositório no GitHub.
2. Em [vercel.com](https://vercel.com), **Add New → Project** e escolha o repositório.
3. Em **Environment Variables**, cadastre as três variáveis do passo 3.
4. **Deploy**. Em cerca de dois minutos o endereço está no ar.

### 6. Primeiro acesso

1. Abra o endereço. Com a base vazia, a tela de login oferece
   **Primeiro acesso**.
2. Crie o acesso master da Celere: e-mail, nome e senha.
3. Entre e vá em **Usuários** para criar os logins da CEVA, de cada
   transportadora e da gerência da Iveco.
4. Em **Indicadores**, ajuste as metas de SLA. Os valores iniciais
   (20 minutos para a emissão, 60 para o recebimento) são ponto de partida —
   corrija com os dados reais depois das primeiras semanas.

---

## Rodar na sua máquina

```bash
npm install
cp .env.example .env.local   # preencha com as chaves do passo 3
npm run dev                  # http://localhost:3000
```

---

## O fluxo

| Checkpoint | Etapa | Quem registra |
| --- | --- | --- |
| CP1 | Romaneio fechado e enviado | CEVA |
| CP2 | Emissão do NOTFIS | Celere |
| CP3 | Recebimento do NOTFIS | Transportadora |
| CP4 | Conferência e emissão do CTe | Transportadora |
| CP5 | Início de carregamento | Transportadora |
| CP6 | Fim de descarregamento | Transportadora |

Quando a emissão trava no CP2, a Celere abre uma ocorrência pelo botão
**A emissão parou**, escolhe o motivo e aponta o responsável entre CEVA,
Celere, transportadora e Inovatech. O relógio da ocorrência corre até alguém
fechá-la, e essas horas aparecem separadas por empresa nos indicadores.

## Perfis

| Perfil | O que faz | O que vê |
| --- | --- | --- |
| **Celere** | Registra qualquer etapa, abre e fecha ocorrências, cria logins, define metas | Tudo |
| **CEVA** | Lança romaneios e acompanha | Fila e histórico completos |
| **Transportadora** | Registra CP3 a CP6 das cargas dela | Só os romaneios da própria transportadora |
| **Gerência (Iveco)** | Nada | Tudo, somente leitura |

---

## Manutenção

**Backup.** O Supabase faz backup diário automático no plano pago. No plano
gratuito, exporte periodicamente: **Database → Backups**, ou pelo histórico
em CSV na tela de Histórico.

**Nova etapa no fluxo.** Insira a linha em `etapas` e ajuste `ETAPA_FINAL`
em `src/lib/domain.ts`. As telas leem o catálogo do banco.

**Nova transportadora.** Acrescente em `TRANSPORTADORAS`, no mesmo arquivo.

**Novo tipo de ocorrência.** `insert into tipos_ocorrencia (...)`. A tela lê
a tabela; nada no código muda.

**Conferir a integridade.** A aba Auditoria tem o botão
**Verificar integridade**. Ele roda `verificar_integridade()` no banco e lista
qualquer evento que não bata com a cadeia.

---

## O que ainda não existe

- **Notificação fora da tela.** O alerta toca para quem está com o painel
  aberto. E-mail ou WhatsApp quando estoura o SLA pede uma Edge Function do
  Supabase com cron — é o próximo passo natural.
- **Relatório agendado.** O CSV é exportado sob demanda, não enviado sozinho.
- **Anexo lido pelo sistema.** A planilha da CEVA é guardada para download;
  as notas dela não são importadas.
