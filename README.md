# case-sre-eks-tshoot
case-sre-eks-tshoot

Verificar o Datadog, se a latencia média da api permanece maior que 2s nos últimos 10 minutos
Verificar canal de alerta, tela de dashboards de latência e erro
verificar estado do cluster e pods:
  kubectl get nodes                  # checar nós e recursos
  kubectl top nodes                  # consumo CPU/mem nodes
  kubectl get pod -n payment         # listar pods payment-api
  kubectl describe pod <pod> -n payment   # eventos recentes (OOMKilled, CrashLoop)
  kubectl top pod -n payment         # CPU/mem de cada réplica
  
  kubectl logs -n payment <pod> -c payment-api --since=15m  #ver logs
  Buscar timeouts ao conectar no RDS e/ou erros de conexão, latência de querys
  


Possiveis causas:
  sem conexões no RDS, muitas threads concorrentes, pool de conexões esgotado
  Query lenta, manutenção ou backup programado
  Recursos insuficientes OOM ou CPU
  Latência entre EKS e RDS (AZ diferente, SG mal configurado)
  Problema de GC ou deadlock na aplicação, picos de latência por pausas de coleta de lixo ou threads bloqueadas

Possiveis ações para correção:
  #Scale-up ( temporário)
  kubectl scale deployment payment-api -n payment --replicas=12

  #reiniciar replicas com problemas
  kubectl rollout restart deployment payment-api -n payment

  #Ajustar pool de conexões
  #Isolar tráfego


| Ferramenta    | Uso principal                                    |
| ------------- | ------------------------------------------------ |
| `kubectl`     | inspeção de pods, events, recursos, logs         |
| `kubectl top` | métricas de CPU/mem de pods e nodes              |
| Datadog       | dashboards de latência, erros, RDS, custom SLI   |
| AWS Console   | Performance Insights (RDS), CloudWatch, EKS logs |
| `aws eks` CLI | recriar kubeconfig, verificar cluster            |


Comunicação do incidente:
  Incidente Payment-API: latência média > 2 s há 10 min. Estou investigando; próximos updates em até 30 min.
  Status updates a cada 30 min:
    O que foi testado, ações em curso e resultados parciais
  Stakeholders não-técnicos (via e-mail ou reunião rápida)
    “Estamos enfrentando lentidão no processamento de pagamentos. Equipe SRE escalou réplicas e reiniciou serviço. Há impacto no faturamento — mitigação em progresso.”

  Encerramento inicial:
  “Métrica de latência voltou a < 1 s. Serviço está operando normalmente. Em análise pós-incidente para root-cause.”

  
Resumo executivo:
  O que aconteceu, impacto, tempo de restauração

Linha do tempo detalhada:
  Horário do alerta, diagnósticos, ações e retomada

Causa raiz (root cause):
  Por exemplo: pool de conexões insuficiente + pico de I/O no RDS

Ações corretivas e preventivas:
  Curto prazo:
    Ajustar pool de conexões default para 100
    Habilitar auto-scaling automático de pods via HorizontalPodAutoscaler
  Médio prazo:
    Indexação de tabelas críticas no RDS
    Implementar circuit breaker (resilience4j) e fallback na aplicação
    Mover backups fora de horários de pico
  Longo prazo:
    Introduzir canary or blue-green deploys para reduzir blast radius
    Padrão de sidecar do OpenTelemetry para rastreamento de chamadas ao banco
    Criação de alertas proativos de conexões RDS e IOPS
  Aprendizados e documentação:
    Atualizar runbooks: “Como responder alerta de latência da payment-api”
    Treinar time de dev em práticas de observabilidade e circuit breaker

