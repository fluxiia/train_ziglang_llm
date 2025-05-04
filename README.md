

## Documentação do Processo: Treinando um LLM para Zig 0.14

Com base nas últimas postagens do Akita em seu blog, decidi tentar ajustar um modelo **Qwen 2.5 3B** para entender a linguagem **Zig**, especificamente a versão **0.14**.

Minha intenção aqui é documentar o máximo possível de cada passo que executei até chegar no resultado final — começando, inclusive, **pelo final**.

### Objetivo Final

Precisamos de uma forma objetiva para avaliar a qualidade das respostas geradas pelo modelo. Nosso **benchmark principal** será:

> ✅ *"O código compila e executa corretamente a tarefa solicitada."*

Para isso, vamos utilizar o **[Codapi](https://www.codapi.org/)**, que compila e executa código Zig (entre outras linguagens) diretamente via API. A versão utilizada pelo Codapi já está atualizada para Zig 0.14, então ela servirá tanto para validar os dados gerados quanto como base para construir o dataset.

### Estratégia Geral

* **Validação Automatizada:** Utilizaremos o Codapi para testar os códigos gerados pelo modelo e determinar se são válidos.
* **Divisão do Dataset:** O dataset será dividido em treino e avaliação. Apesar de termos poucos dados, queremos medir a capacidade de **generalização** do modelo.
* **Escassez de Dados:** LLMs são treinados com *terabytes* de dados. Mesmo os processos de alinhamento envolvem *milhões* de exemplos. Aqui faremos um experimento com um volume muito menor, focando em qualidade e validação prática.

### Abordagem Técnica

1. **Documentação como Fonte:** Vamos usar o arquivo `zig-0.14.0-docs.md` como ponto de partida.
2. **Geração de Dados Sintéticos:**

   * Dividir o arquivo em seções menores.
   * Para cada seção, gerar **pares de perguntas e respostas** usando um LLM (ex: "Como funciona defer?", "Escreva um exemplo com error union").
3. **Validação Automática:**

   * Executar cerca de **5% dos códigos** gerados no Codapi para verificar se são **funcionais**.
4. **Estrutura baseada em Agentes:**

   * Um agente para **ler e dividir** o conteúdo.
   * Um agente para **gerar os pares**.
   * Um agente para **validar os códigos** via Codapi.


### Etapa 1 — Dividindo a Documentação em Seções

Para facilitar o processo de geração de perguntas e respostas, precisamos dividir a documentação oficial do Zig 0.14 (`zig-0.14.0-docs.md`) em seções menores e bem delimitadas.

O script abaixo realiza exatamente isso:

* Identifica títulos e subtítulos (níveis `#`, `##` e `###`).
* Agrupa o conteúdo de cada seção em objetos separados.
* Salva o resultado em dois arquivos:

  * `zig_sections.jsonl`: uma linha por seção com o campo `"text"`.
  * `zig_sections_stats.json`: informações úteis como título, nível e tamanho da seção.

#### Código:

```python
import re
import json
import os

def split_markdown_by_sections(markdown_content):
    """
    Divide o conteúdo markdown em seções baseadas nos cabeçalhos.
    Retorna uma lista de dicionários com o conteúdo de cada seção.
    """
    # Regex para identificar cabeçalhos markdown (# Título, ## Subtítulo, etc)
    section_pattern = r'^(#{1,3})\s+(.+)$'
    sections = []
    current_section = {"title": "Intro", "level": 0, "content": ""}
    
    for line in markdown_content.split('\n'):
        match = re.match(section_pattern, line)
        if match:
            # Se o conteúdo da seção atual não estiver vazio, salva a seção
            if current_section["content"].strip():
                sections.append(current_section)
            
            # Inicia uma nova seção
            level = len(match.group(1))
            title = match.group(2)
            current_section = {
                "title": title, 
                "level": level, 
                "content": f"{match.group(1)} {title}\n"
            }
        else:
            # Continua adicionando à seção atual
            current_section["content"] += line + "\n"
    
    # Adiciona a última seção se tiver conteúdo
    if current_section["content"].strip():
        sections.append(current_section)
    
    return sections

def save_sections_to_jsonl(sections, output_file):
    """
    Salva as seções em formato JSONL com a estrutura: {"text": conteúdo}
    """
    with open(output_file, "w", encoding="utf-8") as f:
        for section in sections:
            # Cada linha é um objeto JSON com o campo "text" contendo o conteúdo da seção
            json_line = json.dumps({"text": section["content"].strip()})
            f.write(json_line + "\n")
    
    print(f"Foram criadas {len(sections)} seções e salvas em {output_file}")

def main():
    input_file = "zig-0.14.0-docs.md.txt"
    output_file = "zig_sections.jsonl"
    
    # Verifica se o arquivo de entrada existe
    if not os.path.exists(input_file):
        print(f"Erro: O arquivo {input_file} não foi encontrado.")
        return
    
    # Lê o conteúdo do arquivo
    with open(input_file, "r", encoding="utf-8") as f:
        content = f.read()
    
    # Divide o documento em seções
    sections = split_markdown_by_sections(content)
    
    # Salva as seções em formato JSONL
    save_sections_to_jsonl(sections, output_file)
    
    # Salva também estatísticas sobre as seções em um arquivo separado
    stats = [{"id": i, "title": s["title"], "level": s["level"], "size": len(s["content"])} 
             for i, s in enumerate(sections)]
    
    with open("zig_sections_stats.json", "w", encoding="utf-8") as f:
        json.dump(stats, f, indent=2)
    
    print(f"Estatísticas das seções salvas em zig_sections_stats.json")

if __name__ == "__main__":
    main()

```

#### Exemplo de saída (`zig_sections.jsonl`):

```json
{"text": "# Introduction\nZig is a general-purpose programming language..."}
{"text": "## Error Handling\nZig has a unique approach to error handling..."}
```

#### Exemplo de saída (`zig_sections_stats.json`):

```json
[
  {"id": 0, "title": "Introduction", "level": 1, "size": 1024},
  {"id": 1, "title": "Error Handling", "level": 2, "size": 893}
]
```

## Etapa 2 — Geração de Dados Sintéticos com LLM (via Novita)

Agora que temos a documentação do Zig 0.14 dividida em seções, o próximo passo é gerar exemplos de **perguntas e respostas (instruction-response pairs)** que sirvam como dados de fine-tuning ou alinhamento.

### Plataforma de IA

Utilizamos a [**Novita.ai**](https://novita.ai/referral?invited_code=NBDEJX), uma alternativa à OpenAI que oferece compatibilidade com a API padrão. Com registro via GitHub e o link acima, você recebe até **\$15 em créditos** — o suficiente para gerar milhares de exemplos para nosso dataset.

### Estratégia de Geração

* Para cada seção da documentação:

  * Geramos de 2 a 3 pares de QA (perguntas conceituais e pedidos de geração de código).
  * Usamos o modelo `meta-llama/llama-3.1-8b-instruct` via Novita.
  * O formato gerado é **JSONL**, com campos `"instruction"` e `"response"`.
* Se a seção for muito longa, o conteúdo é processado em até **3 iterações**, com continuidade baseada na última resposta.

### Estrutura de Prompt

O prompt fornece **somente o conteúdo da documentação**, sem permitir que o modelo use conhecimentos externos, garantindo que os pares sejam realmente baseados no material oficial do Zig 0.14.

### Código

```python
import json
from openai import OpenAI
import time

# Configuração do cliente OpenAI com a API Novita
client = OpenAI(
    base_url="https://api.novita.ai/v3/openai",
    api_key="<API_NOVITA>",
)

def generate_qa_pairs(zig_content, long=False, last_response=None):
    """
    Gera pares de instrução-resposta usando o LLM baseados no conteúdo Zig fornecido.
    
    Args:
        zig_content (str): Conteúdo da documentação Zig a ser processado
        long (bool): Se True, indica que estamos processando um conteúdo longo em partes
        last_response (str): A última resposta recebida, usada para continuidade em conteúdos longos
    
    Returns:
        tuple: (conteúdo_gerado, is_long) onde is_long indica se ainda há mais conteúdo a ser processado
    """
    model = "meta-llama/llama-3.1-8b-instruct"
    max_tokens = 8192
    
    # Preparar o prompt para o LLM
    system_message = "You are a Zig language expert who creates instruction-response pairs to train language models. Use only the information provided, without adding external knowledge."
    
    if long and last_response:
        # Se estamos continuando um processamento longo, usamos um prompt diferente
        prompt = f"""
Continue generating instruction-response pairs based on the same Zig documentation content.
You previously generated the following response:

{last_response}

Now, generate 2-3 more instruction-response pairs using the same format but covering different aspects or examples.

Respond ONLY in JSONL format, with each line containing an object with the fields "instruction" and "response":
{{"instruction": "Question or instruction", "response": "Detailed response or code"}}
{{"instruction": "Another question or instruction", "response": "Another detailed response or code"}}
"""
    else:
        # Prompt original para o primeiro processamento
        prompt = f"""
Based on this content from the Zig language documentation, generate 2-3 instruction-response pairs.
The pairs can be questions about language concepts OR instructions to generate code.

For conceptual questions, create complete and clear questions that can be answered using only the information provided.
For code generation, create specific instructions requesting implementations that demonstrate the concepts covered.
Use only the text below from the Zig 0.14 documentation to generate the questions and answers.

ZIG DOCUMENTATION:
{zig_content}

Respond ONLY in JSONL format, with each line containing an object with the fields "instruction" and "response":
{{"instruction": "Question or instruction", "response": "Detailed response or code"}}
{{"instruction": "Another question or instruction", "response": "Another detailed response or code"}}
"""

    try:
        # Enviar para o LLM
        completion = client.chat.completions.create(
            model=model,
            messages=[
                {
                    "role": "system",
                    "content": system_message
                },
                {
                    "role": "user",
                    "content": prompt
                }
            ],
            max_tokens=max_tokens,
            temperature=0.7,
            stream=False
        )
        
        content = completion.choices[0].message.content.strip()
        
        # Determinar se o conteúdo ainda é longo e precisa de mais processamento
        # Assumimos que ainda é longo se o conteúdo original tiver mais de 5000 caracteres
        # e já tivermos processado menos de 3 iterações (ajuste esses valores conforme necessário)
        is_still_long = long and len(zig_content) > 5000
        
        return content, is_still_long
    except Exception as e:
        print(f"Erro ao gerar QA pairs: {e}")
        return "", False

def main():
    input_file = "zig_sections_improved.jsonl"
    output_file = "zig_dataset.jsonl"
    
    processed = 0
    success = 0
    
    # Abrir o arquivo de saída para escrita
    with open(output_file, "w", encoding="utf-8") as outfile:
        # Ler o arquivo JSONL linha por linha
        with open(input_file, "r", encoding="utf-8") as infile:
            for line_num, line in enumerate(infile):
                section = json.loads(line)
                content = section["text"]
                
                # Verificar se o conteúdo é significativo (mais de 200 caracteres)
                if len(content) > 200:
                    print(f"Processando seção {line_num + 1}...")
                    
                    # Variáveis para processamento de conteúdo longo
                    is_long = len(content) > 5000  # Consideramos longo se tiver mais de 5000 caracteres
                    iteration = 0
                    last_response = None
                    all_qa_pairs = []
                    
                    # Loop para processar o conteúdo em partes se for longo
                    while True:
                        # Gerar pares de QA, possivelmente continuando de uma resposta anterior
                        qa_pairs, still_long = generate_qa_pairs(content, long=is_long, last_response=last_response)
                        
                        print(f"Iteração {iteration + 1}:")
                        print(qa_pairs)
                        
                        if qa_pairs:
                            # Remover marcações de blocos de código ```jsonl ou ``` se presentes
                            cleaned_qa_pairs = qa_pairs.replace("```jsonl", "").replace("```", "")
                            all_qa_pairs.append(cleaned_qa_pairs)
                            last_response = qa_pairs  # Salvar a versão original para a próxima iteração
                        
                        iteration += 1
                        
                        # Se não for mais longo ou já tivermos feito 3 iterações, paramos
                        if not still_long or iteration >= 3:
                            break
                    
                    # Escrever todos os pares no arquivo de saída
                    if all_qa_pairs:
                        # Garantir que não haja marcações de código no arquivo final
                        final_output = "\n\n".join(all_qa_pairs).strip()
                        outfile.write(final_output + "\n\n")
                        success += 1
                    
          
                
                processed += 1
                
                # Imprimir progresso a cada 10 itens
                if processed % 10 == 0:
                    print(f"Progresso: {processed} seções processadas, {success} com sucesso")
    
    print(f"\nProcessamento concluído! {processed} seções processadas, {success} com sucesso")
    print(f"Dataset salvo em {output_file}")

if __name__ == "__main__":
    main()

```

### Exemplo de saída esperada:

```json
{"instruction": "Explique como funciona o comando `defer` em Zig.", "response": "O `defer` em Zig agenda uma instrução para execução no final do escopo atual..."}
{"instruction": "Implemente uma função Zig que usa `defer` para fechar um arquivo.", "response": "const std = @import(\"std\");\n..."}
```


---
Claro! Aqui está seu conteúdo reestruturado, organizado em tópicos claros, com uma linguagem mais fluida e didática — ideal para um post técnico ou tutorial:

---

## Entendendo Chat Template e o Padrão ChatML em Modelos Instrução

Ao treinar ou ajustar modelos de linguagem (LLMs) para interações em formato de chat, um elemento essencial é o **chat template** — responsável por transformar as mensagens do usuário, sistema e assistente em um formato que o modelo consegue compreender e gerar a próxima resposta corretamente.

### O Que é o ChatML?

O ChatML é um dos padrões de formatação de mensagens mais utilizados hoje, especialmente por modelos derivados da OpenAI e suas variantes open-source. Ele funciona com marcações explícitas para cada papel no diálogo, como este exemplo:

```txt
<|im_start|>user
Can I ask a question?<|im_end|>
```

### Como Funciona na Prática

Quando utilizamos uma API que segue o padrão OpenAI (como a da Novita), normalmente enviamos as mensagens no formato:

```python
messages = [
    {"role": "system", "content": "Prompt de sistema"},
    {"role": "user", "content": "Solicitação do usuário"}
]
```

Internamente, o tokenizador converte isso em algo como:

```txt
<|im_start|>system
Prompt de sistema<|im_end|>
<|im_start|>user
Solicitação do usuário<|im_end|>
<|im_start|>assistant
```

O modelo então **gera token por token** a partir desse ponto, completando o papel do assistente. O processo para quando ele encontra o **token de parada**, chamado `eos_token`, que nesse caso é `<|im_end|>`.

### Personalizando o Chat Template

Se quiser, você pode **criar papéis personalizados** no template, como `docs`, `tool`, `function`, entre outros. Isso é muito útil para tarefas como:

* Passar trechos de documentação.
* Simular chamadas de ferramentas.
* Informar dados intermediários no raciocínio do modelo.

### Onde Encontrar o Template?

Você pode descobrir o `chat_template` de qualquer modelo no Hugging Face acessando o arquivo `tokenizer_config.json` do repositório. Lá, procure pela chave `chat_template`.

Esses templates são escritos em **Jinja2**, uma linguagem de templates com lógica condicional. Aqui está um trecho do template do modelo **Qwen 2.5**, da Alibaba:

```jinja2
{% for message in messages %}
  {% if message.role == "user" %}
    <|im_start|>user
    {{ message.content }}<|im_end|>
  {% elif message.role == "assistant" %}
    <|im_start|>assistant
    {{ message.content }}<|im_end|>
  {% endif %}
{% endfor %}
```

No caso do Qwen 2.5 com suporte a ferramentas, o template ainda trata `tool_calls`, `tool_responses`, e outros papéis especiais.

### Importante: Apenas Modelos Instrução Têm Chat Template

Os **modelos base (base models)**, ou seja, não-instruídos, **não possuem chat template**. Eles são treinados com tarefas de preenchimento de texto — ou seja, se você der a eles:

```
Zig is a low-level language that...
```

Eles vão apenas tentar **completar** esse texto, e não responder uma pergunta como um assistente. Já os modelos instruídos (como LLaMA-2-chat, Mistral-instruct, Qwen-chat) respondem comandos e seguem conversas estruturadas.

---

## Convertendo Nosso Dataset para o Padrão ChatML

Agora que entendemos o funcionamento do chat template, o próximo passo do projeto é transformar o nosso dataset de perguntas e respostas em um formato compatível com esse padrão, como o ChatML. Para isso:

* Cada item do dataset será transformado em uma sequência com:

  * Um `system prompt` (opcional, se houver).
  * Um `user message` com a instrução.
  * Um `assistant message` com a resposta.

### Exemplo:

#### Original (instrução e resposta):

```json
{
  "instruction": "Explique como funciona o comando `defer` em Zig.",
  "response": "O `defer` agenda uma operação para o fim do escopo atual..."
}
```

#### Convertido para ChatML:

```txt
<|im_start|>user
Explique como funciona o comando `defer` em Zig.<|im_end|>
<|im_start|>assistant
O `defer` agenda uma operação para o fim do escopo atual...<|im_end|>
```

Isso garante que o modelo treinado ou ajustado saiba exatamente qual mensagem é do usuário e qual é a resposta esperada.
Perfeito! Essa é a terceira etapa do seu pipeline — **preparar o dataset para fine-tuning**, convertendo para o formato esperado pelo `unsloth`, que adota a estrutura `{"conversations": [...]}` em JSONL.

---

## Etapa 3 — Conversão para o Formato ChatML (Compatível com Unsloth)

Depois de gerar os pares de **instrução e resposta**, o próximo passo é preparar o dataset no **formato esperado por frameworks como o `unsloth`**, que utilizam o padrão ChatML para treinar modelos de linguagem com base em conversas.

### Objetivo

Transformar este tipo de entrada:

```json
{"instruction": "What is the purpose of doc comments in Zig?", "response": "Doc comments in Zig are used to provide documentation for the code..."}
```

Em algo assim:

```json
{"conversations": [
  {"role": "user", "content": "What is the purpose of doc comments in Zig?"},
  {"role": "assistant", "content": "Doc comments in Zig are used to provide documentation for the code..."}
]}
```

### Script de Conversão

Abaixo está o script completo que realiza a conversão:

```python
import json
import re

def convert_to_chatml(input_file, output_file):
    """
    Convert the zig_dataset.jsonl file to chatml format.
    Input: JSONL com {"instruction": "...", "response": "..."}
    Output: JSONL com {"conversations": [{"role": "user", ...}, {"role": "assistant", ...}]}
    """
    chatml_data = []
    
    with open(input_file, 'r', encoding='utf-8') as f:
        content = f.read()
    
    content = re.sub(r'(\{[^\{\}]*?)\n([^\{\}]*?\})', r'\1 \2', content, flags=re.DOTALL)
    
    json_objects = []
    current_obj = ""
    in_obj = False
    
    for line in content.splitlines():
        line = line.strip()
        if not line:
            continue
            
        if line.startswith('{') and not in_obj:
            current_obj = line
            in_obj = True
            if line.endswith('}'):
                json_objects.append(current_obj)
                current_obj = ""
                in_obj = False
        elif line.endswith('}') and in_obj:
            current_obj += " " + line
            json_objects.append(current_obj)
            current_obj = ""
            in_obj = False
        elif in_obj:
            current_obj += " " + line
    
    for obj in json_objects:
        try:
            data = json.loads(obj)
            chatml_conversation = {
                "conversations": [
                    {"role": "user", "content": data["instruction"]},
                    {"role": "assistant", "content": data["response"]}
                ]
            }
            chatml_data.append(chatml_conversation)
        except json.JSONDecodeError as e:
            print(f"Erro ao interpretar JSON: {e}")
            print(f"Objeto problemático: {obj[:100]}...")
        except KeyError as e:
            print(f"Chave ausente: {e}")
            print(f"Objeto problemático: {obj[:100]}...")
    
    with open(output_file, 'w', encoding='utf-8') as f:
        for item in chatml_data:
            f.write(json.dumps(item, ensure_ascii=False) + '\n')
    
    return len(chatml_data)


if __name__ == "__main__":
    input_file = "zig_dataset.jsonl"
    output_file = "zig_dataset_chatml.jsonl"
    
    count = convert_to_chatml(input_file, output_file)
    print(f"Conversão concluída! {count} conversas salvas em {output_file}")
```

### Resultado

Este script gera um arquivo chamado `zig_dataset_chatml.jsonl` compatível com:

* **Unsloth**
* **Axolotl**
* **SFT frameworks compatíveis com chat format**

Agora, o modelo sabe exatamente onde o usuário fez uma pergunta e onde ele (como assistente) deve responder.

---

## Etapa 4 — Treinando com Unsloth e Qwen 2.5

Para realizar o fine-tuning do nosso modelo com o dataset gerado, optamos por usar a biblioteca **[Unsloth](https://github.com/unslothai/unsloth)**. Ela é uma otimização sobre a TRL (Transformers Reinforcement Learning) que permite treinar modelos de forma mais leve, eficiente e rápida, especialmente com suporte a LoRA + 4bit quantization.

### Por que Unsloth?

* ✅ Otimização de memória (QLoRA).
* ✅ Treinamento rápido, mesmo em GPUs com 24GB de VRAM.
* ✅ Suporte direto a modelos em 4bit (quantizados).
* ✅ Integração com chat templates prontos, como os da família Qwen.

---

### Instalação dos pacotes necessários:

```bash
pip install --no-deps bitsandbytes accelerate xformers==0.0.29.post3 peft trl==0.15.2 triton cut_cross_entropy unsloth_zoo
pip install sentencepiece protobuf datasets huggingface_hub hf_transfer wandb
pip install --no-deps unsloth
```

---

### Carregando o modelo

Neste projeto, usamos o modelo **Qwen2.5 3B Instruct**. Ele é uma versão compacta e instruída da família Qwen, ideal para fine-tuning com LoRA.

```python
from unsloth import FastLanguageModel
import torch
max_seq_length = 4096 # Tamanho da janela de contexto, quanto maior o contexto maior consumo de vram e mais demorado
dtype = None # Podemos deixar None para otimização automatica
load_in_4bit = True # Vamos carregar em 4bits para quantização automatica, LORA é 8bits e QLORA é 4bits (menor consumo, menor precisão)

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name = "Qwen/Qwen2.5-3B-Instruct", # Aqui carregamos o modelo do hugginface
    max_seq_length = max_seq_length, 
    dtype = dtype, 
    load_in_4bit = load_in_4bit, 
)

# Agora vamos carregar o lora que explicando muito por cima é criar uma copia menor do modelo que vai ser treinada.
# Para uma explicação mais completa sobre lora indico esse artigo: Não, aumentar o Alpha significa multiplicar os pesos LoRA por um número maior. Especificamente, α/r. Para r=4, α=8 o efeito será 2x. O artigo também menciona: 

model = FastLanguageModel.get_peft_model(
    model, # Passamos o modelo
    r = 128, # Aqui vamos definir o tamanho das matrizes, quanto maior mais memoria e mais conteudo é armazenado pelo modelo
    target_modules = ["q_proj", "k_proj", "v_proj", "o_proj",
                      "gate_proj", "up_proj", "down_proj","lm_head", "embed_tokens"],
    # Aqui vamos definir quais camadas vão ser treinadas, adicionei "lm_head" e "embed_tokens", que aumentam o consumo de memoria e tempo de treinamento, mas melhoram o desempenho para aprender novos conteudos.
    lora_alpha = 128, # Aqui vamos definir a importancia das matrizes, quanto maior significa que maior peso vai ser dado a matriz de r
    lora_dropout = 0, # Supports any, but = 0 is optimized 
    bias = "none",    # Supports any, but = "none" is optimized
    use_gradient_checkpointing = "unsloth", # True or "unsloth" for very long context
    random_state = 3407,
    use_rslora = False,  # We support rank stabilized LoRA
    loftq_config = None, # And LoftQ
)

from unsloth.chat_templates import get_chat_template

# Unsloth já define o chat template de cada modelo então precisamos apenas setar a versão 

tokenizer = get_chat_template(
    tokenizer,
    chat_template = "qwen2.5",
)

# Função responsável por converter de json para chatml aplicando o chat template do tokenizer.
def formatting_prompts_func(examples):
    convos = examples["conversations"]
    texts = [tokenizer.apply_chat_template(convo, tokenize = False, add_generation_prompt = False) for convo in convos]
    return { "text" : texts, }
pass

from datasets import load_dataset
# Vamos carregar o dataset zig_dataset_chatml.jsonl
dataset = load_dataset("json", data_files="zig_dataset_chatml.jsonl", split = "train")

# Vamos verificar as respostas do modelo antes do treinamento vou pegar algumas amostras do dataset.
import json
import random
from unsloth.chat_templates import get_chat_template

# Enable native 2x faster inference
FastLanguageModel.for_inference(model)

# Função para processar exemplos e executar o modelo
def process_example(example):
    # Extrair a pergunta e resposta esperada do exemplo
    user_content = example["conversations"][0]["content"]
    expected_response = example["conversations"][1]["content"]
    
    # Criar a mensagem para o modelo
    messages = [
        {"role": "user", "content": user_content},
    ]
    
    # Preparar entrada para o modelo
    inputs = tokenizer.apply_chat_template(
        messages,
        tokenize = True,
        add_generation_prompt = True,  # Must add for generation
        return_tensors = "pt",
    ).to("cuda")
    
    # Gerar resposta com o modelo
    outputs = model.generate(
        input_ids = inputs, 
        max_new_tokens = 256, 
        use_cache = True,
        temperature = 0.7,  # Reduzir a temperatura para respostas mais determinísticas
        min_p = 0.1
    )
    
    # Decodificar a resposta
    model_response = tokenizer.batch_decode(outputs)[0]
    
    # Exibir resultados
    print("\nPergunta:")
    print(user_content)
    print("\nResposta:")
    print(model_response)
    print("\nResposta esperada:")
    print(expected_response)
    print("\n" + "-"*80)

# Carregar o dataset
filename = "zig_dataset_chatml.jsonl"
dataset = []

try:
    with open(filename, 'r', encoding='utf-8') as f:
        for line in f:
            if line.strip():
                try:
                    example = json.loads(line)
                    dataset.append(example)
                except json.JSONDecodeError:
                    continue
    
    print(f"Carregados {len(dataset)} exemplos do dataset.\n")
    
    # Usar perguntas pré-definidas para avaliar o modelo (usando índices fixos)
    # Definir os índices das perguntas que queremos usar para avaliação
    test_indices = [0, 1, 5, 10, 20]  # Você pode modificar estes índices conforme necessário
    
    # Verificar se os índices estão dentro do intervalo válido
    valid_indices = [idx for idx in test_indices if idx < len(dataset)]
    if len(valid_indices) < len(test_indices):
        print(f"Aviso: Alguns índices estavam fora do intervalo. Usando {len(valid_indices)} exemplos.")
    
    # Obter os exemplos pelos índices
    test_examples = [dataset[idx] for idx in valid_indices]
    
    # Processar cada exemplo
    for i, example in enumerate(test_examples, 1):
        print(f"Exemplo {i}/{len(test_examples)}")
        process_example(example)
        
except Exception as e:
    print(f"Erro ao processar o dataset: {e}")


from unsloth.chat_templates import standardize_sharegpt
# Existem dois formatos que são bem parecidos sharegpt e o padrão da openai a diferença entre eles é somente o nome das chaves então unsloth criou standardize_sharegpt que suporta as duas versões.
dataset = standardize_sharegpt(dataset)
dataset = dataset.map(formatting_prompts_func, batched = True,)

print("Dataset Original:")
print(dataset[5]["conversations"])

print("Após passar pelo chat template:")
print(dataset[8]["text"])

from transformers import TrainingArguments
from unsloth import is_bfloat16_supported
from unsloth import UnslothTrainer, UnslothTrainingArguments

trainer = UnslothTrainer(
    model = model,
    tokenizer = tokenizer,
    train_dataset = dataset,
    dataset_text_field = "text",
    max_seq_length = max_seq_length,
    dataset_num_proc = 2,

    args = UnslothTrainingArguments(
        per_device_train_batch_size = 2,
        gradient_accumulation_steps = 8,

        warmup_steps = 10,
        num_train_epochs = 3,

        embedding_learning_rate = 1e-5, # Aqui definimos o treinamento do embed layer menor do que a taxa de treinamento do restante do modelo.
        learning_rate = 5e-5,

        fp16 = not is_bfloat16_supported(),
        bf16 = is_bfloat16_supported(),
        logging_steps = 1,
        optim = "adamw_8bit",
        weight_decay = 0.01,
        lr_scheduler_type = "linear",
        seed = 3407,
        output_dir = "outputs",
        report_to = "wandb", # Wandb para Logs
    ),
)

trainer_stats = trainer.train()
```
---

## Etapa 5 — Avaliação: Antes e Depois do Fine-Tuning

Para validar se o modelo aprendeu com o nosso dataset de Zig 0.14, realizamos um teste simples:

* **Selecionamos 5 exemplos reais do dataset.**
* **Pedimos ao modelo que gerasse a resposta antes e depois do treinamento.**
* **Comparamos a saída gerada com a resposta esperada.**

---

### 🧪 Exemplo 1

**Pergunta:**

> What is the purpose of doc comments in Zig?

**Antes do Fine-Tuning:**

> In Zig programming language, doc comments (also known as "documentation comments"...)
> (*Resposta longa, genérica, e baseada em estilos de outras linguagens*)

**Depois do Fine-Tuning:**

> Doc comments provide documentation for code, allowing other developers to understand what the code does without having to read through the implementation...<|im\_end|>

**Esperado:**

> Doc comments in Zig are used to provide documentation for the code...

✅ *Melhoria: Resposta mais direta, mais próxima do estilo esperado no dataset.*

---

### 🧪 Exemplo 2

**Pergunta:**

> Write a Zig code snippet that demonstrates the use of primitive type literals and assignment.

**Antes:**

```zig
const intLiteral: i32 = 42;
const floatLiteral: f64 = 3.14;
// ...
```

**Depois:**

```zig
const x: i32 = 10; const y: u32 = 20; const z: f64 = 30.5;<|im_end|>
```

**Esperado:**

```zig
const a: i32 = 10; const b: u32 = 20; const c: f64 = 30.5;
```

✅ *Melhoria: Resposta curta, direta e aderente ao estilo do dataset.*

---

### 🧪 Exemplo 3

**Pergunta:**

> What is the purpose of the `defer` statement in Zig?

**Antes:**

> Longa explicação com múltiplos exemplos e explicações repetitivas.

**Depois:**

> The `defer` statement is used to run code at function exit, regardless of how the function exits. It is typically used for cleanup tasks...<|im\_end|>

**Esperado:**

> In Zig, the `defer` statement is used to defer the execution of a block of code until the end of the current scope...

✅ *Melhoria clara na concisão e alinhamento conceitual.*

---

### 🧪 Exemplo 4

**Pergunta:**

> What is the primary purpose of the Zig Standard Library?

**Antes:**

> Lista longa com 6 tópicos explicativos, linguagem genérica.

**Depois:**

> The primary purpose of the Zig Standard Library is to provide a set of useful functions and types...<|im\_end|>

**Esperado:**

> ...commonly used algorithms, data structures, and definitions...

✅ *Mais objetivo, alinhado com a resposta-alvo.*

---

### 🧪 Exemplo 5

**Pergunta:**

> In which places are doc comments allowed in Zig?

**Antes:**

> Explicação confusa, menciona ausência de suporte, fala de doxygen e IDEs.

**Depois:**

> Doc comments can be used at the start of a file, at the start of a function...<|im\_end|>

**Esperado:**

> Doc comments are only allowed in certain places, such as before a function...

✅ *Maior precisão após o treinamento.*

---
## Etapa 6 — Expansão do Dataset e Novo Ciclo de Treinamento

Após a primeira rodada de testes, percebi que embora as respostas do modelo estivessem **melhores, mais concisas e com estilo alinhado**, elas ainda **careciam de profundidade e precisão conceitual** em alguns casos.

Um dos motivos era claro: **nosso dataset original tinha apenas 750 exemplos**, o que é muito pouco para ensinar uma linguagem inteira como Zig, mesmo com LoRA.

### 🔍 Estratégia: Aumentar a Diversidade e Cobertura dos Dados

Para resolver isso, decidi **aumentar significativamente o volume e a variedade de fontes**, utilizando não apenas a documentação oficial, mas também materiais educacionais da comunidade Zig.

### 📚 Novas Fontes Incluídas

| Fonte                                                                                     | Descrição                                        | Linhas |
| ----------------------------------------------------------------------------------------- | ------------------------------------------------ | ------ |
| [zigcc/zig-cookbook](https://github.com/zigcc/zig-cookbook)                               | Receitas práticas em Zig                         | 1.596  |
| [jkitajima/learning-zig-karlseguin](https://github.com/jkitajima/learning-zig-karlseguin) | Curso completo baseado nos textos de Karl Seguin | 1.040  |
| [sobeston/zig.guide](https://github.com/sobeston/zig.guide)                               | Guia introdutório em Markdown                    | 456    |
| [ziglang.org/documentation/master](https://ziglang.org/documentation/master/)             | Documentação oficial mais atual                  | 1.875  |
| [ziglang.org/release-notes 0.14](https://ziglang.org/download/0.14.0/release-notes.html)  | Notas de versão da release-alvo                  | 846    |

📈 **Total: 5.817 exemplos + training-prompts.jsonl original**

---

### ⚙️ Mesma Pipeline, Agora com Mais Dados

O processo de geração de exemplos segue o mesmo fluxo:

1. **Busca por arquivos `.md` e `.mdx`** nos repositórios.
2. **Divisão por seções e tópicos.**
3. **Geração de perguntas e respostas com LLM (Novita).**
4. **Validação parcial por amostragem.**
5. **Conversão para o formato ChatML (`chat_template`) com Unsloth.**

Além disso, reaproveitamos o dataset `training-prompts.jsonl` usado por Akita como parte do oversampling para manter coerência com os exemplos iniciais.

---

### ▶️ Treinamento com o Novo Dataset

Com os novos dados integrados, o **script de treinamento permanece exatamente o mesmo**, apenas com o caminho do novo JSONL atualizado:

```python
dataset = load_dataset("json", data_files="zig_dataset_chatml.jsonl", split = "train")
```

Todo o restante do pipeline (aplicação do chat template, formatação, configuração do LoRA e `UnslothTrainer`) continua funcional, escalando bem mesmo com um dataset significativamente maior.

Excelente avanço! Com essa etapa, você demonstra de forma clara o **ponto de partida antes do fine-tuning** com o novo dataset do Hugging Face. Isso fortalece seu artigo como um caso real de melhoria de LLM por LoRA.

Aqui está a continuação do artigo, explicando essa etapa de forma didática e estruturada:

---

## Etapa 7 — Avaliação Inicial com Dataset do Hugging Face

Antes de realizar o novo fine-tuning, decidimos medir o desempenho atual do modelo **Qwen 2.5** com o dataset já formatado e publicado no Hugging Face:

📁 [JJhooww/ziglang\_sharegpt](https://huggingface.co/datasets/JJhooww/ziglang_sharegpt)

Esse dataset contém dois splits:

* **Treino** (`train`): usado para fine-tuning.
* **Avaliação** (`test`): usado para medir o desempenho antes e depois do treinamento.

---

### ⚙️ Procedimento

* Carregamos o dataset usando `datasets.load_dataset`.
* Avaliamos os **primeiros 5 exemplos do split de teste**, pedindo que o modelo gere uma resposta para cada pergunta.
* Comparamos a **resposta gerada** com a **resposta esperada**.

---

### 📊 Resultados Antes do Treinamento

#### 🔹 Exemplo 1

**Pergunta:**

> What is your knowledge cutoff for Zig?

**Resposta do modelo:**
Fala sobre Zigbee e diz que não tem dados sobre Zig após 2021.

**Resposta esperada:**

> I know about Zig up to version 0.14.0 (2025), with access to the latest documentation.

❌ **Resultado:** resposta incorreta e fora do tema.

---

#### 🔹 Exemplo 2

**Pergunta:**

> What is the latest version of the Zig programming language?

**Resposta do modelo:**

> Zig 1.7.0 (2023)

**Resposta esperada:**

> Zig 0.14.0, publicado em 2025

❌ **Resultado:** desatualizado e incorreto.

---

#### 🔹 Exemplo 3

**Pergunta:**

> Can you show me how to use feature X from Zig 0.14.0?

**Resposta do modelo:**
Resposta vaga, genérica, com exemplos inventados de "zigconfig.zig".

**Resposta esperada:**

> Explicação direta e precisa da feature X conforme a versão 0.14.0.

❌ **Resultado:** modelo responde com "placeholders", sem conhecimento real da versão.

---

#### 🔹 Exemplo 4

**Pergunta:**

> Which version of the Zig programming language are you familiar with?

**Resposta do modelo:**

> Zig 1.7.0, última versão conhecida em 2023

**Resposta esperada:**

> Zig 0.14.0, com base nos documentos de 2025

❌ **Resultado:** novamente desatualizado.

---

#### 🔹 Exemplo 5

**Pergunta:**

> What is your Zig knowledge cutoff?

**Resposta do modelo:**

> Treinamento até 2021

**Resposta esperada:**

> Até 0.14.0 (2025), com acesso à documentação mais recente

❌ **Resultado:** conhecimento incorreto e desatualizado.

---

## Etapa 8 — Tempo de Treinamento e Custo: Eficiência Real com GPU Gratuita

Com o novo dataset e validação por época ativada, o tempo de treinamento naturalmente aumentou. Antes, com \~750 exemplos e sem `eval_dataset`, o tempo total para 3 épocas era de aproximadamente **15 minutos**.

Após a expansão para mais de **5.800 pares de dados**, divididos entre treino e teste, o tempo subiu para:

⏱️ **1 hora, 10 minutos e 36 segundos**
📈 **3 épocas completas**
🔁 Avaliação e salvamento do modelo ao final de cada época.

---

### 🚀 Hardware e Infraestrutura Utilizada

Apesar do aumento, o custo de toda a operação se manteve **zero**, graças a uma combinação inteligente de ferramentas e infraestrutura gratuita:

| Recurso                            | Finalidade                                             | Custo |
| ---------------------------------- | ------------------------------------------------------ | ----- |
| 🧠 **Novita.ai**                   | Geração de dataset com LLM (15 USD de crédito inicial) | R\$ 0 |
| 🖥️ **Google Colab** (GPU T4 16GB) | Treinamento com Unsloth e LoRA                         | R\$ 0 |
| 💾 **Hugging Face Hub**            | Armazenamento e versionamento do dataset               | R\$ 0 |
| 🧪 **Weights & Biases**            | Logs e métricas de treino                              | R\$ 0 |

💡 *Ou seja, até o momento, não gastamos um único centavo


