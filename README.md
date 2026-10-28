# Laboratorio Docker - Integracao DevOps - Unidade 3

Aplicacao ASP.NET Core / .NET 10. O laboratorio cria tres imagens imutaveis e usa um endpoint de saude para decidir promocao e rollback.

## Arquivos

- `Program.cs`: endpoints `/`, `/health`, `/version` na release saudavel 1.1.0; o programa tambem aceita `--health-check` para o Compose.
- `DevOpsWebApi.csproj`: projeto .NET 10 sem dependencias NuGet externas.
- `Dockerfile`: build multi-stage (SDK para compilar, ASP.NET Runtime para executar).
- `.dockerignore`: nao envia obj/bin/git/evidencias para o build.
- `compose.yaml`: ambientes staging e production, ambos na imagem 1.1.0, com healthcheck dentro do container.
- `evidencias/`: log completo, snapshots dos tres Program.cs, testes HTTP e relatorio HTML/PDF (se Edge/Chrome disponivel).

## Pre-requisitos

Docker Desktop/Engine em funcionamento, Docker Compose V2 e SDK .NET 10. O script pode instalar SDK 10 sem administrador no diretorio do usuario.

## Criacao e teste local

```powershell
dotnet new webapi -n DevOpsWebApi
cd DevOpsWebApi
dotnet run
# Acesse http://localhost:<porta-exibida>/health
```

O script substitui o `Program.cs` e o projeto gerado pelo template por versoes minimas, sem pacotes NuGet externos, para reduzir dependencias no computador do laboratorio.

## Build e containers

```powershell
docker run --rm hello-world
docker build -t devops-webapi:1.0.0 .
docker images devops-webapi
docker run -d --name devops-api -p 127.0.0.1:8080:8080 devops-webapi:1.0.0
curl.exe -f http://127.0.0.1:8080/health
docker logs devops-api
docker inspect devops-api
docker stop devops-api
docker rm devops-api
```

## Staging e production

```powershell
docker run -d --name devops-api-staging -p 127.0.0.1:8081:8080 -e APP_ENVIRONMENT=staging -e APP_VERSION=1.0.0 devops-webapi:1.0.0
docker run -d --name devops-api-prod -p 127.0.0.1:8082:8080 -e APP_ENVIRONMENT=production -e APP_VERSION=1.0.0 devops-webapi:1.0.0
```

## Nova versao 1.1.0

Adiciona `/version`, constroi `devops-webapi:1.1.0`, atualiza primeiro o staging, valida `/health` e `/version`, depois promove production para a mesma imagem.

```powershell
docker build -t devops-webapi:1.1.0 .
curl.exe -f http://127.0.0.1:8081/health
curl.exe -f http://127.0.0.1:8081/version
```

## Release 1.2.0 falha e rollback

A versao 1.2.0 retorna HTTP 500 intencionalmente em `/health`. Isso bloqueia a promocao normal. O exercicio simula brevemente a implantacao dela na production LOCAL de laboratorio para demonstrar incidente e recuperacao. Rollback: recriar production com `devops-webapi:1.1.0` e comprovar HTTP 200.

```powershell
docker build -t devops-webapi:1.2.0 .
# Falha esperada apenas na demonstracao:
curl.exe -f http://127.0.0.1:8081/health
# Apos rollback:
curl.exe -f http://127.0.0.1:8082/health
```

O fonte `Program.cs` final e restaurado para a versao saudavel 1.1.0; o snapshot defeituoso permanece apenas como `.txt` nas evidencias.

## Docker Compose + health check

```powershell
docker compose config
docker compose up -d
docker compose ps
docker compose logs
curl.exe -f http://127.0.0.1:8081/health
curl.exe -f http://127.0.0.1:8082/health
docker compose down
```

O container base `mcr.microsoft.com/dotnet/aspnet:10.0` nao garante `curl`. O healthcheck usa o proprio executavel .NET com `--health-check`, que faz uma requisicao HTTP para `/health` e devolve exit code 0 (sucesso) ou 1 (falha), sem instalar ferramentas extras.

## Registro e entrega

Veja `evidencias/Relatorio-Evidencias-Docker-Unidade3.html`, PDF (quando gerado), e `evidencias/execucao-completa.txt`. O ZIP `Entrega-Docker-Unidade3.zip` fica um diretorio acima.

## Git

URL do repositorio: https://github.com/DanielBDeDeus/devops-unidade3-docker

Para publicar com o script, use `-PublishGitHub -PublicRepository` depois de autenticar o GitHub CLI (`gh auth login`). Sem autenticacao, o projeto e os comprovantes continuam locais.

## Seguranca e limpeza

Somente 127.0.0.1 e exposto. O script nao toca em projetos Docker alheios e nao usa `docker system prune`. Ao final, executa `docker compose down` por padrao, mas mantem as tres imagens e os arquivos de evidencias. Para manter staging e production em funcionamento, use `-KeepRunning`.
