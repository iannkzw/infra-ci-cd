# Dockerfiles padrão (repo de templates)

Esta pasta contém Dockerfiles genéricos usados pelo pipeline quando `use_default_dockerfile: true`.

- **Dockerfile.api**: APIs .NET (ASP.NET Core). Exige `ARG PROJECT_NAME` (nome do .csproj).
- **Dockerfile.worker**: Workers .NET (runtime). Exige `ARG PROJECT_NAME` (nome do .csproj).

O build context é a **raiz do repositório da aplicação**. Espera-se estrutura com `src/*.sln` e `src/<ProjectName>/<ProjectName>.csproj`. O pipeline passa `--build-arg PROJECT_NAME=<ProjectName>` quando usa o Dockerfile padrão.

**Importante:** O repositório da aplicação deve ter um `.dockerignore` na raiz excluindo `**/obj/` e `**/bin/`. Caso contrário, o `COPY src/. .` sobrescreve o `obj/` gerado pelo `dotnet restore` dentro do container com o `obj/` do host (runner), e o `dotnet publish --no-restore` falha com NETSDK1064 (pacote não encontrado).
