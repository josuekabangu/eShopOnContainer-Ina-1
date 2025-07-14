# Étape 1 : build
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build

# Évite les problèmes de fallback de chemin NuGet sur Windows
ENV NUGET_PACKAGES=/root/.nuget/packages

WORKDIR /src

# Copier les projets nécessaires au restore
COPY Directory.Packages.props ./
COPY src/eShop.ServiceDefaults/eShop.ServiceDefaults.csproj ./src/eShop.ServiceDefaults/
COPY src/Basket.API/Basket.API.csproj ./src/Basket.API/
COPY src/EventBusRabbitMQ/EventBusRabbitMQ.csproj ./src/EventBusRabbitMQ/

# Restore uniquement Basket.API (ce qui restaure aussi ses dépendances)
RUN dotnet restore src/Basket.API/Basket.API.csproj \
    --use-lock-file --no-cache --force --disable-parallel --ignore-failed-sources

# Copier tout le code
COPY . .

# Build
RUN dotnet build src/Basket.API/Basket.API.csproj -c Release -o /app/build

# Publish (plus léger)
RUN dotnet publish src/Basket.API/Basket.API.csproj -c Release -o /app/publish

# Étape 2 : runtime allégée
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime

WORKDIR /app

# Copier les fichiers de l'app publiés
COPY --from=build /app/publish ./

# Copier le script d'attente
COPY scripts/wait-for-it.sh /app/wait-for-it.sh

# Donner les droits d’exécution + installer curl (nécessaire pour la healthcheck)
RUN chmod +x /app/wait-for-it.sh && \
    apt-get update && apt-get install -y curl && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Entrypoint pour attendre Redis et RabbitMQ avant de lancer l'app
ENTRYPOINT ["/bin/sh", "-c", "/app/wait-for-it.sh redis:6379 -- /app/wait-for-it.sh rabbitmq:5672 -- dotnet Basket.API.dll"]
