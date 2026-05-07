---
tipo: referencia
titulo: Respostas-padrão para suporte (Discord / HubSpot)
origem: https://github.com/basedosdados/pipelines/wiki/Respostas-padr%C3%B5es-para-Discord-Hubspot
---
# Respostas-padrão para suporte

Templates de resposta para cenários recorrentes de contato de usuários no Discord e HubSpot. **Adapte o tom ao contexto**, mas mantenha o conteúdo factual.

> Esta é uma página de **referência**: você sabe o cenário, vem aqui buscar o texto. Para o fluxo de triagem ("chegou um relato, e agora?"), ver `processos/runbooks/` (a criar).

## Resumo

| Campo | Valor |
|---|---|
| Onde se aplica | Discord da BD, HubSpot, e-mail de suporte |
| Quando atualizar | Quando processo, link ou política mudar |
| Dono | Pessoa responsável por suporte na sprint |

## Cenários

### 1. Interesse em contribuir com a BD (genérico)

**Quando usar:** voluntário pergunta como pode ajudar, sem cenário específico.

**Pré-requisitos a comunicar:**
- Disponibilidade para reuniões semanais às 12h
- Familiaridade com Python (`requests`, `pandas`) ou R
- Conhecimentos básicos de Git

**Frentes possíveis:**
- Coleta e tratamento de dados
- Análises e divulgação de dados
- Criação de conteúdo (redes sociais, Medium)

**Próximo passo a indicar:** participar da reunião semanal na sala `sala-dados-1` do Discord, segunda a sexta às 12h.

---

### 2. Não conseguiu acessar dados por serem muito pesados

**Quando usar:** usuário relata timeout, custo alto ou impossibilidade de carregar dataset grande.

**Recomendações a passar:**
- Usar `WHERE` para restringir dados ao recorte desejado
- Limitar período específico ou região relevante
- Consultar partições para otimizar a query
- Especificar apenas colunas necessárias no `SELECT` (evitar `SELECT *`)
- Usar `read_sql` do pacote `basedosdados` ao trabalhar em Python, R ou Stata

**Recurso complementar:** linkar tutorial SQL em vídeo.

---

### 3. Procurando dados que não estão no datalake

**Quando usar:** usuário pergunta sobre dataset que não existe na BD.

**Mensagem-chave:** a BD apenas disponibiliza dados **já públicos e mapeados**. Para dados não presentes na plataforma, o caminho é contatar diretamente o órgão responsável pela produção do dado.

---

### 4. Mudar informações cadastrais pessoais

**Quando usar:** usuário pede alteração de dados pessoais em alguma base.

**Mensagem-chave:** a BD **não produz** os dados, apenas os disponibiliza. Modificações precisam ser solicitadas ao **órgão produtor** da base.

---

### 5. Reporte de problema/inconsistência nos dados

**Quando usar:** usuário relata erro, valor estranho ou inconsistência em dataset publicado.

**Procedimento:**
1. Solicitar a query SQL ou código que evidenciou o problema
2. Encaminhar internamente para análise
3. Manter o usuário informado sobre o andamento

**Canais aceitos:** Discord, e-mail.

---

### 6. Pergunta direta sobre informação específica

**Quando usar:** usuário pergunta um dado específico em vez de buscar na plataforma (ex: "quantos imóveis rurais existem em X?").

**Abordagem:** educar sobre o modelo da BD (somos plataforma, não consultoria) enquanto fornece orientação útil. Indicar:
- Dataset relevante na BD, se existir
- Cursos de SQL para iniciantes
- Fonte primária quando o dado não estiver na BD (ex: Cadastro de Imóveis Rurais da Receita Federal)

