---
title: Tunning Ansible - Disabling Fact Gathering - Pt02
date: 2024-07-18
spoiler: ansible, mais rapido, pt02
---
# Desabilitando Gathering Fact

## Motivação

```js eval
<p className="text-2xl font-sans text-purple-400 dark:text-purple-500">
  <i>Fala, rapeize!!!</i>
</p>
```

Como vimos em: [Tunning Ansible - SSH Host Key Checking - Pt01](https://devopslaboratory.com/tunning-ansible-parte01/ ) é real conseguir diminuir o tempo de execução de um playbook com apenas uma configuração, nesse artigo pretendo  decorrer sobre outro ponto que aumentara a rapidez do seu ansible-playbook.

---
## Gathering fact

O Gathering Fact é um processo pelo qual o Ansible coleta informações de forma automática sobre os sistemas gerenciados antes de executar alguma tarefa. Esses dados são conhecidos como "facts" e incluem uma ampla gama de informações sobre o sistema, como SO, rede, hardware, usuário, grupos, variáveis de ambiente e etc...

### Desabilitando os facts

Para agilizar o processo de run do playbook podemos desabilitar o `gathering fact` , isso implica em uma serie de limitações, se o playbook não usa nenhum fact é possível desabilitar sem muitas implicações, ou limitando como veremos mais tarde.

Para desabilitar, no playbook adicione como no exemplo:

```yaml
---
- hosts: web 
  gather_facts: False
```

Os testes serão feitos em uma, dez e oitenta e nove máquinas. 
#### Teste: Uma máquina

**gather_facts: Ativo**

```bash
time ansible-playbook gather_true.yaml -e hosts=servidor01

ansible-playbook gather.yaml -e hosts=servidor01  1.52s user 0.66s system 23% cpu 9.238 total
```

**gather_facts: Desativado**

```bash
time ansible-playbook gather_false.yaml -e hosts=servidor01

ansible-playbook gather.yaml -e hosts=servidor01  1.21s user 0.59s system 24% cpu 7.433 total
```

- **Ativo:** 9.238 segundos
- **Desativo:** 7.433 segundos
- **Ganho de Tempo:** 1.805 segundos
- **Percentual de Redução:** 19.54%

#### Teste: Dez máquinas

**gather_facts: Ativo**

```bash
time ansible-playbook gather_true.yaml -e hosts=grupoCom10

ansible-playbook gather.yaml -e hosts=grupoCom10  3.65s user 3.99s system 43% cpu 17.637 total
```

**gather_facts: Desativado**

```bash
time ansible-playbook gather_false.yaml -e hosts=grupoCom10

ansible-playbook gather.yaml -e hosts=grupoCom10  3.06s user 3.21s system 42% cpu 14.617 total
```

- **Ativo: 17.637 segundos**
- **Desativo: 14.617 segundos**
- **Ganho de Tempo: 3.02 segundos**
- **Percentual de Redução: 17.12%**

#### Teste: Oitenta e nove máquinas

**gather_facts: Ativo**

```bash
time ansible-playbook gather_true.yaml -e hosts=all

ansible-playbook gather.yaml -e hosts=all  29.02s user 31.73s system 19% cpu 5:15.16 total
```

**gather_facts: Desativado**

```bash
time ansible-playbook gather_false.yaml -e hosts=all

ansible-playbook gather.yaml -e hosts=all  22.23s user 25.91s system 20% cpu 3:58.10 total
```

- **Ativo: 315.16 segundos**
- **Desativo: 238.10 segundos**
- **Ganho de Tempo: 77.06 segundos**
- **Percentual de Redução: 24.45%**

#### Conclusão do primeiro teste 

![[img01.png]]
Os resultados indicam que, ao desabilitar o `gather_facts`, a redução no tempo de execução é proporcionalmente mais significativa com o aumento do número de máquinas. Embora o tempo economizado seja menor em termos absolutos para um pequeno número de máquinas, ele se torna substancialmente mais relevante em ambientes maiores, onde a coleta de facts em cada host pode se tornar um gargalo.
### Coletar Apenas Categorias Específicas de Facts

É possível usar o módulo `setup` com filtros para coletar apenas algumas categorias de facts, diretamente no playbook:

```yaml
- name: Coletar apenas informações de rede
  setup:
    filter: 'ansible_eth*'

- name: Coletar apenas informações de hardware e SO
  setup:
    filter: 'ansible_machine,ansible_distribution*'
```

### Factos de Cache

Existe um meio termo em relação a tudo isso, é possível habilitar o cache de facts. Isso armazena os facts coletados em um cache local no disco, que pode ser reutilizado:

Configure o cache de facts no seu arquivo `ansible.cfg`:

```ini
[defaults]
fact_caching = jsonfile
fact_caching_connection = /etc/ansible/fact_cache
fact_caching_timeout = 86400  # Cache por 24 horas
```

2. Use o cache em seus playbooks:

```yaml
- name: Playbook usando cache de facts
  hosts: all
  gather_facts: yes
```

### Manualmente

Dependendo do cenário é possível pegar a informação no inicio da execução e usa-lo conforme necessario. 

```yaml
---
- name: Instalar Nginx baseado no sistema operacional
  hosts: all
  gather_facts: False

  tasks:
    - name: Obter informações do sistema operacional
      raw: cat /etc/os-release
      register: os_release

    - name: Definir a variável do sistema operacional
      set_fact:
        os_family: >
          {% if 'ID_LIKE=debian' in os_release.stdout %}
          Debian
          {% elif 'ID_LIKE="rhel fedora"' in os_release.stdout %}
          RedHat
          {% endif %}

    - name: Instalar Nginx no Ubuntu
      apt:
        name: nginx
        state: present
      when: os_family == "Debian"

    - name: Instalar Nginx no Red Hat
      yum:
        name: nginx
        state: present
      when: os_family == "RedHat"

```

## Minha Conclusão

Dependendo do seu caso desabilitar o gathering facts pode resultar em uma redução de até 24.45% no tempo de execução, particularmente desativei em quase todos os meus playbooks por conta do ganho. Se o seu ambiente é padronizado, é uma prática interessante vai de você pesar a comodidade com o tempo de execução.
