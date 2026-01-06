# 🛡️ AWS Security & Governance Project – ECS Fargate & Inspector

## 📖 Descrizione

Questo progetto mostra come progettare e distribuire un ambiente **AWS sicuro e gestito** per un’applicazione containerizzata in **ECS Fargate**, integrando:

- VPC dedicata con subnet pubbliche e private  
- Application Load Balancer pubblico + **CloudFront** come front-end  
- Container su **ECS Fargate** in subnet private  
- Database **RDS** cifrato con **KMS**  
- Bucket **S3** per log/backup cifrato  
- Controlli di sicurezza con **Security Hub, Inspector, GuardDuty, Macie, WAF, Shield**

L’obiettivo è avere una piccola applicazione di esempio, ma inserita dentro un’architettura da “mondo reale”, centrata su **Security & Governance**.

---

## 🎯 Obiettivi del progetto

- 🏗️ Progettare una **VPC** segmentata (public/private subnet, IGW, NAT)  
- ☁️ Eseguire l’applicazione in **ECS Fargate** in subnet private  
- 🌐 Esporre l’app in modo sicuro tramite **ALB pubblico + CloudFront + WAF**  
- 🗄️ Gestire i dati con **RDS** e **S3** con cifratura KMS  
- 🔐 Gestire segreti e chiavi con **Secrets Manager** e **KMS**  
- 🧩 Abilitare servizi di **security posture**: Security Hub, Inspector, GuardDuty, Macie  
- 📊 Centralizzare logging e visibility con **CloudWatch Logs**  

---

## 🧠 Architettura del progetto

L’architettura è organizzata su più livelli.

### 🌍 Livello Networking & Perimetro

- **VPC dedicata** `S&G Project-vpc` (`10.0.0.0/16`)
  - 2 **subnet pubbliche** (us-east-1a, us-east-1b)
  - 2 **subnet private** (us-east-1a, us-east-1b)
- **Internet Gateway**  
  → collega le subnet pubbliche a Internet
- **NAT Gateway** in subnet pubblica  
  → permette alle subnet private di uscire su Internet (ECR, update, servizi AWS)
- **Security Group** dedicati:
  - SG per ALB pubblico (HTTP/HTTPS da Internet)
  - SG per ECS Fargate (porta 8080 solo dall’ALB)
  - SG per RDS (solo da ECS e bastion)
  - SG per Bastion host

### 🧾 Livello Applicativo (Compute & Container)

- **Amazon ECS (Fargate)**
  - Cluster ECS dedicato
  - Service Fargate con almeno 1 task in subnet private
  - Task definition con container in ascolto su **porta 8080**
  - Log inviati a **CloudWatch Logs**
- **Amazon ECR**
  - Repository privato per l’immagine Docker dell’app
  - Scansione delle immagini abilitata (`scan_on_push = true`)
- **Bastion Host (EC2)**
  - Istanza in subnet pubblica
  - Gestita preferibilmente via **SSM Session Manager**
  - Punto di accesso amministrativo verso le risorse interne

### 🌐 Livello Ingresso & Edge

- **Application Load Balancer (ALB pubblico)**
  - Scheme `internet-facing`
  - Listener HTTP :80 → Target Group (porta 8080, target type `ip` → Fargate)
  - Health check sull’endpoint `/`
- **Amazon CloudFront**
  - Distribuzione con origin = DNS dell’ALB pubblico
  - Usata come front-end pubblico per l’applicazione
- **AWS WAF**
  - Web ACL associata alla distribuzione CloudFront
  - Managed Rules (CommonRuleSet, ecc.) per protezione da SQLi/XSS, bot, pattern comuni
- **AWS Shield Standard**
  - Protezione DDoS di base su ALB e CloudFront (automaticamente attivo)

### 🗄️ Livello Dati & Storage

- **Amazon RDS**
  - Database in subnet private
  - Cifrato a riposo con **AWS KMS**
  - Accesso consentito solo da ECS e bastion
- **AWS Secrets Manager**
  - Segreto dedicato per le credenziali RDS
  - Utilizzabile da ECS/Fargate tramite IAM role (no password hardcodate)
- **Amazon S3**
  - Bucket per log e backup
  - Cifratura **SSE-KMS**
  - Analizzato da **Amazon Macie** per rilevare dati sensibili (PII)

---

## 🔐 Livello Security & Governance

### 🕵️‍♂️ Visibilità & Threat Detection

- **Amazon Inspector**
  - Scansione automatica delle immagini container in ECR
  - Rileva vulnerabilità (CVE) e problemi di configurazione
- **Amazon GuardDuty**
  - Threat detection gestito (CloudTrail, VPC Flow Logs, DNS logs)
  - Segnala comportamenti sospetti, chiavi compromesse, esfiltrazione dati, ecc.
- **Amazon Macie**
  - Analisi del bucket S3 log/backup
  - Identificazione di dati sensibili / personali

### 🧭 Postura di Sicurezza & Compliance

- **AWS Security Hub**
  - Consolle centralizzata che aggrega findings da:
    - Inspector
    - GuardDuty
    - Macie
    - Altri servizi di sicurezza
  - Applica standard come **AWS Foundational Security Best Practices** e **CIS**
- **AWS KMS**
  - Gestione chiavi di cifratura per S3, RDS e altri servizi
  - Tracciabilità degli accessi alle chiavi
- **AWS IAM**
  - Ruoli dedicati per:
    - ECS Task Execution
    - Bastion/SSM
    - Servizi di sicurezza
  - Policy secondo principio di **least privilege**

*(Opzionale / futuro)*  
- **AWS Firewall Manager**
  - Gestione centralizzata di policy WAF/Shield/Network Firewall in scenari multi-account
- **AWS Network Firewall**
  - Potenziale layer di firewalling stateful tra subnet e traffico in uscita

---

## 👤 Client & User

- **Client (amministratore)**  
  - Accede alla console AWS  
  - Usa SSM Session Manager / Bastion per troubleshooting e gestione

- **User (utente finale)**  
  Percorso della richiesta:
  1. User → `https://<cloudfront-domain>`  
  2. CloudFront → WAF → ALB pubblico  
  3. ALB → ECS Fargate (subnet private)  
  4. ECS ↔ RDS (dati) / S3 (log/backup)

---

## 🛠️ Variante Terraform (Infrastructure as Code)

Il progetto può essere replicato (o ricreato) interamente tramite **Terraform**, separando i vari layer in file:

- `vpc.tf` → VPC, subnet, IGW, NAT, route table  
- `security-groups.tf` → SG per ALB, ECS, RDS, bastion  
- `ecr.tf` → Repository ECR con image scanning  
- `ecs-fargate.tf` → Cluster, task definition, service Fargate  
- `alb.tf` → ALB pubblico, Target Group, Listener  
- `rds.tf` → RDS + subnet group + KMS encryption  
- `s3-kms-macie.tf` → S3 log/backup + SSE-KMS + Macie job  
- `security-services.tf` → Security Hub, GuardDuty, Inspector  
- `waf-cloudfront.tf` → CloudFront distribution + WAF Web ACL

Esempio di flusso:

```bash
terraform init
terraform plan
terraform apply
# push dell'immagine su ECR
# test dell'app tramite DNS di ALB e poi CloudFront
