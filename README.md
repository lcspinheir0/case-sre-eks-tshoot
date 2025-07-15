# case-sre-eks-tshoot

## 1. Diagnóstico Inicial

1. **Verificar o alerta no Datadog**  
   - Confirmar latência média da API > 2 s nos últimos 10 min  
   - Verificar canal de alerta e dashboards de latência e erro

2. **Inspeção do cluster e pods**  
   ```bash
   kubectl get nodes                  # checar nós e recursos
   kubectl top nodes                  # consumo de CPU/mem dos nós
   kubectl get pod -n payment         # listar pods payment-api
   kubectl describe pod <pod> -n payment   # eventos (OOMKilled, CrashLoop)
   kubectl top pod -n payment         # uso CPU/mem de cada réplica
   ```

3. **Análise de logs das réplicas**  
   ```bash
   kubectl logs -n payment <pod> -c payment-api --since=15m
   ```
   - Buscar timeouts ao conectar no RDS e erros de conexão ou latência de queries

---

## 2. Possíveis Causas

- Pool de conexões no RDS esgotado (muitas threads concorrentes)  
- Query lenta ou manutenção/backup programado no RDS  
- Recursos insuficientes (OOM ou CPU throttling)  
- Latência de rede entre EKS e RDS (AZ diferente, Security Group mal configurado)  
- Problemas de GC ou deadlock na aplicação (picos de latência por pausas de coleta de lixo)

---

## 3. Ações Imediatas de Mitigação

1. **Scale-up temporário**  
   ```bash
   kubectl scale deployment payment-api -n payment --replicas=12
   ```

2. **Reiniciar réplicas com problemas**  
   ```bash
   kubectl rollout restart deployment payment-api -n payment
   ```

3. **Ajuste do pool de conexões**  
   - Aumentar limite de conexões na configuração da aplicação

4. **Isolar tráfego**  
   - Usar Blue/Green ou Feature Flags para redirecionar tráfego a uma versão estável

---

## 4. Ferramentas e Comandos

| Ferramenta    | Uso principal                                    |
| ------------- | ------------------------------------------------ |
| `kubectl`     | inspeção de pods, eventos, recursos, logs        |
| `kubectl top` | métricas de CPU/mem de pods e nós                |
| Datadog       | dashboards de latência, erros, RDS, SLIs custom  |
| AWS Console   | RDS Performance Insights, CloudWatch, logs EKS   |
| `aws eks` CLI | atualizar kubeconfig, verificar cluster          |

---

## 5. Comunicação do Incidente

- **Abertura:**  
  > Incidente Payment-API: latência média > 2 s há 10 min. Investigando; próximo update em até 30 min.

- **Updates a cada 30 min:**  
  - O que foi testado, ações em curso e resultados parciais  

- **Stakeholders não-técnicos:**  
  > “Estamos com lentidão no processamento de pagamentos. A equipe SRE escalou réplicas e reiniciou o serviço. Impacto no faturamento — mitigação em progresso.”

- **Encerramento inicial:**  
  > “Latência voltou a < 1 s. Serviço operacional. Em análise pós-incidente para identificar root cause.”

---

## 6. Post-Mortem e Melhores Práticas

### Resumo Executivo
- O que aconteceu, impacto, tempo de restauração

### Linha do Tempo Detalhada
- Horário do alerta, diagnósticos, ações e retomada

### Causa Raiz (Root Cause)
- Exemplo: pool de conexões insuficiente + pico de I/O no RDS

### Ações Corretivas e Preventivas

#### Curto Prazo
- Ajustar pool de conexões default para 100  
- Habilitar HPA (HorizontalPodAutoscaler) para payment-api

#### Médio Prazo
- Indexar tabelas críticas no RDS  
- Implementar circuit breaker (resilience4j) e fallback na aplicação  
- Reposicionar backups fora de horários de pico

#### Longo Prazo
- Adotar deploys Canary/Blue-Green para reduzir blast radius  
- Incluir sidecar OpenTelemetry para rastreamento de chamadas ao banco  
- Criar alertas proativos de conexões RDS e IOPS  

### Aprendizados e Documentação
- Atualizar runbooks: “Como responder alerta de latência da payment-api”  
- Treinar time de desenvolvimento em observabilidade e circuit breaker  
