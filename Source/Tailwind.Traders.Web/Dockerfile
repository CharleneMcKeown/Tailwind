FROM node:10-alpine as build-node
WORKDIR /ClientApp
COPY ClientApp/package.json .
COPY ClientApp/package-lock.json .
RUN npm install
COPY ClientApp/ . 
RUN npm run build

FROM mcr.microsoft.com/dotnet/core/sdk:2.2.402-stretch as build-net
ENV BuildingDocker true
ENV ASPNETCORE_ENVIRONMENT=Development
WORKDIR /app
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet build
RUN dotnet publish -o /ttweb

FROM mcr.microsoft.com/dotnet/aspnet:2.1 as runtime
WORKDIR /web
COPY --from=build-net /ttweb .
COPY --from=build-node /ClientApp/build ./ClientApp/build
ENTRYPOINT [ "dotnet","Tailwind.Traders.Web.dll" ]
