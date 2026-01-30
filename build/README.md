# Dockerfiles padrão (repo de templates)

Esta pasta contém Dockerfiles genéricos usados pelo pipeline quando `use_default_dockerfile: true`.

- **Dockerfile.api**: APIs .NET (ASP.NET Core). Exige `ARG PROJECT_NAME` (nome do .csproj).
- **Dockerfile.worker**: Workers .NET (runtime). Exige `ARG PROJECT_NAME` (nome do .csproj).

O build context é a **raiz do repositório da aplicação**. Espera-se estrutura com `src/*.sln` e `src/<ProjectName>/<ProjectName>.csproj`. O pipeline passa `--build-arg PROJECT_NAME=<ProjectName>` quando usa o Dockerfile padrão.
