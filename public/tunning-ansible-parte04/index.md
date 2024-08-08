---
title: Tunning Ansible - Forks - Pt04
date: 2024-08-08
spoiler: pfvr corte a minha cabeça
---
---

# Forks

## Motivação

```js eval
<p className="text-2xl font-sans text-purple-400 dark:text-purple-500">
  <i>Fala, rapeize!!!</i>
</p>
```

Por padrão, o Ansible executa 5 tarefas em paralelo. No Ansible, o termo “paralelo” refere-se ao número de tarefas que podem ser executadas simultaneamente em diferentes hosts do seu inventário. Essa configuração padrão é controlada por um parâmetro chamado forks.

Esse paralelismo permite que o Ansible gerencie vários hosts com eficiência e é um recurso fundamental para dimensionar a automação em muitos sistemas. No entanto, o número ideal de forks depende de vários fatores, incluindo o desempenho do nó da máquina executando o Ansible, a rede e a complexidade das tarefas. 

## Pontos Fortes e fracos do Uso de Forks

O principal benefício do uso de forks é o aumento significativo no desempenho. Ao executar tarefas em paralelo, você pode reduzir drasticamente o tempo total de execução de seus playbooks, especialmente quando lidando com um grande número de hosts. 

Permitem que o Ansible escale para gerenciar centenas ou até milhares de hosts. Isso é crucial para organizações com **infraestruturas de grande** porte ou em rápido crescimento.

Aumentar o número de forks aumenta o consumo de recursos na máquina de controle do Ansible. Isso pode levar a problemas de desempenho se o sistema não tiver recursos suficientes para lidar com o número de forks configurado.

## Como usar mais forks

### ansible.cfg

É possível mudar o número de forks no arquivo `ansible.cfg` 

```ini
[defaults]
forks = 10
```

### command line

Passar o parâmetro `-f N` ou `--forks N` na linha de comando muda a quantidade de forks, por exemplo `ansible-playbook playbook.yml -f 10`

## Considerações

1. A velocidade de execução.
2. O consumo de recursos na maquina que está sendo executada.
3. O consumo de recursos nas maquinas gerenciados.
4. A largura de banda da rede.
5. O número de conexões simultâneas aos nós gerenciados.

---

## Testes

Foi usado esse playbook, [fork.yaml](https://gist.githubusercontent.com/fbrunoviana/34317592049b371aeb31935a1667fd6d/raw/eac062a6734116065e37b32ba00085c69b5d6164/forks.yaml) para teste e foi passado o `-f N` com número de forks.

### Resultados dos Testes

![[img.png]]
### Cálculos de Melhora de Velocidade

Melhora = (Tempo Base - Tempo Novo) / Tempo Base * 100

1. Para 10 forks:
   Melhora = (176.26 - 102.59) / 176.26 * 100 = 41.96%

2. Para 20 forks:
   Melhora = (176.26 - 92.49) / 176.26 * 100 = 47.62%

## Análise

**Tempo Total de Execução:**
   - Com 5 forks (padrão): 2 minutos e 56.26 segundos (176.26 segundos)
   - Com 10 forks: 1 minuto e 42.59 segundos (102.59 segundos)
   - Com 20 forks: 1 minuto e 32.49 segundos (92.49 segundos)

**Melhora de Velocidade:**
   - Aumentar de 5 para 10 forks resultou em uma melhora de 41.96% no tempo de execução.
   - Aumentar de 5 para 20 forks resultou em uma melhora de 47.62% no tempo de execução.

## Conclusões

O aumento para 10 forks proporciona um ganho substancial de desempenho, reduzindo o tempo de execução em quase 42%, aumentar para 20 forks oferece uma melhoria adicional, mas o ganho é menos pronunciado comparado ao salto inicial de 5 para 10 forks.

### Recomendações

Para a maioria dos casos, configurar o Ansible para usar 10 forks pode ser uma ótima opção, oferecendo um ganho de desempenho.

Se o tempo de execução for crítico e o hardware permitir, o uso de 20 forks pode ser considerado, mas é importante avaliar se o ganho adicional justifica o aumento no uso de recursos, o uso pode ser acompanhado pelo `htop`.

