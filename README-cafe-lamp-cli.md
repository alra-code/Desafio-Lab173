<div align="center">

![header](https://capsule-render.vercel.app/api?type=waving&color=gradient&customColorList=6,11,18&height=200&section=header&text=AWS%20CLI%20%2B%20LAMP%20Challenge&fontSize=44&fontColor=ffffff&animation=fadeIn&desc=Troubleshooting%20de%20script%20e%20deploy%20do%20site%20Caf%C3%A9%20no%20EC2&descSize=18&descAlignY=70)

![AWS CLI](https://img.shields.io/badge/AWS_CLI-232F3E?style=for-the-badge&logo=amazonaws&logoColor=white)
![EC2](https://img.shields.io/badge/Amazon-EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Linux](https://img.shields.io/badge/Amazon_Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Bash](https://img.shields.io/badge/Bash-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white)
![Apache](https://img.shields.io/badge/Apache-D22128?style=for-the-badge&logo=apache&logoColor=white)
![MariaDB](https://img.shields.io/badge/MariaDB-003545?style=for-the-badge&logo=mariadb&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-777BB4?style=for-the-badge&logo=php&logoColor=white)
![Status](https://img.shields.io/badge/Status-Conclu%C3%ADdo-2ea44f?style=for-the-badge)

**Provisionamento de uma instância LAMP pela linha de comando, encontrando e corrigindo erros de um script** 🛠️☁️

</div>

---

## 📌 Sobre o projeto

Neste laboratório, usei a **AWS CLI** para criar uma instância **Amazon EC2 LAMP** (Linux, Apache, MariaDB e PHP) que hospeda o aplicativo web **Café**.

O desafio: o script shell fornecido continha **problemas intencionais**. Meu trabalho foi **ler o script, diagnosticar as falhas e corrigi-las**, validando cada correção até o site ficar no ar e registrar pedidos no banco de dados.

---

## 🎯 Objetivos

- ✅ Conectar a uma instância "host da CLI" com **EC2 Instance Connect**
- ✅ Configurar a **AWS CLI** com `aws configure`
- ✅ Ler e entender um **script Bash** que provisiona infraestrutura
- ✅ Diagnosticar e corrigir **2 problemas** do script
- ✅ Usar o **nmap** para identificar portas bloqueadas
- ✅ Validar o **User Data** pelo log do `cloud-init`
- ✅ Testar o site Café e o registro de pedidos no banco de dados

---

## 🗺️ Arquitetura e fluxo

```mermaid
flowchart LR
    DEV["👩‍💻 Eu<br/>EC2 Instance Connect"] --> CLI

    subgraph AWS["☁️ AWS Cloud · us-west-2"]
        CLI["🖥️ Host da CLI<br/>aws configure<br/>script .sh"] -->|"run-instances"| LAMP

        subgraph VPC["🔷 Cafe VPC"]
            SG["🛡️ Security Group cafeSG<br/>SSH 22 · HTTP 80"] -.protege.-> LAMP
            LAMP["☕ EC2 LAMP cafeserver<br/>Apache + PHP + MariaDB"]
        end
    end

    USER["🌐 Navegador<br/>/cafe"] -->|"HTTP 80"| LAMP

    style DEV fill:#0d6efd,color:#fff,stroke:#0a58ca
    style CLI fill:#232F3E,color:#fff,stroke:#FF9900
    style LAMP fill:#2ea44f,color:#fff,stroke:#22863a
    style SG fill:#D22128,color:#fff,stroke:#a51a20
    style USER fill:#8C4FFF,color:#fff,stroke:#6f3fd1
```

---

## 🧰 Tecnologias e ferramentas

| 🎨 Categoria | 🔧 Ferramenta | 📝 Uso no projeto |
|:---:|:---:|---|
| ⌨️ Automação | **AWS CLI** | Cria a instância e o Security Group por comandos |
| 📜 Script | **Bash** | Orquestra toda a criação dos recursos |
| ☁️ Compute | **Amazon EC2** | Servidor da aplicação |
| 🛡️ Segurança | **Security Group** | Libera as portas 22 e 80 |
| 🕸️ Web | **Apache httpd + PHP** | Servidor web e aplicação |
| 🗄️ Banco | **MariaDB** | Armazena pedidos do Café |
| 🔍 Diagnóstico | **nmap** | Varredura de portas da instância |
| 📋 Logs | **cloud-init** | Confere a execução do User Data |
| ✏️ Editor | **vi** | Edição do script pelo terminal |

---

## 🔄 O que o script faz

| 🔢 Etapa | 📝 Descrição |
|:---:|---|
| 1️⃣ | Define o tamanho da instância (`t3.small`) |
| 2️⃣ | Percorre as regiões até encontrar a VPC chamada **Cafe VPC** |
| 3️⃣ | Busca ID da sub-rede, par de chaves e AMI |
| 4️⃣ | Limpa recursos de execuções anteriores (instância `cafeserver` e SG `cafeSG`) |
| 5️⃣ | Cria o Security Group com as portas 22 e 80 |
| 6️⃣ | Cria a instância com `run-instances` e o arquivo de **User Data** |
| 7️⃣ | Aguarda e exibe o **IP público** da instância |

---

## 🐞 Troubleshooting

### 🔴 Problema 1: AMI não encontrada

```text
An error occurred (InvalidAMIID.NotFound) when calling the RunInstances operation:
The image id '[ami-01477f93b365aa11a]' does not exist
```

![Erro da AMI](./imagem/01-erro-ami.png)

| 🔍 Item | 📝 Detalhe |
|---|---|
| **Sintoma** | O `run-instances` falha dizendo que a AMI não existe |
| **Investigação** | `grep -n "region" create-lamp-instance-v2.sh` listou todas as linhas que usam a região |
| **Causa** | O script encontrou a VPC e a AMI em `us-west-2`, mas a **linha 160** (dentro do `run-instances`) tinha a região **fixa em `us-east-2`**. AMIs são regionais |
| **Correção** | Usar a variável `$region`, como o restante do script |

![grep da região](./imagem/02-grep-region.png)

```diff
- --region us-east-2 \
+ --region $region \
```

---

### 🔴 Problema 2: porta 80 bloqueada no Security Group

| 🔍 Item | 📝 Detalhe |
|---|---|
| **Sintoma** | Instância criada com IP público, mas `http://<ip>` não abria |
| **Diagnóstico** | `nmap -Pn <ip>` mostrou **22/tcp open** e **8080/tcp closed**, sem a porta 80 |
| **Investigação** | `grep -n "port" create-lamp-instance-v2.sh` revelou a regra da porta 80 usando `--port 8080` |
| **Causa** | A **linha 149** liberava a porta **8080** no Security Group, embora a mensagem do script dissesse "Opening port 80". O Apache escuta na 80 |
| **Correção** | Ajustar a regra para a porta 80 |

![nmap com 8080 e grep da porta](./imagem/03-nmap-8080-grep-port.png)

```diff
- --port 8080 \
+ --port 80 \
```

---

### 🟡 Observação: porta 80 `closed` logo após criar a instância

Depois de corrigir o Security Group, o nmap passou a mostrar a **porta 80 como `closed`**. Isso indica que o tráfego já chegava à instância, mas o **Apache ainda não estava escutando**: o **User Data** continuava instalando os pacotes.

![nmap porta 80 closed](./imagem/04-nmap-porta80-closed.png)

Poucos minutos depois, uma nova varredura mostrou a **porta 80 `open`**.

> 💡 **Lição:** o User Data é executado em segundo plano após o boot. Uma porta `closed` logo após criar a instância nem sempre é erro de configuração: pode ser só a inicialização ainda em andamento.

---

## ✅ Resultados

### 🟢 Porta 80 aberta (nmap)

![nmap porta 80 open](./imagem/05-nmap-porta80-open.png)

### 🌐 Servidor web respondendo no navegador

![Servidor web](./imagem/06-servidor-web.png)

### ☕ Site Café registrando pedidos no MariaDB

Dois pedidos feitos pelo site e gravados no banco de dados da instância:

![Histórico de pedidos](./imagem/07-historico-pedidos.png)

---

## 🪜 Passo a passo resumido

1. 🔌 Conectar ao **host da CLI** pelo EC2 Instance Connect
2. 🔑 Rodar `aws configure` (chaves, região e formato `json`)
3. 💾 Criar backup do script: `cp create-lamp-instance-v2.sh create-lamp-instance.backup`
4. 👀 Ler o script e o arquivo de User Data
5. ▶️ Executar o script e observar a falha da AMI
6. 🛠️ Corrigir a região na **linha 160** e executar de novo
7. 🔍 Rodar `nmap -Pn <ip>` e investigar a porta 80
8. 🛠️ Corrigir a porta na **linha 149** e executar de novo
9. ⏳ Aguardar o User Data terminar e confirmar a porta 80 `open`
10. ☕ Acessar `http://<ip>` e `http://<ip>/cafe` e testar pedidos

---

## 🧠 Aprendizados

- ⌨️ Como **provisionar recursos pela AWS CLI**, sem usar o console
- 🌎 Recursos como **AMIs são regionais**: região errada gera erro de "não encontrado"
- 🔍 Usar **nmap** para separar problema de rede (`filtered`) de problema de aplicação (`closed`)
- 🛡️ Revisar as regras do **Security Group** antes de investigar o servidor
- 🔎 Usar **`grep -n`** para localizar rapidamente a linha com problema em um script
- ⏳ O **User Data** roda de forma assíncrona: é preciso esperar a conclusão antes de testar
- 📜 Habilidade de **ler e depurar scripts Bash** de terceiros

---

## 🔐 Boas práticas de segurança

> ⚠️ **Nunca publique no GitHub** chaves de acesso (`AccessKey`/`SecretKey`) ou prints que mostrem credenciais. Os IPs deste laboratório são temporários, mas o hábito vale para projetos reais.
>
> 💰 Ao terminar, **encerre a instância** para evitar custos.

---

<div align="center">

### 👩‍💻 Autora

**LUCIANA RB ASSIS**

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/lucianarbassis)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/ALRA-CODE)

⭐ Se este projeto foi útil, deixe uma estrela no repositório!

![footer](https://capsule-render.vercel.app/api?type=waving&color=gradient&customColorList=6,11,18&height=100&section=footer)

</div>
