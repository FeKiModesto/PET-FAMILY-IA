# Pet Family IA — Assistente Veterinário com IA Generativa

> **Componente de Inteligência Artificial do projeto Pet Family** — Integração com Gemini API para orientação veterinária preventiva personalizada, contextualizada com dados reais do pet e do comedouro IoT.

---

## Contexto do Projeto

O **Pet Family** é um ecossistema de cuidado contínuo para pets desenvolvido para o **Challenge FIAP 2026 — Clyvo Vet**. A plataforma conecta tutores, pets e clínicas veterinárias, promovendo o acompanhamento preventivo e a recorrência de consultas.

Este repositório (`PET-FAMILY-IA`) concentra a proposta, arquitetura e implementação do componente de **Inteligência Artificial Generativa** que alimenta o Assistente Pet Family — presente na tela de chat do aplicativo mobile.

---

## Problema de Negócio

Tutores de pets frequentemente não sabem interpretar sinais do cotidiano do animal — hábitos alimentares irregulares, temperatura ambiente inadequada, comportamentos atípicos — sem orientação especializada. Isso resulta em:

- Consultas veterinárias apenas em emergências, reduzindo o LTV das clínicas
- Vínculo fraco e baixa recorrência entre tutor, pet e clínica
- Dados coletados pelo comedouro IoT (temperatura, umidade, nível de ração, frequência de alimentação) subutilizados — existem, mas não geram valor para o tutor
- Assistente de chat com respostas fixas baseadas em palavras-chave, sem capacidade de entender contexto, nuances ou perguntas inéditas

---

## 🤖 Solução Proposta: IA Generativa com Contexto Dinâmico

Substituição do motor de respostas fixas (`if/else` por palavras-chave) por chamadas reais à **Gemini API (Google AI)**, com um **system prompt dinâmico** que injeta:

- Perfil completo do pet (nome, espécie, raça, idade, peso, notas de saúde)
- Dados em tempo real do comedouro IoT (temperatura ambiente, umidade, nível de ração, horário da última alimentação, contagem de refeições do dia)
- Histórico recente da conversa (últimas 10 mensagens para continuidade de contexto)

O resultado é um assistente capaz de responder qualquer pergunta veterinária preventiva de forma personalizada, como se conhecesse o pet pelo nome e soubesse o que aconteceu com ele nas últimas horas.

---

## Como a IA Agrega Valor

| Cenário | Sem IA (atual) | Com IA Generativa |
|---|---|---|
| "Meu gato comeu hoje?" | Não sabe responder | Consulta dados do IoT e responde com horário e quantidade |
| "A temperatura tá boa para o Rex?" | Resposta genérica sobre temperatura | Usa os 27.3°C do DHT22 e o perfil do Rex (raça, idade) para orientar |
| "Ele tá comendo menos que o normal" | Redireciona para veterinário | Analisa histórico de alimentações e sugere observação ou consulta |
| "Vacina dele vence quando?" | Não tem essa informação | Responde com base nas notas de saúde cadastradas no perfil |
| Pergunta inédita ou combinada | Resposta de fallback genérica | Raciocínio contextual real via LLM |

---

## Arquitetura de Integração

```
┌─────────────────────────────────────────────────────────────────┐
│                        TUTOR (App Mobile)                        │
│              React Native · Expo · Expo Router                   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    chat.tsx                              │   │
│   │  • Interface estilo WhatsApp (já implementada)           │   │
│   │  • Envia mensagem → chama geminiService.ts               │   │
│   │  • Exibe resposta na bolha "ai"                          │   │
│   └──────────────────────┬──────────────────────────────────┘   │
│                           │                                       │
│   ┌───────────────────────▼─────────────────────────────────┐   │
│   │               geminiService.ts  (NOVO)                   │   │
│   │                                                          │   │
│   │  1. Busca perfil do pet → AsyncStorage                   │   │
│   │  2. Busca dados do IoT  → GET /api/status (ESP32)        │   │
│   │  3. Monta system prompt dinâmico com contexto            │   │
│   │  4. Envia histórico + mensagem → Gemini API              │   │
│   │  5. Retorna resposta em linguagem natural                 │   │
│   └──────────────┬──────────────────┬───────────────────────┘   │
└──────────────────┼──────────────────┼───────────────────────────┘
                   │                  │
       ┌───────────▼──┐    ┌──────────▼──────────────┐
       │  Gemini API  │    │   ESP32 Smart Feeder     │
       │  (Google AI) │    │   (PET-FAMILY-IOT)       │
       │              │    │                          │
       │  Model:      │    │  GET /api/status →       │
       │  gemini-     │    │  { temperature,          │
       │   1.5-flash   │    │    humidity,             │
       │              │    │    foodLevel,            │
       │  Entrada:    │    │    lastFeedTime,         │
       │  system +    │    │    feedCount,            │
       │  histórico + │    │    doorState,            │
       │  mensagem    │    │    nextFeedIn }           │
       │              │    │                          │
       │  Saída:      │    │  Servidor web embutido   │
       │  texto       │    │  no próprio ESP32        │
       │  natural     │    │  via WiFi local          │
       └──────────────┘    └──────────────────────────┘
```

---

## Dados Utilizados pela IA

### Origem: Perfil do Pet (AsyncStorage — Mobile)

| Campo | Tipo | Uso pela IA |
|---|---|---|
| `name` | string | Personalização das respostas ("o Rex precisa...") |
| `species` | `dog` \| `cat` | Adapta recomendações por espécie |
| `breed` | string | Considera predisposições da raça |
| `age` | number | Ajusta frequências de vacina, check-up e vermifugação |
| `weight` | number | Orientações de dosagem e nutrição |
| `healthNotes` | string | Contexto de condições preexistentes |

### Origem: Comedouro IoT (ESP32 REST API — `GET /api/status`)

| Campo | Tipo | Uso pela IA |
|---|---|---|
| `temperature` | float | Alerta se fora da faixa ideal para a espécie/raça |
| `humidity` | float | Contexto ambiental para saúde respiratória |
| `foodLevel` | int (0–100%) | Informa nível de ração e sugere reabastecimento |
| `lastFeedTime` | string | "Seu pet foi alimentado às 14:32" |
| `feedCount` | int | Monitora frequência de alimentação do dia |
| `nextFeedIn` | int (ms) | Informa quando será a próxima alimentação automática |

---

## Abordagem de IA: Por que IA Generativa / LLM?

### Alternativas consideradas

| Abordagem | Por que descartada |
|---|---|
| Sistema de regras (if/else) | Já existe, não escala, não entende contexto nem perguntas inéditas |
| Modelo preditivo treinado | Requer dataset próprio de saúde veterinária — inviável no prazo |
| Sistema de recomendação | Focado em items/produtos, não em diálogo natural livre |
| NLP clássico (intent detection) | Limitado a intenções pré-definidas, sem raciocínio contextual |

### Por que Gemini 1.5 Flash

- **IA Generativa com LLM** é a abordagem ensinada pelo Prof. Arnaldo na disciplina (Gemini API, prompt engineering)
- O modelo já possui conhecimento veterinário preventivo embutido — não requer fine-tuning
- **Personalização via prompt** (não via treinamento): o contexto do pet e do IoT é injetado em cada requisição
- Gemini 1.5 Flash é gratuito no Google AI Studio para uso acadêmico, sem custo para o grupo
- API REST simples, integrável diretamente no `geminiService.ts` do projeto mobile

---

## Estrutura do Repositório

```
PET-FAMILY-IA/
├── README.md                        # Este documento
├── src/
│   ├── geminiService.ts             # Serviço principal: monta prompt + chama API
│   ├── promptBuilder.ts             # Constrói o system prompt dinâmico
│   └── iotClient.ts                 # Busca dados do ESP32 via fetch
├── docs/
│   └── architecture-diagram.png    # Diagrama arquitetural exportado
└── examples/
    ├── example-prompt.txt           # Exemplo de system prompt gerado
    └── example-responses.md        # Exemplos de interações reais
```

---

## Fluxo Detalhado de uma Mensagem

```
Usuário digita: "O Rex tá comendo bem hoje?"
         │
         ▼
geminiService.ts
  ├── 1. getPet() → { name: "Rex", species: "dog", age: 3, breed: "Labrador" }
  ├── 2. fetch("http://[IP_ESP32]/api/status") →
  │       { feedCount: 2, lastFeedTime: "08:12", foodLevel: 68,
  │         temperature: 27.3, humidity: 61 }
  ├── 3. promptBuilder.buildSystemPrompt(pet, iotData) →
  │       "Você é o Assistente Pet Family, veterinário virtual preventivo.
  │        Pet em acompanhamento: Rex, Labrador, 3 anos, macho.
  │        Dados do comedouro (agora mesmo): temperatura ambiente 27.3°C,
  │        umidade 61%, nível de ração 68%, 2 alimentações hoje,
  │        última às 08:12. Responda sempre de forma empática,
  │        em português, com orientações preventivas baseadas
  │        nos dados reais. Não invente dados que não foram fornecidos."
  └── 4. Gemini API ← [system prompt] + [histórico] + "O Rex tá comendo bem hoje?"

         ▼
Gemini responde:
"O Rex se alimentou 2 vezes hoje, a última às 08:12 da manhã 🐾
 Para um Labrador de 3 anos, o ideal são 2 refeições diárias —
 então está dentro do esperado! O nível de ração ainda está em 68%,
 sem necessidade de reabastecimento por agora.
 A temperatura de 27.3°C está um pouco elevada para a raça.
 Mantenha água fresca sempre disponível e evite exercícios no calor do dia."
         │
         ▼
chat.tsx exibe na bolha "ai"
```

---

## Como Executar (Sprint 4)

> A implementação completa será entregue na Sprint 4. Esta Sprint 3 documenta a proposta e arquitetura.

```bash
# Clonar repositório
git clone https://github.com/PET-FAMILYY/PET-FAMILY-IA.git

# Instalar dependências (integrado ao projeto mobile)
cd PET-FAMILY-MOBILE
npm install

# Configurar chave da API (arquivo .env)
echo "EXPO_PUBLIC_GEMINI_API_KEY=sua_chave_aqui" > .env

# Executar
npx expo start
```

---

## Variáveis de Ambiente

| Variável | Descrição |
|---|---|
| `EXPO_PUBLIC_GEMINI_API_KEY` | Chave da Gemini API (Google AI Studio) |
| `EXPO_PUBLIC_IOT_BASE_URL` | URL base do ESP32 (ex: `http://192.168.1.42`) |

---

## Critérios de Avaliação — Sprint 3

| Critério | Como atendido | Peso |
|---|---|---|
| Aplicação técnica de IA | LLM (Gemini) com prompt dinâmico contextualizado com IoT e perfil do pet | 60 pts |
| Clareza e didática do vídeo | Pitch de 5 min com demonstração do fluxo completo | 20 pts |
| Organização e documentação | Este README + diagrama arquitetural + exemplos | 20 pts |

---

## Integrantes

| Nome | RM | Turma |
|---|---|---|
| Felipe Kimodesto | RM 561810 | 2TDS |
| Pedro Vaz | RM 566551 | 2TDS |
| João Victor Luiz Oliveira Resende | RM 565139 | 2TDS |
| Vitor | — | 2TDS |

---

## Contexto Acadêmico

> **FIAP 2026** · Challenge Clyvo Vet · 2º Semestre  
> Disciplina: **Disruptive Architectures: IoT, IoB & Generative IA**  
> Prof. Arnaldo · Turma 2TDS  
> Entrega Sprint 3: **12/09/2026**

*Pet Family IA v0.1.0 — Proposta e Arquitetura (Sprint 3)*
