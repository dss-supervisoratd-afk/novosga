# Acesso aos dados do NovoSGA para o dashboard

Como obter os dados do NovoSGA de forma automatizada para alimentar um dashboard da AMN.

## Caminho recomendado: leitura direta do banco

O melhor caminho é ler direto do banco, com um usuário só de leitura. O NovoSGA já tem uma visão pronta para isso, a `view_atendimentos`, que junta as senhas do dia (`atendimentos`) com as já arquivadas (`historico_atendimentos`). Assim, a extração continua funcionando mesmo depois que o reset move as senhas para o histórico.

### Opções para obter os dados

| Caminho | Bom para | Limitação |
|---|---|---|
| **Banco (só leitura)** | Histórico completo, cruzamentos e indicadores | Depende da estrutura das tabelas, que pode mudar em atualizações |
| **Webhooks** | Eventos em tempo real (senha criada, chamada, finalizada...) | Só registra o que acontece depois de ativado, sem histórico |
| **API REST** | Consultas pontuais | O histórico só tem busca por `id`, `status` e `resolucao`; não foi identificado filtro por data, então não serve bem para extração em massa |

### Passo a passo

1. **Acesso:** criar um usuário `dashboard_ro` no Postgres, com permissão só de leitura.
2. **Camada de indicadores:** criar um schema `analytics` separado, com views SQL próprias. Se uma atualização do NovoSGA mudar alguma tabela, só essas views precisam de ajuste, e o dashboard não muda.
3. **Automação:** o dashboard consulta essas views diretamente. Um script Python agendado só compensa se houver volume grande ou necessidade de juntar dados de outros sistemas.
4. **Tempo real (opcional):** enviar webhooks para um serviço pequeno em Python, apenas se for necessário ver a fila ao vivo.

### Dados disponíveis por senha

- **Horários:** agendamento, chegada, chamada, início e fim.
- **Tempos já calculados:** espera, deslocamento, atendimento e permanência.
- **Classificação:** status, resolução, unidade, serviço, prioridade, local, atendente, quem fez a triagem e cliente.
- **Serviços realizados:** ficam na `view_atendimentos_codificados`, com o peso de cada um.

Com isso dá para acompanhar, por exemplo, tempo médio e percentis de espera, volume por hora e dia da semana, produtividade por atendente, taxa de não comparecimento e cumprimento de metas de espera (SLA).

### Cuidados

- **Fuso horário:** o módulo de relatórios converte a data local para UTC antes de consultar, então os horários parecem estar gravados em UTC. Isso ainda precisa ser confirmado com dados reais, porque um erro aqui deslocaria os gráficos por hora.
- **Dados de teste:** o banco local está vazio. Para validar os indicadores, é preciso gerar dados de teste ou usar uma cópia da base real.

## O acesso direto ao banco é necessário?

Para ler do banco, sim, mas é um acesso só de leitura: um usuário que consulta os dados e não consegue alterar nada.

**Na instalação local:** o acesso já existe, porque o banco roda no Docker (`localhost:5433`).

**Em produção, depende de quem controla o servidor:**

- **Se for da AMN:** basta criar o usuário de leitura e liberar a conexão apenas para a máquina do dashboard. Não é preciso expor o banco para a internet; se o dashboard ficar em outra rede, use VPN ou túnel SSH.
- **Se for de terceiros (TI externa ou fornecedor):** é preciso pedir esse acesso a quem administra. Vale pedir somente leitura, o que costuma facilitar a aprovação.
- **Se for um serviço hospedado sem acesso ao banco:** essa opção não está disponível, e restam as alternativas abaixo.

### Alternativas sem acesso direto

| Alternativa | O que é necessário | Limitação |
|---|---|---|
| **Webhooks** | Login de administrador no NovoSGA e um serviço que receba os eventos, acessível pelo servidor do NovoSGA | Só registra o que acontecer depois de ativado, e eventos perdidos deixam lacunas |
| **API REST** | Credenciais OAuth2 | Não foi identificado filtro por data no histórico, então a extração tende a ser lenta e incompleta. Ainda precisa ser testada |
| **Exportação periódica** | Alguém que gere e envie um dump ou CSV do banco | Os dados não ficam em tempo real, mas costuma ser fácil de aprovar |
| **Endpoint próprio no fork** | Poder implantar o fork em produção | Exige desenvolvimento, mas a API passa a entregar exatamente os dados necessários, sem acesso externo ao banco |

Se o acesso direto for negado, a sugestão é combinar a **exportação periódica** para o histórico com **webhooks** para os dados novos. Se a AMN implanta o próprio fork, o endpoint próprio também é uma boa saída.

## Pendências

1. Quais indicadores faltam nos relatórios atuais (os 3 a 5 mais importantes).
2. Se já existe uma instalação em produção com dados reais, onde ela roda e qual banco usa. Os testes foram feitos em Postgres, e as views teriam que ser ajustadas para MySQL.
3. Onde o dashboard vai rodar (Metabase, Grafana, Power BI ou solução própria). Sem preferência definida, a sugestão é começar pelo Metabase, que é gratuito, roda em Docker e dispensa programar os gráficos.
4. Quem administra o NovoSGA de produção da AMN.
