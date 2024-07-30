---
title: Tunning Ansible - Ansible e SSH  - Pt03
date: 2024-07-24
spoiler: pipelining = True
---
# Otimizando Conexões SSH no Ansible
## Motivação

```js eval
<p className="text-2xl font-sans text-purple-400 dark:text-purple-500">
  <i>Fala, rapeize!!!</i>
</p>
```

Separei esse pedaço da internet para mostrar duas opções de otimização do ssh com ansible, como elas são bem semelhantes juntei em um post só.

## Parâmetros de Otimização

Ansible usa SSH para conectar-se aos nós gerenciados. Você pode otimizar o uso do SSH configurando os seguintes parâmetros no arquivo de configuração do Ansible `/etc/ansible/ansible.cfg` ou no `ansible.cfg` local:

### `ControlMaster=auto`

A opção `ControlMaster` permite que múltiplas sessões SSH simultâneas compartilhem uma única conexão, permitindo que sessões SSH reutilizem a conexão da primeira sessão, economizando o tempo necessário para estabelecer novas conexões. Isso significa que a primeira tarefa demorará mais para ser executada, mas as tarefas que vem depois serão executadas mais rapidamente.

```ini
[ssh_connection]
ssh_args = -o ControlMaster=auto
```
### ControlPersist

Já o `ControlPersist` define quanto tempo a conexão mestre permanece aberta, aguardando novas sessões após o término da última sessão. Configurá-la para 60 segundos significa que a conexão mestre SSH permanecerá aberta por 60 segundos após o fechamento da última sessão. Se uma nova sessão SSH começar dentro desse período, ela pode rapidamente usar a conexão existente. Se nenhuma nova sessão começar dentro de 60 segundos, a conexão mestre se fechará.

```ini
[ssh_connection]
ssh_args = -o ControlPersist=60s
```
### Pipelining

A opção `pipelining` habilita um recurso que otimiza como comandos e tarefas são executados em servidores remotos, particularmente em relação a como os módulos do Ansible são transferidos e executados.

Sem pipelining, o Ansible geralmente funciona nos seguintes passos ao executar um módulo na maquina remota:

1. Conectar: Estabelecer uma conexão SSH com o host remoto.
2. Transferir: Copiar o código do módulo (um script temporário) para o host remoto.
3. Executar: Executar o script na maquina remota.
4. Limpar: Remover o script que foi executado remotamente.

Cada um desses passos envolve operações SSH separadas, o que pode gastar mais tempo para executar um playbook com muitas tarefas.

O pipelining modifica esse processo para reduzir o número de operações de rede. Em vez de transferir o módulo como um script independente para o host remoto, o código do módulo é enviado pela conexão SSH e executado diretamente no shell remoto. Assim funciona com pipelining:

1. Conectar: Estabelecer uma conexão SSH com o host remoto.
2. Executar: Enviar o código do módulo pela conexão SSH e executá-lo diretamente no shell.

Para ativar no seu ansible no arquivo `ansible.cfg` adicione:

```ini
[ssh_connection]
pipelining = True
```
#### Limitações do Pipelining

No entanto, há uma limitação com esse recurso. O pipelining frequentemente entra em conflito com a elevação de privilégios (usando `become` no Ansible), que é necessária para tarefas que requerem privilégios mais altos. Isso ocorre porque, com pipelining, o código do módulo é executado no contexto do usuário SSH, e trocar de usuário no meio de uma operação de pipelining pode ser problemático ou inseguro em alguns ambientes, então cautela e que isso seja pesado na sua tomada de decisão.

## Testando com 1, 10 e 89 Máquinas

### Teste: Uma Máquina

*Controle de Conexão Habilitado:*

```bash
time ansible-playbook gather.yaml -e hosts=server01

ansible-playbook gather.yaml -e hosts=server01  2.47s user 1.32s system 22% cpu 16.865 total
```

*Controle de Conexão Desabilitado:*

```bash
time ansible-playbook gather.yaml -e hosts=server01

ansible-playbook gather.yaml -e hosts=server01  1.49s user 0.64s system 26% cpu 8.190 total
```

**Comparação**

- **Habilitado**: 16.865 segundos
- **Desabilitado**: 8.190 segundos
- **Ganho de Tempo**: 8.675 segundos
- **Percentual de Redução**: 51.46%

### Teste: Dez Máquinas

*Controle de Conexão Habilitado:*

```bash
time ansible-playbook gather.yaml -e hosts=grupoCom10

ansible-playbook gather.yaml -e hosts=grupoCom10  6.91s user 7.48s system 58% cpu 24.675 total
```

*Controle de Conexão Desabilitado:*

```bash
time ansible-playbook gather.yaml -e hosts=grupoCom10

ansible-playbook gather.yaml -e hosts=grupoCom10  4.58s user 4.11s system 54% cpu 15.977 total
```

**Comparação**

- **Habilitado**: 24.675 segundos
- **Desabilitado**: 15.977 segundos
- **Ganho de Tempo**: 8.698 segundos
- **Percentual de Redução**: 35.26%

### Teste: Oitenta e Nove Máquinas

*Controle de Conexão Habilitado:*

```bash
time ansible-playbook gather.yaml -e hosts=all

ansible-playbook gather.yaml -e hosts=all  32.84s user 31.45s system 18% cpu 5:39.37 total
```

*Controle de Conexão Desabilitado:*

```bash
time ansible-playbook gather.yaml -e hosts=all

ansible-playbook gather.yaml -e hosts=all  42.81s user 23.03s system 17% cpu 6:24.36 total
```

**Comparação**

- **Habilitado**: 339.37 segundos
- **Desabilitado**: 384.36 segundos
- **Ganho de Tempo**: 44.99 segundos
- **Percentual de Redução**: 11.71%

## Minha Conclusão

As limitações dessas configurações devem ser consideradas, a redução de tempo significativa deve ficar do outro lado da balança, em alguns momentos perdi a conexão com o host o parâmetro `-o ConnectionAttempts=3` pode ser usado se a sua rede apresenta alguma instabilidade, ele permite múltiplas tentativas, com você aumenta as chances de estabelecer uma conexão bem-sucedida.

