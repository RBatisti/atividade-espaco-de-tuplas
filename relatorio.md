# Relatório — Atividade Prática: Espaço de Tuplas com Apache River

**Disciplina:** Computação Distribuída
**Membro:** Rogerio Batisti Junior
**Data:** 19/11/2026

---

## Nível 0 — Rodar e observar

### Observação inicial

1. Qual serviço aparece nos logs primeiro? Por que ele precisa existir antes dos outros?

> _Resposta:_ O reggie. Ele é quem permite que os outros serviços se achem, o javaspaces se registra nele e o produtor e o consumidor perguntam a ele onde está o espaço. Se ele não subir primeiro, ninguém encontra ninguém.

2. O produtor menciona o nome ou o endereço do consumidor em algum momento? O consumidor menciona o produtor?

> _Resposta:_ Não. Nenhum dos dois cita o outro. Os dois só sabem que existe um espaço de tuplas.

3. O consumidor começa a processar tarefas antes que o produtor termine de depositar todas? O que isso diz sobre como os dois se coordenam?

> _Resposta:_ Sim. O consumidor começa a trabalhar enquanto o produtor ainda está depositando tarefas. Eles não esperam um pelo outro, cada um só usa o espaço, e isso basta para coordenar os dois.

---

### Experimento de desacoplamento temporal

4. O consumidor encontrou as tarefas mesmo sendo iniciado depois que o produtor já havia encerrado? O que isso demonstra?

> _Resposta:_ Sim. As tarefas continuaram lá, guardadas no espaço. Isso mostra o desacoplamento temporal, produtor e consumidor não precisam estar ligados ao mesmo tempo.

5. Em comunicação direta via socket, seria possível esse comportamento? Por quê não?

> _Resposta:_ Não. Com socket, os dois lados precisam estar ativos ao mesmo tempo. Se o consumidor estiver desligado, o produtor não consegue enviar, e não existe nada no meio para guardar a mensagem.

---

## Nível 1 — Inspecionar

### 1.1 As três operações

Preencha a tabela com base no que você observou nos logs:

| Operação River | Equivalente Linda | O que ela faz? | Bloqueia quando não encontra correspondência? | Altera o estado do espaço? |
|---------------|-------------------|----------------|----------------------------------------------|---------------------------|
| `write(entry)` | `OUT` | Coloca uma entrada no espaço | Não | Sim (adiciona) |
| `take(template)` | `IN` | Pega uma entrada e a remove | Sim, até o timeout | Sim (remove) |
| `read(template)` | `RD` | Só olha, deixa a entrada lá | Sim, até o timeout | Não |

---

### 1.2 O papel do `reggie`

1. Quando o `reggie` caiu, os serviços que já estavam conectados ao espaço continuaram funcionando? Por quê?

> _Resposta:_ Sim, continuaram funcionando. O reggie só é necessário para achar o espaço no começo. Quem já tinha achado continua falando direto com o espaço.

2. O que aconteceria com um produtor ou consumidor que tentasse iniciar enquanto o `reggie` estivesse fora do ar?

> _Resposta:_ Não conseguiria achar o espaço. Ficaria mostrando "Aguardando espaço... (n/20)" e, se o reggie não voltasse, desistiria depois das 20 tentativas.

3. Qual sistema moderno cumpre papel equivalente ao `reggie` em uma arquitetura de microsserviços?

> _Resposta:_ Consul, um lugar onde os serviços se registram e são encontrados pelo nome, sem IP fixo.

---

### 1.3 Desacoplamento espacial

1. O produtor tem qualquer informação sobre quantos consumidores existem?

> _Resposta:_ Não.

2. O consumidor tem qualquer informação sobre quem produziu a tarefa que ele retirou?

> _Resposta:_ Não.

3. Como produtor e consumidor se coordenam se não se conhecem?

> _Resposta:_ Pelo espaço de tuplas. O produtor escreve a tarefa e o consumidor retira uma que combine com o que ele procura. O reggie só ajuda a achar o espaço, não participa da troca de tarefas.

---

### 1.4 Comportamento de bloqueio

1. O que o consumidor fez enquanto o espaço estava vazio?

> _Observado:_ Ficou parado no `take()`, esperando uma tarefa aparecer.

2. Quando o produtor depositou a primeira tarefa, o que aconteceu imediatamente?

> _Observado:_ Assim que a primeira tarefa foi escrita, o `take()` terminou e o consumidor começou a processá-la na hora.

3. Esse comportamento tem nome no modelo Linda. Qual é e por que ele é útil em sistemas distribuídos reais?

> _Resposta:_ É o IN bloqueante do Linda. É útil porque o consumidor não precisa ficar perguntando de tempos em tempos se tem trabalho novo, ele espera e acorda assim que algo chega.

---

### 1.5 Escalabilidade horizontal

1. Uma mesma tarefa foi processada por dois consumidores ao mesmo tempo?

> _Observado:_ Não. Cada tarefa foi processada por um único consumidor, porque quem faz o `take()` remove a tarefa do espaço e mais ninguém a pega.

2. O produtor precisou ser modificado para suportar dois consumidores?

> _Resposta:_ Não.

3. Esse comportamento tem um nome em arquitetura de sistemas. Qual é?

> _Resposta:_ Escalabilidade horizontal (também chamado de consumidores concorrentes), para dar conta de mais trabalho, basta subir mais consumidores, sem mexer no produtor.

---

## Nível 2 — Modificar

### 2.1 Modificação A — Prioridade de tarefas

1. As tarefas de prioridade alta foram processadas antes das de prioridade baixa? Cole um trecho dos logs que evidencie isso:

> _Resposta:_ Sim. As tarefas de prioridade 1 (ids 2, 4 e 5) foram retiradas primeiro, e só depois as de prioridade 2 (ids 1 e 3).

```
produtor-1    | [PRODUTOR] Espaço encontrado via lookup.
produtor-1    | [PRODUTOR] write: TaskEntry{id=1, tipo="calcular", prioridade=2}
produtor-1    | [PRODUTOR] write: TaskEntry{id=2, tipo="calcular", prioridade=1}
consumidor-1  | [CONSUMIDOR-1] take: TaskEntry{id=2, tipo="calcular", prioridade=1}
produtor-1    | [PRODUTOR] write: TaskEntry{id=3, tipo="calcular", prioridade=2}
consumidor-1  | [CONSUMIDOR-1] Processamento concluído: tarefa 2
produtor-1    | [PRODUTOR] write: TaskEntry{id=4, tipo="calcular", prioridade=1}
consumidor-1  | [CONSUMIDOR-1] take: TaskEntry{id=4, tipo="calcular", prioridade=1}
produtor-1    | [PRODUTOR] write: TaskEntry{id=5, tipo="calcular", prioridade=1}
consumidor-1  | [CONSUMIDOR-1] Processamento concluído: tarefa 4
consumidor-1  | [CONSUMIDOR-1] take: TaskEntry{id=5, tipo="calcular", prioridade=1}
produtor-1    | [PRODUTOR] Todas as tarefas depositadas. Encerrando.
consumidor-1  | [CONSUMIDOR-1] Processamento concluído: tarefa 5
consumidor-1  | [CONSUMIDOR-1] Nenhuma tarefa de alta prioridade disponível. Buscando qualquer tarefa...
consumidor-1  | [CONSUMIDOR-1] take: TaskEntry{id=1, tipo="calcular", prioridade=2}
consumidor-1  | [CONSUMIDOR-1] Processamento concluído: tarefa 1
consumidor-1  | [CONSUMIDOR-1] Nenhuma tarefa de alta prioridade disponível. Buscando qualquer tarefa...
consumidor-1  | [CONSUMIDOR-1] take: TaskEntry{id=3, tipo="calcular", prioridade=2}
consumidor-1  | [CONSUMIDOR-1] Processamento concluído: tarefa 3
```

2. O produtor precisou ser modificado para que isso funcionasse?

> _Resposta:_ Não.

3. Como o consumidor consegue selecionar apenas tarefas de uma prioridade específica? Qual mecanismo do espaço de tuplas torna isso possível?

> _Resposta:_ Pelo template. O consumidor pede `TaskEntry(null, null, 1)`, os campos `null` aceitam qualquer valor e o campo prioridade só aceita 1. O próprio espaço procura uma tarefa que combine, então o consumidor não precisa olhar todas as tarefas. Isso é a busca por conteúdo (template matching) do espaço de tuplas, e não tem a ver com o reggie.

---

### 2.2 Modificação B — Serviço monitor

1. Qual operação você usou no monitor — `read()` ou `take()`? Por quê a outra seria problemática?

> _Resposta:_ Usei `read()`. Ele só olha a tarefa e deixa ela no espaço. Com `take()`, o monitor removeria as tarefas e o consumidor nunca receberia nenhuma.

2. Como você contou as tarefas pendentes usando apenas `read()`? Que limitação isso revela?

> _Resposta:_ Fiz um `read()` para cada id, de 1 a 100, e contei quantos acharam uma tarefa. A limitação é que preciso saber quais ids existem, uma tarefa com id acima de 100 não seria contada.

3. Por que um espaço de tuplas puro não tem operação `count()`? O que seria necessário adicionar ao modelo para suportá-la?

> _Resposta:_ Porque o espaço só tem `write`, `read` e `take`, nada que conte. Além disso, o espaço foi feito para trocar tarefas (colocar e retirar), não para consultar o estado. Para isso, seria preciso adicionar uma operação nova, como `count()`.

Saída do monitor na mesma execução (a contagem cai sem atrapalhar o consumidor):

```
[MONITOR] Tarefas pendentes no espaço: 2
[MONITOR] Tarefas pendentes no espaço: 2
[MONITOR] Tarefas pendentes no espaço: 1
[MONITOR] Tarefas pendentes no espaço: 0
```

Cole o trecho do `Monitor.java` que você completou:

```java
private static int contarTarefas(JavaSpace space) throws Exception {
    int count = 0;

    for (int id = 1; id <= ID_MAXIMO; id++) {
        TaskEntry template = new TaskEntry(id, null, null);
        TaskEntry encontrada = (TaskEntry) space.read(template, null, JavaSpace.NO_WAIT);
        if (encontrada != null) {
            count++;
        }
    }

    return count;
}
```

---

## Observações livres

_(Comportamentos inesperados, erros encontrados, dificuldades técnicas — descreva o que aconteceu e como você resolveu)_

>

---

## Dúvida para a próxima aula

_(Formule uma pergunta substantiva que surgiu durante a atividade)_

>
