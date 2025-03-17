## Kata Arquitecto Soluciones – Actualización Sistema de Acciones

- [Contexto del Problema](Contexto-de-Problema.md)
- [Arquitectura Propuesta](https://drive.google.com/file/d/1SQ38EJh01E6XO-mPCJv5T3u879xlCfvg/view?usp=sharing)


### Assumptions

1. **Conectividad e Integraciones Externas**  
   - Se asume que existen **VPNs/IPSec** o **AWS Direct Connect** para integrar de forma segura los servicios on-premise y de terceros (core bancario, CRM, Banco de la República, Bolsa de Valores de Colombia, etc.).  
   - Los endpoints externos (BVC, Sebra) exponen interfaces seguras (HTTPS/TLS) o un equivalente acorde al protocolo manejado, y se cuenta con **certificados digitales** válidos para cifrar la información en tránsito.
   - Todos los sistemas Legacy o de la organización, pueden ser accedidos mediante algún tipo de integración (ODBC / JDBC / Sockets TCP-IP / Web HTML / SOAP / REST / GraphQL / etc.)

2. **Orquestación y Manejo de Flujos**  
   - Se asume que los flujos de “Cargue Inicial”, “Validaciones” y “Confirmaciones” se pueden orquestar mediante colas SQS, sistemas de notificación SNS y AWS Lambdas o bien una equivalencia de **Step Functions** donde aplique. 
   - Se asume que se cuenta con esquema robusto de gestión de errores en cada paso (retries, circuit breaker, rate limit, compensaciones, etc.).  
   - Las Lambdas que reciben o procesan eventos asumen la **idempotencia** de las llamadas, guardando estado mínimo en bases de datos o metadatos para no repetir transacciones. (Request-ID UUID)

3. **Manejo de Errores y Retries**  
   - Se asume la configuración de **DLQ (Dead Letter Queue)** en SQS para redirigir mensajes con fallas recurrentes y ser validados posteriormente.  
   - Las funciones Lambda cuentan con **mecanismos de reintento exponencial** y logs específicos (nivel de severidad) en CloudWatch o en una solución centralizada de observabilidad.

4. **Observabilidad y Monitoreo**  
   - Se asume el uso de **Amazon CloudWatch** para capturar logs, métricas (latencias, conteo de errores, tasas de éxito) y generar alarmas (por ejemplo, cuando la tasa de fallas supera un umbral).  
   - **AWS X-Ray** se emplea para trazar llamadas entre Lambdas, servicios externos y diagnosticar cuellos de botella o tiempos de espera excesivos.  
   - Se asume un tablero (dashboard) en CloudWatch o Grafana que consolide KPIs críticos (tiempo promedio de procesamiento, número de liquidaciones diarias, etc.).

5. **Seguridad y Cumplimiento**  
   - Cada Lambda, Bucket S3 y demás componentes utiliza **roles y políticas IAM** con privilegio mínimo (principle of least privilege) y permisos granulares que permitan el acceso solo al recurso que necesita cada lambda, sqs, bucket, etc.
   - **AWS KMS** se emplea para cifrar en reposo (S3, RDS, Secrets Manager) y además se usa para cifrar las credenciales del Portal del Banco de la República.  
   - Se asume la **federación** con el directorio corporativo (AD) o un IdP (por ejemplo, AWS SSO/Okta) para la autenticación de usuarios que acceden a la consola o a APIs internas.  
   - **AWS WAF**/Shield están configurados en los endpoints expuestos (API Gateway, por ejemplo) para proteger de amenazas comunes (OWASP Top 10, DDoS, etc.).

6. **Persistencia y Manejo de Datos**  
   - Se asume que los datos críticos (por ejemplo, información de accionistas) residen en bases de datos gestionadas como **RDS (Aurora PostgreSQL)** y el almacenamiento de archivos (reportes, PDFs, CSV) en **S3** con versionado habilitado.  
   - El versionado en S3 y las **lifecycle policies** gestionan de forma automática la retención y archivado de información histórica.  
   - Para integraciones simples o colas internas, se usa **SQS**, asegurando durabilidad y **FIFO** donde aplique.

7. **Alta Disponibilidad y Escalabilidad**  
   - Las Lambdas y Step Functions se benefician de la **escalabilidad automática** inherente a AWS serverless, asumiendo que los límites de concurrencia están configurados de acuerdo a la demanda máxima prevista.  
   - Los servicios clave (por ejemplo, RDS) se despliegan con **Multi-AZ**, garantizando failover automático.  
   - Se asume un diseño multi-AZ en la VPC y uso de **Amazon Route 53** (o similar) para la resolución de nombres de los endpoints, minimizando la latencia y asegurando alta disponibilidad.

8. **Mecanismo de Autenticación y Autorización en las APIs**  
   - Las APIs expuestas por API Gateway utilizan **JWTs o IAM Auth** para autorizar solicitudes, exigiendo tokens con vigencia limitada y validación robusta.  
   - Se asume la implementación de **API Keys** o client certificates cuando se realicen integraciones con servicios externos de la BVC o Banco de la República que así lo requieran.

9. **Automatización e Infraestructura como Código**  
   - Se asume el uso de **Terraform** para describir la infraestructura (Lambdas, SQS, EventBridge, RDS, etc.), facilitando la trazabilidad y la reproducibilidad en distintos entornos (DEV, QA, PROD).  
   - Los despliegues siguen un pipeline de CI/CD (GitHub Actions) con pruebas automatizadas (unitarias, integración y seguridad).

10. **Compatibilidad con Sistemas Legados**  
   - Dado que aún se requiere interacción con un core bancario y repositorios on-premise, se asume que se mantiene **un canal de comunicación seguro** (VPN/Direct Connect) y un proceso de migración progresivo hacia servicios modernos en la nube.  
   - Se asume que, donde sea imposible prescindir de archivos planos (por temas regulatorios o históricos), se implementará un **flujo MFT** (Managed File Transfer) integrado con S3, Lambdas y AWS Transfer Family para automatizar su procesamiento.

11. **Política de Backups y DR**  
   - Se asume que **Aurora** y S3 están configurados con planes de respaldo automático (snapshots diarios, point-in-time restore) y replicación cruzada (CRR) si se requiere **Disaster Recovery** en otra región.  
   - Se realizan simulacros regulares de restauración para garantizar la validez de los respaldos y tiempos de RTO/RPO definidos.

12. **Confiabilidad de Terceros**  
   - Se asume que las entidades externas (BVC, Banco de la República) tienen alta disponibilidad y planes de contingencia propios.  
   - En caso de que un tercero falle o esté fuera de línea, se dispone de **mecanismos de reintento** y se notifica al usuario final sobre la indisponibilidad temporal de ese componente.
