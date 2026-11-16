# 📬 Email Triage IA

![Versão](https://img.shields.io/badge/vers%C3%A3o-1.0.0-blue?style=flat-square)
![Stack](https://img.shields.io/badge/stack-Apps%20Script%20%2B%20Groq-informational?style=flat-square)
![Licença](https://img.shields.io/badge/licen%C3%A7a-MIT-green?style=flat-square)

Triagem automática de e-mails de carreira com IA no Gmail: classifica, organiza e arquiva o ruído, resgata oportunidades do Spam e entrega um resumo diário — a custo zero.

**Repositório:** [github.com/soudevictor/email-triage](https://github.com/soudevictor/email-triage)

> Projeto pessoal, em uso real na minha própria caixa de entrada. Roda inteiramente na sua conta Google: nenhum dado passa por servidor de terceiros além do texto enviado à Groq para classificação. Não é afiliado ao Google nem à Groq.

---

**Índice:** [Visão geral](#visão-geral) · [Stack](#stack-tecnológica) · [Pré-requisitos](#pré-requisitos) · [Como instalar e rodar](#como-instalar-e-rodar) · [Arquitetura de pastas](#arquitetura-de-pastas) · [Personalização](#personalização) · [Qualidade](#qualidade) · [Limitações](#limitações-conhecidas) · [Problemas comuns](#problemas-comuns) · [Autor](#autor) · [Licença](#licença)

---

## Visão geral

Quem procura emprego ou freela vive o mesmo ciclo: a caixa de entrada acumula newsletters, promoções de curso e cold e-mails comerciais, enquanto as plataformas de recrutamento (Gupy, Greenhouse, Lever...) despejam dezenas de rejeições automáticas. No meio disso, um convite de entrevista pode passar batido — ou pior, cair no Spam sem nunca ser visto. Abrir a caixa de entrada todos os dias vira uma tarefa desgastante, e o risco é deixar uma oportunidade esfriar.

O Email Triage IA resolve isso com um agente autônomo que roda dentro do Gmail. Todos os dias, ele busca as conversas novas das últimas 48 horas (Caixa de Entrada, Promoções, Atualizações **e Spam**), envia remetente, assunto e até 2.000 caracteres do corpo para um LLM e recebe de volta uma classificação em JSON estrito: `OPORTUNIDADE`, `NEGATIVA_VAGA`, `MARKETING` ou `OUTROS`. Com essa categoria, o script age direto no Gmail — etiqueta, arquiva ou move a conversa — e encerra enviando **um único e-mail de resumo** com as oportunidades do dia.

O diferencial técnico é o custo e a simplicidade da operação: não há servidor, Docker, VPS ou orquestrador. Tudo roda no Google Apps Script (JavaScript V8), dentro das cotas gratuitas do Google, com a Groq no tier gratuito. O script ainda descobre sozinho qual modelo de texto está ativo na conta (consultando `GET /models`), o que o mantém funcionando quando a Groq renomeia ou aposenta modelos; trata o corpo do e-mail estritamente como dado, ignorando instruções embutidas (anti-prompt-injection); e usa travas de concorrência e de tempo para nunca estourar o limite de 6 minutos do Apps Script.

**Principais funcionalidades**

- Classificação por IA em 4 categorias, com saída estruturada em JSON e `temperature: 0` para consistência.
- Etiquetas automáticas (`IA/Oportunidade`, `IA/Negativa-Vaga`, `IA/Marketing`, `IA/Outros`) e etiqueta de controle `IA-Processado`, que evita reprocessar e gastar tokens.
- Limpeza da Inbox: `MARKETING` e `NEGATIVA_VAGA` são arquivados automaticamente, sem apagar nada.
- Resgate de oportunidades do Spam para a Caixa de Entrada, mesmo que o filtro antispam tenha barrado o convite.
- Digest diário único por e-mail, com empresa, resumo de uma linha e link direto para a conversa — enviado somente quando existe oportunidade.
- Segurança por padrão: chave de API em `PropertiesService`, nunca no código; falha em um e-mail não o marca como processado, então ele é reavaliado no dia seguinte.

---

## Stack Tecnológica

| Camada | Tecnologia | Uso no projeto |
| --- | --- | --- |
| Orquestração | Google Apps Script (JavaScript V8) | Executa o ciclo diário na nuvem do Google, sem servidor próprio |
| Agendamento | Time-driven trigger (`atHour(18)`) | Dispara `triarEmails` uma vez por dia, entre 18h e 19h |
| IA / LLM | Groq API (`/openai/v1/chat/completions`) | Classifica cada e-mail e devolve JSON estrito (`response_format: json_object`) |
| Integração com Gmail | `GmailApp` | Busca, etiqueta, arquiva e move conversas (`moveToInbox`, `moveToArchive`) |
| Configuração e segredos | `PropertiesService` | Guarda `GROQ_API_KEY` fora do código-fonte |
| Notificação | `MailApp` | Envia o digest diário em HTML e texto plano |
| Concorrência e limites | `LockService` + janela de 4min30s | Impede execução dupla e timeout de 6 min do Apps Script |
| Verificação | `node --check` + Registro de execuções | Não há suíte de testes automatizada |
| Deploy | Conta Google (serverless) | Nada para hospedar: o projeto vive no Apps Script |

Não há Node.js, npm ou build no fluxo principal — as dependências são zero por design. O Node aparece apenas como ferramenta opcional de verificação de sintaxe e para o `clasp`.

---

## Pré-requisitos

- **Conta Google** com Gmail (pessoal ou Google Workspace).
- **Conta gratuita na Groq** — [console.groq.com](https://console.groq.com). Não exige cartão de crédito.
- **Navegador atualizado** (o setup inteiro é feito em interfaces web).
- **Conhecimento técnico:** nenhum. O processo é copiar, colar e clicar.
- **Tempo estimado:** ~10 minutos.

> ⚠️ O projeto usa a sua própria conta. Nenhuma pessoa (nem o autor) tem acesso aos seus e-mails ou à sua chave.

---

## Como instalar e rodar

Diferente de uma aplicação web, este projeto **não tem servidor local**: a "instalação" é colar o código no Apps Script e configurar duas coisas — a chave da Groq e o fuso horário. Depois disso, a automação roda sozinha na nuvem.

Siga os passos na ordem. Cada um termina com um **checkpoint** para confirmar que está tudo certo antes de continuar.

### Passo 1 — Criar sua chave de API na Groq

A Groq é quem faz a "leitura inteligente" dos e-mails. É gratuita e responde em ~0,5 a 1 segundo por e-mail.

1. Acesse **[console.groq.com](https://console.groq.com)** e faça login (pode usar sua conta Google ou GitHub).
2. No menu lateral, clique em **API Keys**.
3. Clique em **Create API Key**.
4. Dê um nome qualquer, por exemplo `email-triage`.
5. Clique em **Submit** e **copie a chave** que aparece (começa com `gsk_...`).

> 🔒 **A chave aparece uma única vez.** Se fechar a janela, terá que criar outra. Guarde-a num gerenciador de senhas.
>
> ❗ Nunca cole essa chave em código, README, print de tela, issue ou repositório público. O Passo 5 mostra onde armazená-la com segurança.

**✅ Checkpoint:** você tem uma chave no formato `gsk_xxxxxxxxxxxxxxxxxxxx` guardada.

---

### Passo 2 — Criar o projeto no Google Apps Script

1. Acesse **[script.google.com](https://script.google.com/home)** com **a mesma conta Google do Gmail** que você quer organizar.
2. Clique em **Novo projeto** (canto superior esquerdo).
3. No topo, clique no nome **"Projeto sem título"** e renomeie para `Triagem de E-mails com IA`.
4. Clique em **Renomear** para confirmar.

**✅ Checkpoint:** você está dentro do editor, com um arquivo `Code.gs` aberto contendo `function myFunction() {}`.

---

### Passo 3 — Colar o código do projeto

1. Abra o arquivo **[`src/TriagemEmails.gs`](./src/TriagemEmails.gs)** aqui no GitHub.
2. Clique no botão **Raw** (canto superior direito do arquivo) para ver o texto puro.
3. Selecione tudo (`Ctrl+A` / `⌘+A`) e copie (`Ctrl+C` / `⌘+C`).
4. Volte ao Apps Script, clique dentro do editor e apague **todo** o conteúdo padrão (`function myFunction() {}`).
5. Cole o código copiado (`Ctrl+V` / `⌘+V`).
6. Salve com `Ctrl+S` (ou `⌘+S`).

> 💡 O arquivo no editor pode continuar se chamando `Code.gs` — isso **não** importa. O que importa é o conteúdo.

**✅ Checkpoint:** o editor mostra `const CONFIG = {...}` no topo e a função `triarEmails` na lista de funções, na barra superior.

---

### Passo 4 — Ajustar o fuso horário (não pule!)

O agendamento das 18h usa o fuso do projeto. Se ele estiver errado, o robô roda na hora errada (ou você acha que ele "não funcionou").

1. No menu lateral esquerdo, clique na engrenagem **⚙️ Configurações do projeto**.
2. Em **Fuso horário**, escolha o seu. No Brasil: **(GMT-03:00) Horário de Brasília – São Paulo**.
3. Confirme que **Mostrar o arquivo de manifesto `appsscript.json`** está marcado (opcional, mas recomendado).

**✅ Checkpoint:** o fuso horário exibido é o seu.

<details>
<summary><b>O que é o arquivo <code>appsscript.json</code> que aparece no projeto?</b></summary>

É o **manifesto** do projeto — o único arquivo de configuração do Apps Script (equivalente ao `package.json`). Ele não roda; ele declara como o projeto se comporta:

| Campo | Para que serve |
| --- | --- |
| `timeZone` | Fuso usado pelo `atHour(18)` e pelas datas. É o que faz o robô rodar às 18h **de Brasília**. |
| `runtimeVersion: V8` | Habilita o JavaScript moderno (`const`, arrow functions). |
| `oauthScopes` | Pré-declara as permissões pedidas ao usuário (Gmail, envio de e-mail, gatilhos, chamadas externas, e-mail da conta). |
| `exceptionLogging` | Envia erros com stack trace para o log de execuções. |
| `dependencies` | Bibliotecas externas (este projeto não usa nenhuma). |

No repositório ele está em [`src/appsscript.json`](./src/appsscript.json) e é usado principalmente por quem instala via `clasp` (veja [Instalação via clasp](#instalação-via-clasp-avançado)). Se você colou o código à mão, **não precisa dele**: o Apps Script descobre as permissões sozinho na primeira execução — só não esqueça de ajustar o fuso horário acima.

> ⚠️ Se você adicionar uma funcionalidade nova (por exemplo, acessar o Drive), inclua o escopo correspondente em `oauthScopes`. Com a lista explícita, ela é **fechada**: permissão não declarada gera o erro *"requires permission not granted"*.

</details>

---

### Passo 5 — Guardar a chave da Groq com segurança

A chave **nunca** vai no código. Ela fica nas **Propriedades do script**, uma área privada da sua conta.

1. Ainda em **⚙️ Configurações do projeto**, role até **Propriedades do script**.
2. Clique em **Adicionar propriedade do script**.
3. Preencha exatamente assim:

   | Campo | Valor |
   | --- | --- |
   | Propriedade | `GROQ_API_KEY` |
   | Valor | `gsk_sua_chave_aqui` (a chave copiada no Passo 1) |

4. Clique em **Salvar propriedades do script**.

> ⚠️ O nome precisa ser **exatamente** `GROQ_API_KEY` — sem espaços, sem acentos, tudo em maiúsculas. Se errar, o script avisa: *"GROQ_API_KEY não configurada"*.

**✅ Checkpoint:** a propriedade `GROQ_API_KEY` aparece na lista, com o valor mascarado.

---

### Passo 6 — Autorizar e rodar a primeira vez (manual)

A primeira execução é manual, para conceder as permissões e ver o resultado antes de automatizar.

1. Na barra superior do editor, no seletor de funções, escolha **`triarEmails`**.
2. Clique em **▶ Executar**.
3. O Google vai pedir autorização. Clique em **Revisar permissões**.
4. Escolha a sua conta Google.
5. Aparecerá o aviso **"O Google não verificou este app"** — isso é **normal**, porque o app é o *seu próprio* script. Clique em **Avançado** → **Acessar Triagem de E-mails com IA (não seguro)**.
6. Na tela de permissões, clique em **Continuar** (ou **Permitir**).

O script vai pedir permissão para:

| Permissão | Por quê |
| --- | --- |
| Ver, editar, criar e excluir seus e-mails | Ler, etiquetar, arquivar e mover conversas |
| Enviar e-mail em seu nome | Enviar o digest diário para você mesmo |
| Conectar-se a um serviço externo | Chamar a API da Groq |
| Ver seu endereço de e-mail principal | Definir o destinatário do digest |
| Gerenciar seus gatilhos | Criar a execução automática diária (pode aparecer só no Passo 8) |

> Dependendo da sua conta, o Google pode agrupar ou pedir essas permissões em dois momentos: agora e ao criar o gatilho. É normal.

7. Aguarde. A execução termina em poucos segundos (ou até ~1s por e-mail).
8. Abra o **Registro de execução** (painel inferior do editor) e procure a linha final:

```
[FIM] Sucesso: 12 | Falhas: 0 | Oportunidades: 2 | Tempo: 14s
```

**✅ Checkpoint:** o log mostra `[FIM] ...` sem `[ERRO FATAL]`, e as etiquetas `IA/Oportunidade`, `IA/Negativa-Vaga`, `IA/Marketing`, `IA/Outros` e `IA-Processado` já aparecem no seu Gmail (barra lateral esquerda, seção "Marcadores").

---

### Passo 7 — Conferir o resultado no Gmail

1. Abra o Gmail e clique em **Marcadores** (na lateral esquerda, pode estar em "Mais").
2. Você deve ver a estrutura:

```
IA-Processado
IA/Oportunidade
IA/Negativa-Vaga
IA/Marketing
IA/Outros
```

3. Confira se os e-mails de marketing/rejeição saíram da Caixa de Entrada (**arquivados**, não apagados — continuam buscáveis).
4. Se houve oportunidades no dia, verifique a chegada do e-mail com assunto **`[IA] N oportunidade(s) na sua caixa de entrada`** (dê uma olhada no Spam na primeira vez 😅).

> Nada é apagado. Todo e-mail continua no "Todos os e-mails" e pode ser encontrado por busca.

**✅ Checkpoint:** as etiquetas existem e a caixa de entrada está visivelmente mais limpa.

---

### Passo 8 — Agendar a execução diária automática

Agora o robô passa a trabalhar sozinho.

1. No seletor de funções, escolha **`configurarTriggerDiario`**.
2. Clique em **▶ Executar** — **uma única vez**.
3. Se pedir autorização novamente, repita o processo do Passo 6.
4. No menu lateral, clique no ícone de relógio **⏰ Acionadores**.
5. Confirme que existe uma linha assim:

| Função | Origem | Tipo | Horário |
| --- | --- | --- | --- |
| `triarEmails` | Baseado em tempo | Diariamente | 18h–19h |

> 🔁 Pode executar essa função quantas vezes quiser: ela **apaga o gatilho antigo antes de criar um novo**, então nunca duplica. Use isso se mudar o horário ou o fuso.

**✅ Checkpoint:** o acionador diário aparece na lista.

**Pronto! 🎉** A partir de agora, todos os dias entre 18h e 19h o script:

1. busca e-mails novos das últimas 48h (Caixa de Entrada, Promoções, Atualizações **e Spam**);
2. classifica com IA e organiza com etiquetas;
3. arquiva marketing e rejeições;
4. resgata oportunidades do Spam;
5. envia o resumo do dia, se houver algo relevante.

---

### Passo 9 — (Opcional) Ver as execuções do robô

Quer saber se ele rodou enquanto você não estava? No menu lateral, clique em **Execuções** (ícone de lista). Cada linha mostra data, duração e status. Clique em uma execução para ver o **Registro** completo.

Se aparecer `Falha`, copie a mensagem de erro e procure na seção [Problemas comuns](#problemas-comuns).

---

### Configuração (Propriedades do script)

O projeto tem uma única variável obrigatória, e ela **nunca** vai para o repositório:

| Variável | Obrigatória | Padrão | Efeito |
| --- | --- | --- | --- |
| `GROQ_API_KEY` | Sim | — | Autentica as chamadas à Groq; sem ela a execução aborta com mensagem explícita |

As demais configurações são constantes no código (veja [Personalização](#personalização)). Nenhum valor real de segredo é documentado aqui — apenas o nome da chave e o efeito dela.

---

### Funções disponíveis

| Função | O que faz |
| --- | --- |
| `triarEmails` | Execução principal: busca, classifica, aplica ações e envia o digest. É a função chamada pelo gatilho |
| `configurarTriggerDiario` | Cria (ou recria) o gatilho diário. Deve ser executada **uma vez** |
| `enviarResumoOportunidades` | Monta e envia o digest. Chamada internamente por `triarEmails` |
| `garantirLabels_`, `chamarGroq`, `aplicarAcoesGmail` | Funções auxiliares internas (sufixo `_` = uso privado) |

---

### Testando sem esperar até as 18h

- **Rodar agora:** selecione `triarEmails` e clique em ▶ Executar.
- **Forçar reprocessamento de um e-mail:** no Gmail, remova a etiqueta `IA-Processado` da conversa e rode `triarEmails` novamente.
- **Testar o digest sem oportunidades:** é normal não receber nada. Para testar o envio, aplique manualmente a etiqueta `IA/Oportunidade` em algum e-mail, remova `IA-Processado` e rode de novo.

---

### Instalação via clasp (avançado)

Se você prefere versionar o projeto no seu computador em vez de copiar e colar:

> Antes de começar, ative a **Apps Script API** em [script.google.com/home/usersettings](https://script.google.com/home/usersettings) (chave "Apps Script API" → **Ativar**). Sem isso, o `clasp` responde com erro 403.

```bash
# 1. Instale o Node.js (https://nodejs.org) e o clasp
npm install -g @google/clasp

# 2. Faça login na sua conta Google
clasp login

# 3. Clone este repositório
git clone https://github.com/soudevictor/email-triage.git
cd email-triage

# 4. Crie um projeto no Apps Script apontando para a pasta src/
clasp create --type standalone --title "Triagem de E-mails com IA" --rootDir src

# 5. Envie os arquivos
clasp push
```

Depois:

- A chave **continua** indo em **Configurações do projeto → Propriedades do script** (`GROQ_API_KEY`). O arquivo `src/appsscript.json` já traz o fuso `America/Sao_Paulo` e as permissões OAuth necessárias — ajuste o fuso se precisar.
- Abra o projeto com `clasp open` e rode `triarEmails` e depois `configurarTriggerDiario` uma vez pelo editor.
- O arquivo `.clasp.json` gerado é **local** e não deve ser commitado.

---

## Arquitetura de pastas

```text
.
├── src/
│   ├── TriagemEmails.gs     # código principal (o que se cola no Apps Script)
│   └── appsscript.json      # manifesto: fuso horário e permissões OAuth
├── .gitignore               # bloqueia .env e arquivos locais do clasp
├── README.md                # este guia
└── LICENSE                  # MIT
```

O repositório é intencionalmente plano: um arquivo de código, um manifesto e a documentação. Não há `node_modules`, build ou artefato gerado.

**Fluxo de dependência:** `gatilho diário → triarEmails → extração (GmailApp) → classificação (Groq API) → ações no Gmail (GmailApp) → digest (MailApp)`.
O código é organizado em blocos com prefixo `_` para funções privadas (convenção do Apps Script), sem imports externos.

### Onde editar cada coisa

| Conteúdo | Arquivo / símbolo |
| --- | --- |
| Prompt e regras de classificação | `SYSTEM_INSTRUCTION` em `src/TriagemEmails.gs` |
| Categorias e nomes das etiquetas | `CATEGORIAS`, `CATEGORIAS_ARQUIVADAS` e `LABELS` |
| Query do Gmail, limites, horário e pausa | objeto `CONFIG` |
| Comportamento por categoria (arquivar, mover) | `aplicarAcoesGmail` |
| Formato do resumo diário | `enviarResumoOportunidades` |
| Fuso horário e permissões OAuth | `src/appsscript.json` |

---

## Personalização

Tudo que você precisa ajustar está no topo do arquivo, no objeto `CONFIG` e no objeto `LABELS`:

| Constante | Padrão | Para que serve |
| --- | --- | --- |
| `MAX_THREADS_POR_EXECUCAO` | `40` | Máximo de conversas analisadas por execução. Reduza para `20` se bater no limite da Groq. |
| `GMAIL_QUERY` | `in:anywhere -in:trash -label:IA-Processado newer_than:2d` | Filtro de busca. Use o [guia de operadores do Gmail](https://support.google.com/mail/answer/7190). |
| `MAX_CARACTERES_CORPO` | `2000` | Quanto do corpo é enviado à IA. Menos = mais barato/rápido; mais = mais contexto. |
| `SLEEP_ENTRE_CHAMADAS_MS` | `1000` | Pausa entre chamadas (respeita o limite de requisições por minuto). |
| `TEMPO_MAXIMO_EXECUCAO_MS` | `4,5 min` | Trava de segurança antes do limite de 6 min do Apps Script. |
| `HORA_EXECUCAO_DIARIA` | `18` | Hora do dia (0–23). **Depois de mudar, rode `configurarTriggerDiario` novamente.** |

### Exemplos de mudanças comuns

<details>
<summary><b>Rodar 2x por dia (18h e 8h)</b></summary>

Substitua a função `configurarTriggerDiario` por:

```javascript
function configurarTriggerDiario() {
  ScriptApp.getProjectTriggers().forEach(function (t) {
    if (t.getHandlerFunction() === 'triarEmails') ScriptApp.deleteTrigger(t);
  });

  [8, 18].forEach(function (hora) {
    ScriptApp.newTrigger('triarEmails')
      .timeBased()
      .everyDays(1)
      .atHour(hora)
      .create();
  });
}
```

</details>

<details>
<summary><b>Adicionar uma nova categoria (ex.: "Financeiro")</b></summary>

1. Em `LABELS`, adicione `FINANCEIRO: 'IA/Financeiro'`.
2. Em `CATEGORIAS`, adicione `'FINANCEIRO'`.
3. Em `SYSTEM_INSTRUCTION`, descreva a nova categoria e o que entra nela.
4. Em `garantirLabels_`, adicione a criação da etiqueta.
5. (Opcional) Em `CATEGORIAS_ARQUIVADAS`, inclua se quiser arquivar automaticamente.

</details>

<details>
<summary><b>Manter as rejeições na Caixa de Entrada (não arquivar)</b></summary>

Remova `'NEGATIVA_VAGA'` de `CATEGORIAS_ARQUIVADAS`:

```javascript
const CATEGORIAS_ARQUIVADAS = ['MARKETING'];
```

</details>

<details>
<summary><b>Nunca analisar o Spam</b></summary>

Troque a query para:

```javascript
GMAIL_QUERY: '-in:spam -in:trash -label:IA-Processado newer_than:2d',
```

</details>

<details>
<summary><b>Usar outro provedor de IA (OpenAI, Gemini, Ollama...)</b></summary>

O código chama um endpoint compatível com o formato OpenAI (`/chat/completions`), então basta:

1. Trocar `CONFIG.ENDPOINT` pelo endpoint do provedor.
2. Ajustar `obterModeloGroq_()` (ou fixar um modelo direto no `payload`) e o cabeçalho `Authorization`.
3. Guardar a nova chave nas Propriedades do script, com outro nome, e ajustar a leitura em `triarEmails`.

</details>

---

## Qualidade

Não há suíte de testes automatizada nem CI configurado; a verificação é manual e local, na ordem abaixo:

```bash
# Valida a sintaxe do arquivo principal (Node.js usado apenas para a checagem)
cp src/TriagemEmails.gs /tmp/triagem.js && node --check /tmp/triagem.js

# Valida o manifesto
python3 -m json.tool src/appsscript.json
```

No Apps Script, antes de considerar uma alteração concluída:

1. Rode `triarEmails` manualmente e confirme `[FIM] ...` no Registro, sem `[ERRO FATAL]`.
2. Confira no Gmail se as 5 etiquetas existem e se `MARKETING`/`NEGATIVA_VAGA` saíram da Caixa de Entrada.
3. Teste o digest aplicando `IA/Oportunidade` manualmente ([Testando sem esperar até as 18h](#testando-sem-esperar-até-as-18h)).
4. Rode `configurarTriggerDiario` e confirme o acionador na aba **Acionadores**.
5. Se mexer em `oauthScopes`, reautorize e valide a execução do zero.

> Não há medição de cobertura, Lighthouse ou métricas de performance: este projeto não tem front-end.

---

## Limitações conhecidas

- **Só a última mensagem da conversa é analisada.** Em threads longas, o contexto anterior é ignorado.
- **Uma conversa já etiquetada com `IA-Processado` não é reanalisada**, mesmo que chegue uma resposta nova nela. Se quiser reprocessar, remova a etiqueta.
- **Janela de 2 dias (`newer_than:2d`) + limite de 40 por execução:** se você receber mais de 40 e-mails em 2 dias, os mais antigos podem sair da janela antes de serem processados. Aumente o limite ou reduza a janela conforme seu volume.
- **A IA pode errar.** A classificação é probabilística; o prompt foi escrito para "na dúvida, marcar como OPORTUNIDADE", mas revise o digest.
- **Nada é apagado** — apenas etiquetado, arquivado ou movido. Ainda assim, teste primeiro em uma conta secundária se quiser segurança total.
- **O conteúdo dos e-mails é enviado para a Groq** para classificação (apenas remetente, assunto e até 2.000 caracteres do texto). Leia a política de privacidade deles antes de usar em uma caixa com dados sensíveis.
- **Projeto não oficial:** não é afiliado ao Google nem à Groq. Alterações nas APIs podem exigir ajustes no futuro.

---

## Problemas comuns

| Sintoma | Causa provável | Solução |
| --- | --- | --- |
| `GROQ_API_KEY não configurada em Propriedades do script.` | Propriedade ausente, nome errado ou não salva | Refaça o **Passo 5**. O nome deve ser exatamente `GROQ_API_KEY` |
| `HTTP 401: invalid_api_key` | Chave errada, revogada ou copiada incompleta | Gere uma nova chave na Groq e atualize a propriedade |
| `HTTP 403` | Conta sem acesso ao recurso / restrição de organização | Verifique sua conta na Groq; em Workspace, o admin pode bloquear apps externos |
| `HTTP 429` | Limite de requisições por minuto/dia | Reduza `MAX_THREADS_POR_EXECUCAO` para 20 e/ou aumente `SLEEP_ENTRE_CHAMADAS_MS` para 2000 |
| `HTTP 404: model_not_found` | Modelo aposentado | Normalmente não ocorre (há descoberta automática). Se persistir, confira se sua conta tem modelos de texto disponíveis |
| `Nenhum modelo de texto disponível na sua conta Groq.` | Conta sem permissão/modelos | Crie a conta em [console.groq.com](https://console.groq.com) e confira a seção *Models* |
| Nada acontece / log vazio | A execução atual não foi salva | `Ctrl+S` e execute de novo |
| `Tempo máximo de execução atingido` | Muitos e-mails no lote | Normal: o restante é processado na próxima execução. Reduza `MAX_THREADS_POR_EXECUCAO` se recorrente |
| `Outra execução já está em andamento` | Uma execução anterior ainda rodando | Aguarde alguns minutos; o `LockService` evita duplicidade |
| Etiquetas não aparecem no Gmail | Cache da interface | Recarregue o Gmail (`F5`) e procure em **Marcadores** |
| Digest não chegou | Não houve oportunidade, ou caiu no Spam | O digest só é enviado quando há `OPORTUNIDADE`. Confira o Spam/Lixeira e o log |
| Gatilho não dispara | Fuso horário errado, gatilho removido ou autorização revogada | Confira **Acionadores**, ajuste o **Passo 4** e rode `configurarTriggerDiario` de novo |
| Pediu autorização de novo do nada | Permissões (escopos) mudaram | Autorize novamente; se necessário, revogue em [myaccount.google.com/permissions](https://myaccount.google.com/permissions) |
| `Service invoked too many times` / cota diária | Limite do Apps Script | Aguarde o reset diário (meia-noite no fuso do projeto) ou reduza o volume |

> 💡 **Regra de ouro para depurar:** menu **Execuções** → clique na execução com falha → leia o **Registro**. 90% dos problemas estão descritos ali.

---

## Privacidade e segurança

- A chave da API **nunca** vai para o código-fonte: fica em `PropertiesService.getScriptProperties()`, visível apenas para a sua conta.
- O script roda **na sua conta**, com **as suas** permissões. Ninguém mais tem acesso à sua caixa de entrada.
- **Se você suspeitar que a chave vazou** (print, commit, vídeo, issue), revogue-a imediatamente em [console.groq.com/keys](https://console.groq.com/keys) e crie outra.
- Em repositórios, mantenha `.env`, `.clasp.json` e afins no `.gitignore` — e ative o **secret scanning** no GitHub.
- Para revogar o acesso do script ao Gmail: [myaccount.google.com/permissions](https://myaccount.google.com/permissions).

---

## Quanto custa

**R$ 0,00.** O projeto inteiro roda dentro dos planos gratuitos.

| Recurso | Limite gratuito (aproximado) | Observação |
| --- | --- | --- |
| Google Apps Script | ~90 min de execução/dia (conta Gmail comum) | Cada execução leva poucos segundos |
| Groq API | ~30 req/min e ~1.000 req/dia por modelo | Os limites mudam; consulte [console.groq.com/settings/limits](https://console.groq.com/settings/limits) |
| Envio de e-mail (MailApp) | 100 e-mails/dia (Gmail) · 1.500/dia (Workspace) | O script envia no máximo **1** digest por dia |
| Gmail | — | Não consome cota paga |

> Não é necessário cartão de crédito em nenhuma etapa.

---

## Como desativar ou remover tudo

1. **Parar a automação:** no Apps Script, abra **⏰ Acionadores** e exclua o gatilho de `triarEmails`. (Ou apague o projeto em [script.google.com/home](https://script.google.com/home) → ⋮ → *Remover*.)
2. **Remover as etiquetas:** no Gmail, passe o mouse sobre cada etiqueta `IA/...` e `IA-Processado` → ⋮ → *Remover etiqueta*. Os e-mails **não** são apagados.
3. **Revogar o acesso:** em [myaccount.google.com/permissions](https://myaccount.google.com/permissions), remova o acesso do projeto.
4. **Apagar a chave:** em [console.groq.com/keys](https://console.groq.com/keys), revogue a chave usada.

---

## Roadmap

- [ ] Reanalisar conversas quando chega uma resposta nova (etiqueta por mensagem, não por thread)
- [ ] Suporte a mais de um provedor de IA com fallback automático
- [ ] Painel de métricas (quantos e-mails por categoria, economia de tempo)
- [ ] Configuração via `appsscript.json` sem editar código
- [ ] Suporte a idiomas além do português
- [ ] Testes automatizados com mocks do Gmail e da Groq

Tem uma ideia? Abra uma **Issue** 🙂

---

## Contribuindo

1. Faça um fork do projeto.
2. Crie uma branch: `git checkout -b feat/minha-melhoria`.
3. Faça o commit: `git commit -m "feat: minha melhoria"`.
4. Envie: `git push origin feat/minha-melhoria`.
5. Abra um **Pull Request** descrevendo o que mudou e por quê.

Encontrou um bug? Abra uma **Issue** com: o que você esperava, o que aconteceu e o trecho relevante do **Registro de execução** — **sem colar sua chave de API**.

---

## Documentação

- [`src/TriagemEmails.gs`](./src/TriagemEmails.gs) — implementação comentada, organizada em blocos (configuração, extração, integração com a Groq, ações no Gmail, digest, gatilho).
- [`src/appsscript.json`](./src/appsscript.json) — manifesto do projeto: fuso horário, runtime e permissões OAuth.
- Este README — instalação, personalização, qualidade e diagnóstico de problemas.
- [Issues](https://github.com/soudevictor/email-triage/issues) — bugs e ideias; o roadmap acima resume o que está planejado.

> Este projeto não mantém `SPEC.md` nem `agent.md`: por ser um arquivo único e sem build, o código comentado e este README são a fonte de verdade.

---

## Autor

**João Victor** — desenvolvedor de software, automações e integrações com IA

- GitHub: [@soudevictor](https://github.com/soudevictor)
- LinkedIn: [in/soudevictor](https://www.linkedin.com/in/soudevictor)

---

## Licença

MIT — veja [`LICENSE`](./LICENSE).
