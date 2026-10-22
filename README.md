# ftr-bot-weather
Workflow com N8N para consultar temperaturas de cidades do Brasil

# 🌤️ Chatbot de Temperatura no Telegram com n8n
Chatbot para Telegram, construído no **n8n**, que informa a **temperatura atual de qualquer cidade do Brasil**. O usuário envia o nome da cidade e do estado (ex.: `São Paulo,SP,BR`), o workflow consulta a API gratuita da **OpenWeather** e responde com uma mensagem curta e amigável:

```
🌤️ A temperatura em Belo Horizonte é de 25°C.
```

Se a cidade não existir ou a API retornar algo inválido, o bot responde:

```
❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).
```

---

## 📁 Arquivos do repositório

| Arquivo | Descrição |
|---|---|
| `workflow-chatbot-telegram.json` | Workflow exportado do n8n (sem credenciais) |
| `docker-compose.yml` | Stack do n8n + túnel HTTPS (ngrok) para rodar localmente / no Portainer |
| `README.md` | Este documento |

---

## 🔄 Como o workflow funciona

```
Telegram Trigger → Capturar Entrada → Normalizar Cidade → Consultar OpenWeather → Validar Resposta
                                                                                    ├─ true  → Formatar Mensagem → Enviar Temperatura
                                                                                    └─ false → Mensagem de Erro  → Enviar Erro
```

1. **Telegram Trigger** – recebe as mensagens de texto enviadas ao bot.
2. **Capturar Entrada** (Set) – guarda o texto recebido na variável `queue` e o `chatId`.
3. **Normalizar Cidade** (Code) – remove acentos, converte para minúsculas, remove espaços extras (inclusive ao redor das vírgulas) e monta a consulta no formato `cidade,br`. Ex.: `São Paulo , SP, BR` → `sao paulo,br`.
   > A OpenWeather só aceita código de estado para cidades dos EUA, por isso a UF é lida e guardada, mas a consulta é feita com `cidade,br`.
4. **Consultar OpenWeather** (HTTP Request) – `GET https://api.openweathermap.org/data/2.5/weather` com *Send Query Parameters* ativado:
   - `q` / `queue`: mensagem formatada
   - `units`: `metric` (graus Celsius)
   - `lang`: `pt_br`
   - `appid`: `={{ $env.OPENWEATHER_API_KEY }}`

   A opção *Full Response* + *Never Error* está ligada para que o status HTTP chegue ao próximo nó mesmo em caso de 404.
5. **Validar Resposta** (IF) – verifica se `statusCode == 200`, se `body.main.temp` existe e se `body.name` não está vazio.
6. **Formatar Mensagem** (Code) – arredonda a temperatura (`Math.round`) e monta a string final com o nome da cidade.
7. **Mensagem de Erro** (Set) – monta a mensagem de erro padrão.
8. **Enviar Temperatura / Enviar Erro** (Telegram) – respondem no mesmo chat em que a mensagem chegou.
9. **Basic LLM** - Usado o Gemini 3.5-flash de forma opcional, para melhorar as respostas para o usuário

---

## ✅ Pré-requisitos

- Docker Desktop (com ou sem Portainer)
- Conta gratuita na [OpenWeather](https://openweathermap.org/api) e uma **API key**
- Um bot no Telegram criado pelo [@BotFather](https://t.me/BotFather) e seu **token**
- Conta gratuita no [ngrok](https://ngrok.com) com **authtoken** e um **domínio estático gratuito** (Dashboard → *Domains*)

---

## 🔐 Variáveis de ambiente

| Variável | Uso |
|---|---|
| `OPENWEATHER_API_KEY` | Lida pelo nó HTTP Request via `{{ $env.OPENWEATHER_API_KEY }}` |
| `NGROK_AUTHTOKEN` | Autenticação do túnel ngrok |
| `NGROK_DOMAIN` | Domínio estático do ngrok (ex.: `meu-bot.ngrok-free.app`) |

Nenhum valor real é armazenado no repositório. Use variavais de ambientes como nos exemplos abaixo:

OPENWEATHER_API_KEY=sua_chave_openweather_aqui
NGROK_AUTHTOKEN=seu_authtoken_ngrok_aqui
NGROK_DOMAIN=seu_dominio_aqui.ngrok-free.app

---

## 🐳 Subindo o ambiente

### Opção A – Portainer

1. Abra o Portainer → **Stacks** → **Add stack**.
2. Dê um nome (ex.: `n8n-chatbot`) e escolha **Web editor**.
3. Cole o conteúdo do `docker-compose.yml`.
4. Em **Environment variables**, clique em **Add an environment variable** e cadastre `OPENWEATHER_API_KEY`, `NGROK_AUTHTOKEN` e `NGROK_DOMAIN`.
5. Clique em **Deploy the stack**.
6. Acesse `http://localhost:5678` e crie a conta de administrador do n8n.

### Opção B – Docker Compose pela linha de comando

```bash
cp .env.example .env   # preencha os valores
docker compose up -d
```

Para conferir se o túnel está ativo, abra `http://localhost:4040`.

---

## 📥 Importando o workflow no n8n

1. No n8n, clique em **Create Workflow** (ou abra um workflow vazio).
2. No menu **⋯** (canto superior direito) escolha **Import from File...** e selecione `workflow-chatbot-telegram.json`.
3. O workflow aparece com os nós do Telegram sinalizando credencial ausente — isso é esperado, pois o arquivo não contém credenciais, gere-a e inclua no trigger (passos a seguir de como gerar).

---

## 🔑 Configurando as credenciais

### Telegram (`TELEGRAM_BOT_TOKEN`)

O token do bot é cadastrado diretamente no n8n, que o armazena criptografado no próprio banco de dados (volume `n8n_data`). Ele não fica no `docker-compose.yml`, no `.env` nem no JSON do workflow.

1. Abra o nó **Telegram Trigger** → em *Credential to connect with*, escolha **Create New Credential**.
2. No campo **Access Token**, cole o token gerado pelo @BotFather (o valor que corresponde a `TELEGRAM_BOT_TOKEN`).
3. Salve a credencial.
4. Selecione **a mesma credencial** nos nós **Enviar Temperatura** e **Enviar Erro**.

### OpenWeather (`OPENWEATHER_API_KEY`)

Não é preciso criar credencial no n8n: o nó **Consultar OpenWeather** lê a chave da variável de ambiente pela expressão `={{ $env.OPENWEATHER_API_KEY }}`. Basta que a variável esteja definida no container (Portainer ou `.env`).

> Chaves novas da OpenWeather podem levar até cerca de 2 horas para serem ativadas. Antes disso, a API devolve erro 401.

---

## ▶️ Executando o chatbot

1. Salve o workflow e **ative/publique** (toggle *Active* ou botão *Publish*, dependendo da versão do n8n).
2. No Telegram, abra o seu bot e envie uma cidade no formato `Cidade,UF,BR`.

### Exemplos de teste

| Mensagem enviada | Resposta esperada (temperatura varia) |
|---|---|
| `São Paulo,SP,BR` | `🌤️ A temperatura em São Paulo é de 22°C.` |
| `belo horizonte, mg, br` | `🌤️ A temperatura em Belo Horizonte é de 25°C.` |
| `Recife,PE,BR` | `🌤️ A temperatura em Recife é de 28°C.` |
| `Cidadeinexistente,XX,BR` | `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).` |

Também funciona enviando só o nome da cidade (ex.: `Curitiba`).

---

## 🔒 Segurança

- O `workflow-chatbot-telegram.json` **não contém** tokens, chaves nem IDs de credenciais.
- Nunca faça commit do arquivo `.env`; adicione-o ao `.gitignore`.
- Antes de publicar o repositório, confira com `grep -i "token\|appid\|key" workflow-chatbot-telegram.json` que só aparecem referências a variáveis.
