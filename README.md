# Trabalho 2 – WordPress com Múltiplas Instâncias usando Nginx e Docker Compose

## Arquitetura

```
                    ┌──────────────┐
               ┌───►│  wordpress1  │───┐
┌───────┐   ┌──┴──┐ ├──────────────┤   │   ┌───────┐
│ Ator  │──►│nginx│►│  wordpress2  │───┼──►│ mysql │
└───────┘   └──┬──┘ ├──────────────┤   │   └───────┘
   :80         └───►│  wordpress3  │───┘
                    └──────────────┘
      rede "frontend"          rede "backend"
```

| Contêiner  | Imagem                          | Porta no host | Redes              |
|------------|---------------------------------|---------------|--------------------|
| nginx      | `nginx:1.19.0`                  | **80**        | frontend           |
| wordpress1 | `wordpress:5.4.2-php7.2-apache` | nenhuma       | frontend, backend  |
| wordpress2 | `wordpress:5.4.2-php7.2-apache` | nenhuma       | frontend, backend  |
| wordpress3 | `wordpress:5.4.2-php7.2-apache` | nenhuma       | frontend, backend  |
| mysql      | `mysql:5.7`                     | nenhuma       | backend            |

Como os requisitos do enunciado são atendidos:

- **WordPress acessível só pelo Nginx**: os contêineres do WordPress não têm `ports`. Só o Nginx publica a porta 80.
- **MySQL acessível só pelo WordPress**: o MySQL não tem `ports` e fica apenas na rede `backend`, onde o Nginx não está.
- **Pasta `/var/www/html` compartilhada com o host**: os três WordPress montam `./wordpress:/var/www/html`.
- **Nginx mapeia a mesma pasta em `/usr/share/nginx/html`**: `./wordpress:/usr/share/nginx/html`.
- **Mesmo conteúdo nas três instâncias**: todas usam o mesmo banco (`mysql`) e a mesma pasta.

## Arquivos

- [docker-compose.yml](docker-compose.yml): define os 5 contêineres, as redes e os volumes.
- [nginx/nginx.conf](nginx/nginx.conf): configuração do balanceador (upstream com os 3 WordPress).
- [evidencias/](evidencias/): prints e resultado dos testes.

## Pré-requisitos (Windows)

1. Instale o WSL: abra o PowerShell **como administrador**, rode `wsl --install` e reinicie o computador.
2. Instale o [Docker Desktop](https://www.docker.com/products/docker-desktop/) e abra-o. Espere aparecer "Engine running".
3. Confira no terminal:
   ```powershell
   docker --version
   docker compose version
   ```

## Como executar

Na pasta do projeto:

```powershell
docker compose up -d
```

Na primeira vez as imagens são baixadas e o `wordpress1` copia os arquivos do WordPress para `./wordpress`. O `wordpress2` e o `wordpress3` só sobem depois disso. Acompanhe com:

```powershell
docker compose ps
```

Quando os 5 contêineres estiverem `Up`/`running`, acesse http://localhost no navegador e conclua a instalação do WordPress (idioma, título, usuário e senha).

## Como testar

### Teste principal (pedido no enunciado)

Rode várias vezes:

```powershell
curl.exe -I http://localhost/
```

> No PowerShell use `curl.exe`, porque `curl` sozinho é um apelido do `Invoke-WebRequest`.

O cabeçalho `X-Upstream` vai alternar entre três IPs diferentes (ex.: `172.18.0.3:80`, `172.18.0.4:80`, `172.18.0.5:80`), que são os IPs de wordpress1, wordpress2 e wordpress3. Para ver o IP de cada um:

```powershell
docker inspect -f "{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}" wordpress1 wordpress2 wordpress3
```

Para ver só o `X-Upstream` de 9 requisições seguidas:

```powershell
1..9 | ForEach-Object { curl.exe -sI http://localhost/ | Select-String "X-Upstream" }
```

### Outros testes

```powershell
# Só o nginx publica porta no host
docker ps --format "table {{.Names}}\t{{.Ports}}"

# Os WordPress enxergam o MySQL...
docker exec wordpress1 getent hosts mysql
# ...e o nginx não (não retorna nada, código de saída 2)
docker exec nginx getent hosts mysql

# A pasta compartilhada é a mesma no host e no nginx
docker exec nginx ls /usr/share/nginx/html
```

O resultado completo dos testes está em [evidencias/resultado-testes.txt](evidencias/resultado-testes.txt).

### Teste pelo navegador

1. Abra http://localhost.
2. Aperte **F12**, vá na aba **Rede** (Network) e recarregue a página (F5).
3. Clique na primeira requisição (`localhost`) e procure `X-Upstream` em **Cabeçalhos de resposta**.
4. Recarregue algumas vezes e veja o IP mudar.

## Evidências (prints)

Os prints ficam na pasta [evidencias/](evidencias/):

| Print | O que mostra |
|-------|--------------|
| `01-docker-compose-ps.png` | os 5 contêineres rodando |
| `02-portas-publicadas.png` | só o nginx publica porta no host (`0.0.0.0:80->80/tcp`) |
| `03-ips-wordpress.png` | os IPs dos 3 contêineres WordPress |
| `04-curl-balanceamento-completo.png` | 3 respostas do `curl.exe -I`, cada uma atendida por um WordPress diferente |
| `05-curl-balanceamento-resumo.png` | 9 requisições alternando o `X-Upstream` entre os 3 IPs |
| `06-mysql-acesso.png` | os WordPress enxergam o MySQL; o nginx não |
| `07-pasta-compartilhada.png` | a mesma pasta no host e no nginx |
| `08-script-testes-completo.png` | a bateria completa de testes (contêineres, portas, IPs, balanceamento, MySQL e pasta compartilhada) |
| `09-site-wordpress-navegador.png` | o site WordPress aberto em http://localhost |
| `10-navegador-network-x-upstream-1.png` | F12 > Network mostrando o cabeçalho `X-Upstream` |
| `11-docker-desktop-logs-balanceamento.png` | Docker Desktop: os 5 contêineres e os logs com o nginx (`172.19.0.5`) repassando as requisições para wordpress1, 2 e 3 |

Mapeamento dos IPs (rede `frontend`, usada pelo nginx): wordpress1 = `172.19.0.2`, wordpress2 = `172.19.0.4`, wordpress3 = `172.19.0.3`. Os IPs podem mudar se os contêineres forem recriados.

## Encerrar

```powershell
docker compose down        # para e remove os contêineres
docker compose down -v     # também apaga o banco de dados
```
