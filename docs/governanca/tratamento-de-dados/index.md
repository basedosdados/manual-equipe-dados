# Tratamento de dados

O que a BD faz e o que não faz com o dado da fonte, e como a equipe garante isso no pipeline, no dbt e na revisão de PR.

## Conteúdo

### Entender

- [Princípios de tratamento de dados](explicacao/principios-de-tratamento.md) — o princípio geral (o dado o mais próximo possível da fonte), os quatro tipos de modificação permitidos e os compromissos com o dado publicado.

### Como fazer

- [Como tratar um código sem rótulo no dicionário](como-fazer/como-tratar-codigo-sem-rotulo.md) — o que fazer quando o `custom_dictionary_coverage` acusa um código que não tem rótulo: confirmar que vem da fonte, buscar o significado e, se não houver, incluir com o rótulo "Código <chave>: sem significado nas fontes oficiais (nota da BD)" e pedir via LAI.

### Referências

- [Práticas de tratamento](referencia/praticas-de-tratamento.md) — as regras de engenharia verificadas na revisão de PR.
