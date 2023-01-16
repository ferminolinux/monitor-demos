# prometheus-grafana-example

Um exemplo simples de monitoramento de uma aplicação python usando prometheus, grafana e o alertmanager tudo de modo containerizado.

## Atenção

---
Algumas coisas precisam ser ajustadas para executar testes neste projeto, essas questões serão detalhadas abaixo:

### AlertManager

Antes de tentar executar esse projeto, certifique-se de alterar os valores das propriedades  **slack_api_url** e **channel** para:&nbsp;sua url do slack e o nome do seu canal respectivamente.

```yaml

global:
  # Opções globais para todos os receivers do slack
  slack_api_url:  'SUA URL DO SLACK'
  resolve_timeout: 5m

route:
  receiver: 'slack-notifications'
  group_by: 
    - alertname
    - severity

receivers:
- name: 'slack-notifications'
  slack_configs:
  - send_resolved: true
    channel: 'SEU CANAL DO SLACK'
    icon_url: 'https://avatars3.githubusercontent.com/u/3380462'
    username: 'AlertManager'
    title: '[ {{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
    text: '*Alert:* {{ .CommonAnnotations.summary }} - {{ .CommonLabels.severity }} {{ printf "\n" }}*Description:* {{ .CommonAnnotations.description }}' 

```

### Grafana

Para visualizar o dashboard do grafana será necessário importa-lo, o JSON que contém as informações para isso está presente no diretório 'dashboard'.  

## Execução

---

### Docker

Para executar com o docker você pode utilizar o docker-compose juntamente com o arquivo docker-compose.yaml presente neste repositório, através do comando:

```bash
docker compose up
```

### Podman

Caso você utilize o podman e queira executar este projeto a maneira mais simples de faze-lo é através do uso de pods, dessa forma você pode agrupar e gerenciar todos os containers  de uma vez(como faria com o docker-compose).


Vamos começar criando a rede para isso
```bash
podman network create monitor
```

Em seguida criamos o nosso pod
```bash
podman pod create monitor
```

Agora vamos criar os containers, lembrando que containers que vão alterar arquivos do host através dos volumes do tipo 'bind' precisam ter acesso privilegiado, para isso adicione a opção '--privileged'.
```bash
# container da aplicação
podman create --name jjba -i \
  -p 8080:5000 \
  -e "REQUEST_LATENCY=0" \
  --network monitor \
  --pod monitor \
  ghcr.io/ferminolinux/jojoba:latest

# container do prometheus
podman create --name prometheus -i \
  -p 9090:9090 \
  --network monitor \
  --pod monitor \
  --privileged \
  --mount type=bind,src=prometheus/prometheus.yml,dst=/usr/local/etc/prometheus/prometheus.yml \
  --mount type=bind,src=prometheus/rules/,dst=/usr/local/etc/prometheus/rules/ \
  ghcr.io/ferminolinux/prometheus:latest

# container do alertmanager
podman create --name alertmanager -i \
  -p 9093:9093 \
  --network monitor \
  --pod monitor \
  --privileged \
  --mount type=bind,src=alertmanager/alertmanager.yml,dst=/usr/local/etc/alertmanager/alertmanager.yml \
  ghcr.io/ferminolinux/alertmanager:latest

# container do grafana
podman create --name grafana -i \
  -p 3000:3000 \
  --network monitor \
  --pod monitor \
  docker.io/grafana/grafana:latest
```

Após a execução dos comandos acima você pode gerenciar os containers através do pod com o comando 'podman pod'.

```bash
# Inicia o pod monitor
podman pod start monitor
# Para o pod monitor
podman pod stop monitor
# Remove o pod monitor 
podman pod rm monitor
# Reinicia o pod monitor para aplicar novas configurações
podman pod kill --signal=SIGHUP monitor
```

