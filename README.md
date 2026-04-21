# Maju Ferrão, garota do tempo no Telegram 🌤️

Um chatbot de clima pra Telegram feito no N8N, que responde com a temperatura atual de qualquer cidade do Brasil. O nome e a personalidade são uma homenagem à Maju Coutinho e à Mariana Ferrão, então ela fala com jeitinho, tem bom humor e não deixa ninguém na mão, nem quando a pessoa manda assunto totalmente fora do clima.

Esse projeto é o meu desafio da pós em Automação e Inteligência Artificial.

## O que a Maju faz

Você manda pra ela uma cidade pelo Telegram (exemplo: "Qual a temperatura de Salvador?" ou "tá quente em BH?") e ela responde com a temperatura atual daquela cidade, puxando da API da OpenWeather.

Mas ela não é só um robô de temperatura. Se você mandar outra coisa, tipo "oi", "me ensina bolo de cenoura" ou "qual o melhor time do Brasil", ela responde com carinho e redireciona de volta pro clima, que é o superpoder dela.

Se a cidade não for encontrada, ela também avisa de um jeito leve e te mostra o formato certo pra tentar de novo.

## Como o workflow funciona por dentro

Fiz uma arquitetura em camadas pra garantir que a Maju sempre responda bem, mesmo quando a entrada é bagunçada:

1. **Telegram Trigger** recebe a mensagem do usuário.
2. **analisador_de_msg** é um AI Agent que extrai cidade e estado do texto, usando o Tavily como ferramenta auxiliar pra resolver casos ambíguos (tipo quando a pessoa só escreve "São José" e tem várias).
3. **normalizacao** limpa o texto, tira acentos, deixa minúsculo e monta no formato `cidade,uf,br` que a OpenWeather espera.
4. **Text Classifier** decide se é uma consulta de clima válida ou um assunto aleatório.
5. Se for clima, vai pro **busca_clima** (OpenWeather) e depois pro **AI Agent** que formata a resposta final com a personalidade da Maju.
6. Se não for clima, vai pro **novos_assuntos**, que é outro AI Agent responsável por responder com charme e redirecionar a pessoa de volta pro tema do bot.
7. Tem um **IF** (`funcionou?`) checando se a OpenWeather retornou dados válidos. Se não retornou, cai no agente **ops! Algo errado**, que avisa a pessoa que a cidade não foi encontrada.

Todos os agentes que geram texto final usam o **Google Gemini** como modelo, e o Text Classifier usa o **OpenRouter**. O analisador de mensagem usa o **OpenAI**.

## O que você precisa pra rodar

Antes de importar o workflow, você vai precisar ter:

- N8N rodando (local, em nuvem ou Docker).
- Conta no Telegram com um bot criado pelo BotFather.
- Conta na OpenWeather com uma API key.
- Conta no Google Cloud com chave de API do Gemini.
- Conta na OpenAI com API key.
- Conta na OpenRouter com API key.
- Conta na Tavily com API key (pra busca complementar de cidade).

As chaves que são obrigatórias mesmo pra função principal do bot funcionar são:

- `TELEGRAM_BOT_TOKEN`
- `OPENWEATHER_API_KEY`

As outras (Gemini, OpenAI, OpenRouter, Tavily) são usadas pra dar inteligência e personalidade ao bot. Se você quiser rodar uma versão mais simples, dá pra adaptar o workflow removendo esses agentes, mas a graça da Maju some um pouco.

## Como criar o bot do Telegram

No Telegram, procure por `@BotFather` e envie `/newbot`. Ele vai te pedir um nome (o meu ficou "Maju Ferrão") e um username que precisa terminar em `bot`. No final, ele te devolve um token. Guarde esse token, porque é ele que vai virar o `TELEGRAM_BOT_TOKEN`.

## Como pegar a API key da OpenWeather

Cria uma conta em `https://home.openweathermap.org/users/sign_up` e confirma o email. Depois acessa `https://home.openweathermap.org/api_keys` e copia a chave que aparece lá. Essa é a `OPENWEATHER_API_KEY`. Vale lembrar que a chave nova demora alguns minutos pra começar a funcionar, então se der erro logo de cara, espera um pouquinho.

## Como importar o workflow no N8N

1. Baixa o arquivo `workflow-telegram-chatbot.json` desse repositório.
2. Abre o N8N e clica em **Import from File** (ou no menu dos três pontinhos no canto superior direito do canvas).
3. Seleciona o arquivo e confirma a importação.
4. O workflow vai aparecer com vários nodes já conectados, mas **todas as credenciais vão estar vazias**. Isso é proposital pra proteger as minhas.

## Como configurar as credenciais no N8N

Depois de importar, você precisa configurar uma credencial pra cada integração. No N8N, credenciais ficam em **Credentials** no menu lateral, ou você clica direto em cima do node e ele abre o campo pra selecionar/criar.

**Telegram (usado no Telegram Trigger e nos nodes de envio):**
Cria uma credencial do tipo **Telegram API**, cola o `TELEGRAM_BOT_TOKEN` no campo `Access Token` e salva. Depois é só selecionar essa credencial nos nodes `Telegram Trigger`, `envio_do_clima`, `explicacao_assunto` e `envio_do_problema`.

**OpenWeather (usado no node `busca_clima`):**
Esse é um detalhe importante. No meu workflow, por preguiça durante os testes, a `appid` ficou colada direto no query parameter do node. **Antes de subir pro teu GitHub, abre o node `busca_clima`, vai em Query Parameters, acha o campo `appid` e troca o valor pela expressão `={{ $env.OPENWEATHER_API_KEY }}`.** Depois disso, configure a variável de ambiente `OPENWEATHER_API_KEY` no teu N8N (se tiver usando Docker, é no `docker-compose.yml`; se for N8N Cloud, é nas configurações de environment variables).

**Google Gemini (usado nos 3 AI Agents de resposta):**
Cria uma credencial do tipo **Google Gemini API** e cola tua chave. Seleciona ela nos nodes `Google Gemini Chat Model`, `Google Gemini Chat Model1` e `Google Gemini Chat Model2`.

**OpenAI (usado no analisador de mensagem):**
Credencial do tipo **OpenAI API**, cola a chave, seleciona nos nodes `OpenAI Chat Model` e `OpenAI Chat Model1`.

**OpenRouter (usado no Text Classifier):**
Credencial do tipo **OpenRouter API**, cola a chave, seleciona no node `OpenRouter Chat Model`.

**Tavily (usado no node `pesquisa` como ferramenta do analisador):**
Cria uma credencial do tipo **HTTP Header Auth** com o nome que preferir, põe `Authorization` no campo Name e `Bearer SUA_CHAVE_TAVILY` no campo Value. Seleciona essa credencial no node `pesquisa`.

## Como executar e testar

Depois de todas as credenciais configuradas, ativa o workflow no toggle do canto superior direito do N8N. Abre o Telegram, procura pelo username do teu bot e manda uma mensagem.

Testes que recomendo pra ter certeza que tá tudo redondo:

1. **Cidade simples:** manda "Qual a temperatura de Salvador?". A resposta deve vir com a temperatura atual de Salvador, no estilo da Maju.
2. **Cidade com sigla:** manda "tá frio em SP?". Deve funcionar também, porque o analisador resolve a ambiguidade.
3. **Assunto aleatório:** manda "oi" ou "me fala uma piada". A Maju deve responder com carinho e te convidar a pedir uma cidade.
4. **Cidade inexistente:** manda "Qual a temperatura de Xpto123?". Deve cair no ramo de erro e a Maju avisa de um jeito leve que não achou.

## Variáveis de ambiente esperadas

Essas são as duas obrigatórias pela regra do desafio:

- `OPENWEATHER_API_KEY`: chave de API da OpenWeather, usada no node `busca_clima`.
- `TELEGRAM_BOT_TOKEN`: token do bot do Telegram, usado nas credenciais dos nodes do Telegram.

As demais (Gemini, OpenAI, OpenRouter, Tavily) ficam nas credenciais do próprio N8N, não precisam virar variável de ambiente.

## Link do BOT no telegram:
https://t.me/maju_ferrao_clima_bot

## Avisos finais

O arquivo `workflow-telegram-chatbot.json` desse repositório não contém nenhum token ou segredo real. Se você baixar e importar, vai precisar plugar as tuas próprias chaves, como expliquei acima.

Se quiser rodar o N8N localmente via Docker, dá uma olhada no `docker-compose.yml` (se estiver incluso nesse repo). As versões que testei foram o N8N `1.x` mais recente na data da entrega.

Qualquer dúvida, abre uma issue ou me chama. 🌤️
