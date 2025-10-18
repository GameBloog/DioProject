# Brute Force Lab — Kali Linux + Medusa + Metasploitable2 + DVWA

> **Aviso importante:** Este repositório documenta testes de segurança **somente** realizados em **ambiente controlado e isolado** (VMs host-only / internal network). **Não** execute ataques contra sistemas alheios. Tudo aqui é para fins educativos.

---

## Índice

1. [Objetivo](#objetivo)
2. [Ambiente de Teste](#ambiente-de-teste)
3. [Topologia e Preparação](#topologia-e-prepara%C3%A7%C3%A3o)
4. [Enumeração (Nmap)](#enumera%C3%A7%C3%A3o-nmap)
5. [Testes Executados (Medusa e outras ferramentas)](#testes-executados)

   * FTP (brute force)
   * Web (form-based)
   * SMB (password spraying)
6. [Wordlists e Scripts](#wordlists-e-scripts)
7. [Evidências e Logs](#evid%C3%AAncias-e-logs)
8. [Análise e Resultados](#an%C3%A1lise-e-resultados)
9. [Mitigações e Recomendações](#mitiga%C3%A7%C3%B5es-e-recomenda%C3%A7%C3%B5es)
10. [Boas práticas de documentação](#boas-pr%C3%A1ticas)
11. [Referências](#refer%C3%AAncias)

---

## Objetivo

Demonstrar, em ambiente controlado, técnicas de força bruta e password spraying contra serviços comumente configurados em VMs vulneráveis (FTP, Web form e SMB), usando *Kali Linux* como atacante e *Medusa* como ferramenta principal, bem como documentar a metodologia, evidências e recomendações de mitigação.

---

## Ambiente de Teste

* **Kali Linux** (máquina atacante) — ISO/VM oficial.
* **Metasploitable2** (máquina alvo vulnerável) — VM intencionalmente vulnerável.
* **DVWA** (Damn Vulnerable Web App) — aplicação web vulnerável (usada para testes de formulário).
* **VirtualBox** — rede `Host-Only` ou `Internal Network` para isolar o tráfego entre as VMs.
* *Snapshots:* recomenda-se criar snapshots antes de começar os testes para fácil rollback.

> Observação: mantenha todas as VMs desconectadas de redes produtivas/internet durante os testes.

---

## Topologia e Preparação

1. Crie uma rede host-only no VirtualBox e conecte as VMs (Kali, Metasploitable2 e, se for o caso, uma VM com DVWA).
2. Configure endereços IP estáticos ou verifique via `ifconfig` / `ip a` os IPs atribuídos.
3. Atualize Kali e instale/atualize ferramentas utilizadas (Medusa, Nmap, Hydra, Burp Suite se for usar).
4. Tire snapshots antes de grandes mudanças.

Exemplo rápido (Kali):

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install medusa nmap hydra -y
```

---

## Enumeração (Nmap)

**Objetivo:** descobrir hosts ativos, portas abertas e serviços/banners.

Comandos úteis:

```bash
# Scan completo de portas + detecção de versão
nmap -sV -p- <IP_ALVO>

# Scan mais focado (serviços comuns)
nmap -sV -p 21,22,80,443,139,445 <IP_ALVO>

# Scan com scripts leves para enumeração
nmap -sV -sC <IP_ALVO>
```

Guarde a saída em arquivos para anexar às evidências:

```bash
nmap -sV -p- <IP_ALVO> -oN outputs/nmap_full_<IP_ALVO>.txt
```

---

## Testes Executados

> **Lembrete de segurança:** comandos exemplificados abaixo são para **laboratório isolado**. Documente data/hora, IP alvo, parâmetros e wordlists usadas para cada execução.

### 1) FTP — Força Bruta (Medusa)

**Objetivo:** testar força bruta de senha para um usuário FTP conhecido.

**Exemplo de comando:**

```bash
# Exemplo básico — substitua <IP>, <user> e caminhos
medusa -h <IP_ALVO> -u <USUARIO> -P wordlists/ftp-passwords.txt -M ftp -t 8 -f
```

Explicação rápida dos parâmetros:

* `-h` — host alvo
* `-u` — usuário alvo (usar -U para arquivo de usuários)
* `-P` — palavra(s) ou arquivo de senhas
* `-M` — módulo (ftp)
* `-t` — threads paralelas
* `-f` — parar após sucesso (opcional)

**Logs:** salve a saída em `outputs/medusa_ftp_<IP>.log`.

---

### 2) Web (DVWA) — Form-based

**Objetivo:** automatizar tentativas de login em formulário web.

**Observação:** Medusa tem suporte limitado a alguns formulários; ferramentas como `hydra` (módulo `http-form-post`) ou `Burp Suite` (Intruder) normalmente oferecem mais flexibilidade. Abaixo um exemplo com **Hydra**:

```bash
# Exemplo com hydra (HTTP POST form)
hydra -l admin -P wordlists/web-passwords.txt <IP_ALVO> http-post-form \
"/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:F=incorrect"
```

No exemplo acima você deve adaptar:

* caminho do formulário `/dvwa/login.php`
* campos `username` e `password`
* string que indica falha (`F=incorrect`) — use o texto/response que aparece quando o login falha.

**Alternativa:** usar Burp Proxy + Intruder para maior controle (recomendado para aprendizado sobre CSRF, tokens dinâmicos, etc.).

---

### 3) SMB — Password Spraying

**Objetivo:** testar poucas senhas comuns contra muitos usuários (técnica de password spraying) para evitar bloqueios por muitas tentativas em único usuário.

**Fluxo sugerido:**

1. Enumere usuários (se possível) — via `enum4linux`, `smbclient`, ou outras técnicas de enumeração no ambiente controlado.
2. Aplique um conjunto restrito de senhas (ex.: 5–10 senhas comuns) contra a lista de usuários, com delays entre tentativas.

**Exemplo de comando (modelo Medusa):**

```bash
# Supondo que seu Medusa tenha suporte a SMB (ajuste o módulo conforme disponibilidade)
medusa -h <IP_ALVO> -U wordlists/users.txt -P wordlists/spray-passwords.txt -M smb -t 4
```

**Observação:** ajuste `-t` (threads) e inclua delays entre execuções para simular um spraying mais realista e para fins de estudo sobre detecção/mitigação.

---

## Wordlists e Scripts

* **Recomendação:** crie wordlists compactas e específicas para o lab (100–500 entradas) — isso facilita reproduzir testes e entender resultados.
* Local sugerido: `/wordlists`

Exemplo de arquivos:

* `wordlists/ftp-passwords.txt`
* `wordlists/web-passwords.txt`
* `wordlists/users.txt`
* `wordlists/spray-passwords.txt`

Se criar scripts para automatizar execuções, inclua comentários explicando os parâmetros e coloque-os em `/scripts`.

---

## Evidências e Logs

Guarde capturas e saídas organizadas:

* `/images/` — screenshots (topologia VirtualBox, outputs do Nmap, telas do DVWA, confirmações de acesso).
* `/outputs/` — logs de comandos (nmap, medusa, hydra etc.).

**Mascaramento:** se por algum motivo houver credenciais sensíveis, **mascare-as** antes de subir ao repositório público.

---

## Análise e Resultados

Para cada teste, documente:

* objetivo do teste;
* parâmetros usados (comandos);
* wordlist utilizada (tamanho, origem);
* resultado (ex.: credenciais válidas encontradas? quais? — lembrar de mascarar antes de publicar);
* como você validou o acesso (ex.: listar um arquivo que existe no target, verificar banner, etc.);
* riscos associados ao serviço na configuração testada.

Exemplo de resumo que você pode colocar aqui após executar o lab:

> **FTP** — força bruta com wordlist de 200 senhas; credencial `ftpuser:weakpass` obtida; acesso validado listando `/home/ftpuser`.

> **Web (DVWA)** — formulário vulnerável sem proteção de lockout; com hydra, `admin:1234` foi testado com sucesso.

> **SMB** — password spraying com 10 senhas comuns em 50 usuários; 2 contas comprometeram; sistema não tem lockout.

---

## Mitigações e Recomendações

Recomendações práticas para reduzir risco de brute force / password spraying:

* **Bloqueio/Rate-limiting:** aplicar bloqueio temporário após X tentativas falhas.
* **MFA (Autenticação Multifator):** sempre que possível.
* **Política de senhas fortes:** comprimento mínimo, proibição de senhas comuns, verificação por listas (banned passwords).
* **Monitoramento & Alertas:** coletar logs de autenticação e criar alertas para picos de tentativas.
* **Proteções em aplicações web:** tokens anti-CSRF, validação server-side, captcha para formularios críticos.
* **Segmentação e princípio do menor privilégio:** reduzir exposição de serviços e aplicar patches.

Inclua recomendações específicas por serviço no seu relatório (ex.: desativar FTP anon, configurar SMB com políticas de lockout, usar HTTPS em aplicações web).

---

## Boas práticas de documentação

* Use timestamps em cada execução (`date`) e explique o propósito de cada comando.
* Inclua diagramas simples (p.ex. figura da topologia) e legendas.
* Garanta que todos os comandos estejam claramente marcados como "apenas para laboratório".
* Evite postar credenciais reais sem mascaramento.

---

## Referências

* Kali Linux — site oficial
* DVWA — Damn Vulnerable Web Application (repositório oficial)
* Medusa — documentação e man pages
* Nmap — documentação
* Hydra, Burp Suite — referências para testes em formulários

---

## Conclusão e próximos passos

* Resuma aprendizados e descreva próximos passos (ex.: testar outras ferramentas como `Crowbar`, `Patator`, aprofundar em mitigação, integrar IDS para detecção de brute force).

---

> **Licença & Uso:** Este material é educacional. Não me responsabilizo por uso externo a ambientes de laboratório isolados.
