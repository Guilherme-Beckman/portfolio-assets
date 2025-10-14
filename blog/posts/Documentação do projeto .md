![[Pasted image 20251006152233.png]]


---

## 🧭 Visão Geral da Arquitetura

A infraestrutura atual é composta por um **proxy reverso (Nginx)**, um **API Gateway (Kong)**, um **Service Discovery (Consul)** e os **bancos de dados (Postgres e MySQL)**.  
O objetivo é centralizar o controle de acesso, roteamento, segurança e descoberta de serviços dentro de uma **rede Docker isolada** (`kong-net`).

O fluxo de comunicação pode ser visualizado da seguinte forma:

```
User Request → Nginx → Kong Gateway → Consul → Services
```

---

## ⚙️ Fluxo da Requisição

1. **Entrada via Nginx**
    
    - O usuário realiza uma requisição HTTP ou HTTPS para o domínio configurado.
        
    - O Nginx atua como o ponto de entrada (reverse proxy) e faz o redirecionamento de HTTP → HTTPS.
        
    - Após isso, ele encaminha a requisição para o **Kong Gateway**, dentro da mesma rede Docker.
        
2. **Kong Gateway (API Gateway)**
    
    - O Kong mantém uma lista de **rotas** configuradas (`route1`, `route2`, `route3` etc.), cada uma apontando para um serviço lógico.
        
    - Quando uma requisição chega, o Kong identifica a rota correspondente e consulta o **Consul** para descobrir o IP e a porta do serviço real.
        
    - Após receber a informação do Consul, ele encaminha a requisição para o container de destino (por exemplo, o serviço `pantanal-app` ou outro microserviço).
        
3. **Consul (Service Discovery)**
    
    - O Consul atua como o **resolvedor DNS** interno da arquitetura.
        
    - Ele é responsável por registrar e armazenar os IPs e portas dos serviços disponíveis.
        
    - O Kong usa o endereço `172.18.0.10:8600` (porta DNS do Consul) para resolver os serviços via nomes (`service.consul`).
        
4. **Resposta ao Usuário**
    
    - O serviço retorna a resposta para o Kong.
        
    - O Kong encaminha de volta para o Nginx.
        
    - O Nginx devolve o resultado final ao cliente.
        

---

## 🐳 Estrutura do `docker-compose.yml`

O arquivo `docker-compose.yml` define **somente a infraestrutura base do servidor**, sem incluir os microserviços de negócio. Ele contém os seguintes componentes:

|Serviço|Função|IP|Porta(s)|Observações|
|---|---|---|---|---|
|**nginx**|Ponto de entrada HTTPS/HTTP, proxy reverso|`172.18.0.12`|80, 443|Redireciona e encaminha para o Kong|
|**kong-gateway**|API Gateway principal|Automático|8000–8445|Roteamento, autenticação e controle de tráfego|
|**kong-database**|Banco de dados PostgreSQL usado pelo Kong|`172.18.0.11`|5432|Armazena configurações e rotas|
|**kong-migrations**|Executa migrações do banco do Kong|—|—|Roda antes do `kong-gateway` iniciar|
|**consul**|Serviço de descoberta (DNS interno)|`172.18.0.10`|8500, 8600|Resolução de nomes e status de serviços|
|**mysql**|Banco do sistema `Pantanal 3D`|`172.18.0.14`|3306|Somente para a aplicação do Pantanal|
|**pantanal-app**|Aplicação Laravel (Pantanal 3D)|`172.18.0.15`|8080|Conectada ao MySQL; será extraída futuramente|

---

## 🔄 Próximos Passos

1. **Criar um “Service Registrar”**
    
    - Um serviço leve (pode ser em Go, Node.js, ou Java Spring WebFlux) que faça registro automático de novos serviços no **Consul**.
        
    - Ele pode monitorar containers em execução (via Docker SDK ou Consul API) e registrar/desregistrar dinamicamente.
        
2. **Remover `pantanal-app` deste compose**
    
    - O `docker-compose.yml` atual deve conter **somente a infraestrutura base** (Nginx, Kong, Consul, bancos).
        
    - O `pantanal-app` deve ser movido para um arquivo separado (`docker-compose.services.yml`), dedicado a aplicações.
        
3. **Centralizar DNS no Consul**
    
    - Garantir que todos os serviços se registrem com nomes como `service-name.service.consul`, permitindo que o Kong resolva automaticamente via DNS interno (`KONG_DNS_RESOLVER`).
        
4. **Automatizar configuração de rotas do Kong**
    
    - Criar um script que registre rotas dinamicamente no Kong ao detectar novos serviços registrados no Consul.
        

---

