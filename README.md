# 🛍️ Usados de Qualidade

Site estático de vendas criado como projeto de estudo em testes automatizados, CI/CD com GitHub Actions e deploy no GitHub Pages.

<a href="https://santosqa.github.io/pgats-test-ci-cd/" target="_blank" rel="noopener noreferrer">
  <img src="https://img.shields.io/badge/GitHub-Acessar%20O%20Projeto-%2350fa7b?style=for-the-badge&logo=github&logoColor=white" alt="GitHub">
</a>

---

## Pré-requisitos

| Ferramenta | Versão mínima |
|---|---|
| Node.js | 24 |
| npm | (incluso no Node) |
| Python | 3 |
| Playwright | (instalado via npm) |

---

## Configuração local

```bash
# 1. Clone e entre na pasta
git clone https://github.com/santosqa/pgats-test-ci-cd.git
cd pgats-test-ci-cd

# 2. Instale as dependências Node
npm install

# 3. Instale o navegador usado nos testes funcionais
npx playwright install chromium

# 4. Gere o catálogo de produtos (necessário antes de rodar testes funcionais ou o site)
npm run build:catalog

# 5. Suba o site localmente
npm run serve -- --bind 127.0.0.1
# Acesse: http://127.0.0.1:8000
```

---

## Executar testes localmente

Valide tudo localmente antes de abrir um PR ou fazer push.

### Testes unitários

```bash
npm run test:unit:ci -- --runInBand
```

Relatórios gerados:

| Arquivo / Pasta | Conteúdo |
|---|---|
| `coverage/` | Cobertura em HTML, LCOV e JSON |
| `coverage/index.html` | Relatório HTML navegável |
| `coverage/coverage-summary.json` | Resumo de cobertura |
| `reports/unit-results.json` | Resultado dos testes em JSON |
| `junit.xml` | Resultado JUnit (consumido pela esteira) |

### Testes funcionais

O comando gera o catálogo antes de iniciar os testes.

```bash
npm run test:functional
```

Relatórios gerados:

| Arquivo / Pasta | Conteúdo |
|---|---|
| `reports/functional-results.json` | Resultado funcional em JSON |
| `reports/functional-junit.xml` | Resultado funcional em JUnit |
| `playwright-report/` | Relatório HTML do Playwright |
| `test-results/` | Evidências, traces e anexos de falha |

```bash
# Abrir o relatório HTML do Playwright no navegador
npx playwright show-report
```

### Todos os testes (unitários + funcionais)

```bash
npm run test:ci
```

---

## Referência rápida de comandos

```bash
npm install                              # instala dependências
npx playwright install chromium          # instala navegador para testes funcionais
npm run build:catalog                    # gera catalog.json
npm run test:unit:ci -- --runInBand      # testes unitários com cobertura e JUnit
npm run test:functional                  # testes funcionais com Playwright
npm run test:ci                          # todos os testes
npm run serve -- --bind 127.0.0.1        # sobe o site em http://127.0.0.1:8000
```

---

## Estrutura do projeto

```text
.
├── .github/workflows/
│   ├── ci-testes-unitarios.yml     # CI: testes unitários a cada push/PR
│   ├── testes-automatizados.yml    # Fluxo reutilizável: testes unitários + E2E
│   ├── schedule-testes.yml         # Agendamento: chama testes-automatizados.yml
│   ├── execucao-manual.yml         # Disparo manual com inputs
│   └── deploy-pages.yml            # Deploy no GitHub Pages
├── products/                       # Dados dos produtos
├── scripts/
│   ├── build_catalog.py            # Gera catalog.json
│   └── ci_test_summary.js          # Gera Step Summary na esteira
├── test/
│   ├── app.test.js                 # Testes unitários (Jest)
│   └── functional/catalog.spec.js  # Testes funcionais (Playwright)
├── app.js
├── index.html
├── jest.config.js
├── package.json
├── playwright.config.js
└── styles.css
```

---

## Esteira CI/CD

A esteira é composta por **5 workflows** com responsabilidades separadas.

```text
Push / PR para dev ou main
        │
        ▼
ci-testes-unitarios.yml  ──────────────────────────────▶  ✅ ou ❌  (feedback rápido)


Push para main / Manual / Schedule
        │
        ├──▶  schedule-testes.yml   ─┐
        ├──▶  execucao-manual.yml   ─┼──▶  testes-automatizados.yml  (workflow_call)
        └──▶  deploy-pages.yml      ─┘
                    │
                    ▼
              test ──▶ build ──▶ deploy ──▶ notify (e-mail)
```

---

### 1. `ci-testes-unitarios.yml` — CI: Testes Unitários

**Quando roda:** a cada `push` ou `pull_request` para `dev` e `main`.  
**Objetivo:** feedback rápido. Não roda testes E2E.

| Step | O que faz |
|---|---|
| Checkout | Clona o repositório |
| Setup Node.js 24 | Configura Node com cache do npm |
| `npm ci` | Instala dependências de forma determinística |
| `build:catalog` | Gera `catalog.json` |
| Testes unitários | Executa Jest com cobertura e gera `junit.xml` |
| Test Reporter | Publica relatório Jest na aba Checks |
| Upload artefatos | Salva `coverage/` e `junit.xml` por 30 dias |

---

### 2. `testes-automatizados.yml` — Fluxo Reutilizável (E2E)

**Quando roda:** chamado via `workflow_call` pelos fluxos de schedule, execução manual e deploy. Pode também ser disparado diretamente via `workflow_dispatch`.  
**Objetivo:** suite completa — unitários + funcionais — com relatórios, artefatos e comentário em PR.

**Por que `continue-on-error: true` nos steps de teste:**  
Sem essa flag, uma falha interrompe o job imediatamente, impedindo que os steps seguintes (relatórios, artefatos, e-mail) sejam executados. Com ela, o job continua mesmo após uma falha, e o resultado real é verificado no step final.

**Por que `if: always()` nos steps de relatório:**  
Garante que steps como upload de artefatos, Step Summary e comentário em PR rodem mesmo quando um step anterior foi marcado como falha.

| Step | O que faz |
|---|---|
| Checkout | Clona o repositório |
| Setup Node.js 24 | Configura Node com cache do npm |
| `npm ci` | Instala dependências |
| `build:catalog` | Gera `catalog.json` |
| Testes unitários (`continue-on-error`) | Jest com cobertura + `junit.xml` |
| Test Reporter Jest | Publica relatório na aba Checks |
| Instalar Playwright | Instala Chromium com dependências |
| Testes funcionais (`continue-on-error`) | Playwright em Chromium |
| Test Reporter Playwright | Publica relatório funcional na aba Checks |
| Step Summary | Tabela com totais de testes via `ci_test_summary.js` |
| Upload artefatos | `coverage/`, `junit.xml`, `reports/`, `playwright-report/`, `test-results/` por 30 dias |
| Comentário no PR | Resultado por tipo de teste + link para a execução |
| Resultado final | Falha o job se unitários **ou** funcionais falharam — garante que o status correto suba para o caller |

**Outputs expostos para o workflow chamador:**

| Output | Valor |
|---|---|
| `unit_outcome` | `success` ou `failure` |
| `functional_outcome` | `success` ou `failure` |
| `overall_result` | `success` ou `failure` |

---

### 3. `schedule-testes.yml` — Agendamento

**Quando roda:** agendamento via `cron` + `workflow_dispatch`.  
**Objetivo:** disparar a suite de testes sem duplicar steps — apenas chama `testes-automatizados.yml` via `workflow_call`.

**Cron exemplo:**

```yaml
schedule:
  - cron: '*/15 * * * *'
```

> Ajuste o cron conforme a necessidade do projeto. Exemplo para toda segunda-feira às 07h BRT (10h UTC): `0 10 * * 1`.

**Como ler uma expressão cron:**

```text
┌───── minuto   (0–59)
│ ┌─── hora UTC (0–23)
│ │ ┌─ dia do mês (1–31)
│ │ │ ┌ mês       (1–12)
│ │ │ │ ┌ dia da semana (0=Dom … 6=Sab)
│ │ │ │ │
* * * * *
```

> O GitHub usa UTC. São Paulo (BRT) é UTC-3, então `10 UTC = 07 BRT`.

---

### 4. `execucao-manual.yml` — Execução Manual

**Quando roda:** `workflow_dispatch` com inputs.  
**Objetivo:** permitir disparar testes, deploy ou ambos manualmente pelo GitHub Actions UI.

**Inputs disponíveis:**

| Input | Opções | Descrição |
|---|---|---|
| `acao` | `testes` / `deploy` / `testes+deploy` | O que executar |
| `debug` | `true` / `false` | Ativa `--verbose` nos testes |

**Lógica de execução:**

```text
acao = "testes"         →  chama testes-automatizados.yml
acao = "deploy"         →  dispara deploy-pages.yml diretamente
acao = "testes+deploy"  →  testa primeiro; se passar, dispara deploy-pages.yml
```

---

### 5. `deploy-pages.yml` — Deploy no GitHub Pages

**Quando roda:** `push` para `main` e `workflow_dispatch`.  
**Objetivo:** publicar o site. O deploy só ocorre se os testes passarem.

**Fluxo de jobs:**

```text
test  ──▶  build  ──▶  deploy  ──▶  notify
 │
 └── se falhar: build e deploy são skipped automaticamente
                notify ainda roda (if: always()) e informa por e-mail
```

#### Job `test`

Chama `testes-automatizados.yml` via `workflow_call`. Se qualquer teste falhar, os jobs seguintes são pulados automaticamente pelo GitHub Actions (comportamento padrão de `needs`).

#### Job `build`

Só executa se `needs.test.outputs.overall_result == 'success'`.

| Step | O que faz |
|---|---|
| Checkout + Node.js 24 + Python 3.12 | Prepara ambiente |
| `npm ci` | Instala dependências |
| `build_catalog.py` | Gera `catalog.json` |
| Prepara `site/` | Copia `index.html`, `styles.css`, `app.js`, `catalog.json`, `products/`, `logos/` |
| Configure Pages | Configura o artefato para o GitHub Pages |
| Upload artefato Pages | Envia `site/` para publicação |

#### Job `deploy`

Usa `actions/deploy-pages` para publicar o artefato. Expõe `page_url` como output para o job `notify`.

#### Job `notify` — E-mail único consolidado

Usa `if: always()` e depende de todos os jobs anteriores (`needs: [test, build, deploy]`). Envia **um único e-mail** com o resultado de cada etapa, independentemente do que passou ou falhou.

Conteúdo do e-mail:

```text
Status geral: PASSOU ✅ / FALHOU ❌

Repositório: ...
Execução: #N
Trigger: push / workflow_dispatch
Branch: main
Commit: abc1234

Resultado dos jobs:
  - Testes (unitários + funcionais): success / failure
  - Build do site: success / skipped / failure
  - Deploy GitHub Pages: success / skipped / failure

URL publicada: https://usuario.github.io/repo/
Relatório completo: https://github.com/.../actions/runs/...
```

---

## Relatórios e onde encontrá-los

| Canal | Como acessar |
|---|---|
| **Checks** | Repositório → commit ou PR → aba Checks |
| **Step Summary** | Repositório → Actions → execução → Summary |
| **Artefatos** | Repositório → Actions → execução → Artifacts |
| **Comentário em PR** | Pull Request correspondente |
| **E-mail** | Caixa configurada em `NOTIFICATION_EMAIL` |

**Artefatos gerados por execução (retidos 30 dias):**

```text
unit-test-report-<N>/       # gerado pelo ci-testes-unitarios.yml
  coverage/
  junit.xml

e2e-test-report-<N>/        # gerado pelo testes-automatizados.yml
  coverage/
  junit.xml
  reports/
  playwright-report/
  test-results/
```

---

## Configuração de secrets para e-mail

Acesse: `Settings → Secrets and variables → Actions → New repository secret`

| Secret | Obrigatório | Exemplo |
|---|---|---|
| `SMTP_HOST` | Sim | `smtp.gmail.com` |
| `SMTP_PORT` | Não (padrão 587) | `587` ou `465` |
| `SMTP_USERNAME` | Sim | seu e-mail no SMTP |
| `SMTP_PASSWORD` | Sim | senha ou app password |
| `NOTIFICATION_EMAIL` | Recomendado | e-mail de destino |
| `NOTIFICATION_FROM` | Não | remetente exibido |

> **`NOTIFICATION_EMAIL`**: se não configurado, a esteira tenta usar o e-mail público do perfil GitHub do dono do repositório. Se o e-mail não estiver público, o envio é ignorado com aviso no log.  
> **Gmail**: use uma [App Password](https://myaccount.google.com/apppasswords) em vez da senha da conta.

---

## Configuração do repositório

### Habilitar GitHub Pages

```
Settings → Pages → Source: "GitHub Actions"
```

### Permissões do Actions

```
Settings → Actions → General → Workflow permissions
→ ✅ Read and write permissions
→ ✅ Allow GitHub Actions to create and approve pull requests
```

---

## Conceitos utilizados

| Conceito | Descrição |
|---|---|
| `workflow_call` | Permite que um workflow seja chamado por outro, evitando duplicação de steps |
| `workflow_dispatch` | Gatilho de execução manual com inputs configuráveis |
| `schedule` / `cron` | Agendamento automático de execuções |
| `needs` | Define dependência entre jobs; jobs dependentes são skipped se o anterior falhar |
| `if: always()` | Força a execução do step mesmo após falhas anteriores |
| `continue-on-error` | Permite que o job continue mesmo quando um step falha, para garantir relatórios |
| `outputs` | Expõe valores de um job ou workflow para uso em jobs/workflows chamadores |
| `concurrency` | Cancela execuções duplicadas no mesmo branch |
| `permissions` | Define o nível mínimo de acesso necessário por workflow |
| `dorny/test-reporter` | Publica relatórios JUnit na aba Checks do GitHub |
| `GITHUB_STEP_SUMMARY` | Arquivo especial que gera o resumo visual na página da execução |
| `secrets` | Variáveis seguras para credenciais; injetadas via `Settings → Secrets` |

---

## Sobre o autor

🐞 Caçador de bugs, guardião da qualidade e parceiro do time: antecipo problemas e reforço a qualidade reduzindo dor de cabeça em produção.

Projeto mantido por **Ricardo Santos** — QA Engineer

Foco em: Qualidade de software · Automação de testes · Testes Web, API e Mobile · Observabilidade

🌐 [santosqa.github.io](https://santosqa.github.io) 👋🏼

---

[![AWS](https://img.shields.io/badge/AWS-Simulado%20CERTIFICAÇÃO%20CLF-%23FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://simuladoawsclf.pages.dev/) [![Santos Links](https://img.shields.io/badge/GitHub-Santos%20Links-%23FFD700?style=for-the-badge&logo=github&logoColor=white)](https://santosqa.github.io) [![GitHub](https://img.shields.io/badge/GitHub-About.me-%2350fa7b?style=for-the-badge&logo=github&logoColor=white)](https://github.com/santosqa) [![LinkedIn](https://img.shields.io/badge/LinkedIn-SantosQA-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/santosqa) [![Instagram](https://img.shields.io/badge/Instagram-%23ff5555.svg?&style=for-the-badge&logo=instagram&logoColor=white)](https://www.instagram.com/santosqa_/) [![Santos QA](https://img.shields.io/badge/www-santosqa.com-%23bd93f9.svg?&style=for-the-badge&logo=firefox-browser&logoColor=white)](https://www.santosqa.com)