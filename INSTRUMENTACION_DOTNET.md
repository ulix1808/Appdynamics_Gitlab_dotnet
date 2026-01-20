# Guía Detallada: Instrumentación AppDynamics para .NET

Esta guía proporciona instrucciones paso a paso para instrumentar aplicaciones .NET con AppDynamics y automatizar el proceso mediante GitLab CI/CD.

## Índice

1. [Requisitos Previos](#requisitos-previos)
2. [Instrumentación Manual (.NET Framework)](#instrumentación-manual-net-framework)
3. [Instrumentación Manual (.NET Core/.NET 5+)](#instrumentación-manual-net-corenet-5)
4. [Automatización con GitLab CI/CD](#automatización-con-gitlab-cicd)
5. [Configuración del Pipeline](#configuración-del-pipeline)
6. [Verificación](#verificación)
7. [Troubleshooting](#troubleshooting)

---

## Requisitos Previos

### Información Requerida de AppDynamics

- [ ] **Controller Host**: Hostname o IP del Controller
- [ ] **Controller Port**: Puerto (típicamente 8090 o 443)
- [ ] **Protocolo**: HTTP o HTTPS
- [ ] **Account Name**: Nombre de la cuenta de AppDynamics
- [ ] **Access Key**: Clave de acceso (Account Settings > Access Keys)
- [ ] **Application Name**: Nombre de la aplicación en AppDynamics
- [ ] **Tier Name**: Nombre del tier/servidor
- [ ] **Node Name**: Nombre único del nodo

### Información del Entorno

- [ ] **Versión de .NET**: .NET Framework o .NET Core/.NET 5+
- [ ] **Plataforma**: Windows, Linux, o Docker
- [ ] **Servidor Web**: IIS, Kestrel, u otro
- [ ] **GitLab CI/CD**: Habilitado y Runner configurado

---

## Instrumentación Manual (.NET Framework)

### Paso 1: Descargar el Agente .NET

1. Acceder al portal de AppDynamics
2. **Settings > Downloads > .NET Agent**
3. Descargar el instalador o ZIP
4. Guardar en ubicación accesible

### Paso 2: Instalar el Agente

**Opción A: Instalador MSI/EXE**

```powershell
# Ejecutar instalador silenciosamente
.\AppServerAgent-x64-Install.exe /S /D=C:\AppDynamics\AppServerAgent

# O con MSI
msiexec /i AppServerAgent-x64.msi /quiet /norestart TARGETDIR=C:\AppDynamics\AppServerAgent
```

**Opción B: Extracción Manual**

```powershell
# Extraer ZIP manualmente
Expand-Archive -Path AppServerAgent.zip -DestinationPath C:\AppDynamics
```

### Paso 3: Configurar controller-info.xml

**Ubicación:** `C:\AppDynamics\AppServerAgent\conf\controller-info.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<controller-info>
    <!-- Información del Controller -->
    <controller-host>controller.appdynamics.com</controller-host>
    <controller-port>8090</controller-port>
    <controller-ssl-enabled>false</controller-ssl-enabled>
    
    <!-- Credenciales -->
    <account-name>customer1</account-name>
    <account-access-key>xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx</account-access-key>
    
    <!-- Información de la Aplicación -->
    <application-name>DotNet-Application</application-name>
    <tier-name>DotNet-WebAPI-Tier</tier-name>
    <node-name>DotNet-Node-01</node-name>
    
    <!-- Configuración adicional -->
    <sim-enabled>false</sim-enabled>
</controller-info>
```

### Paso 4: Instrumentar la Aplicación

**Para aplicaciones ASP.NET (web.config):**

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <!-- AppDynamics Configuration Section -->
    <section name="appDynamics" type="AppDynamics.Agent.Common.Config.AppDynamicsSection, AppDynamics.Agent.Common" />
  </configSections>
  
  <appDynamics>
    <instrumentation>
      <enabled>true</enabled>
      <applicationName>DotNet-Application</applicationName>
      <tierName>DotNet-WebAPI-Tier</tierName>
      <nodeName>DotNet-Node-01</nodeName>
    </instrumentation>
  </appDynamics>
  
  <system.web>
    <httpModules>
      <add name="AppDynamicsAgent" type="AppDynamics.Agent.AgentHttpModule, AppDynamics.Agent.Web" />
    </httpModules>
    <httpHandlers>
      <add path="AppDynamics*" verb="*" type="AppDynamics.Agent.AgentHttpHandler, AppDynamics.Agent.Web" />
    </httpHandlers>
  </system.web>
  
  <system.webServer>
    <modules>
      <add name="AppDynamicsAgent" type="AppDynamics.Agent.AgentHttpModule, AppDynamics.Agent.Web" />
    </modules>
    <handlers>
      <add name="AppDynamicsAgent" path="AppDynamics*" verb="*" type="AppDynamics.Agent.AgentHttpHandler, AppDynamics.Agent.Web" />
    </handlers>
  </system.webServer>
</configuration>
```

**Para aplicaciones de consola o servicio:**

Agregar referencia al agente y configurar en código:

```csharp
using AppDynamics.Agent;

namespace MyApplication
{
    class Program
    {
        static void Main(string[] args)
        {
            // Inicializar AppDynamics
            AppDynamicsAgent.Initialize();
            
            // Tu código de aplicación
            // ...
        }
    }
}
```

---

## Instrumentación Manual (.NET Core/.NET 5+)

### Paso 1: Instalar NuGet Package

```bash
# Usando dotnet CLI
dotnet add package AppDynamics.Agent

# O usando Package Manager Console
Install-Package AppDynamics.Agent
```

### Paso 2: Configurar en Program.cs

```csharp
using AppDynamics.Agent;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Hosting;

namespace MyApplication
{
    public class Program
    {
        public static void Main(string[] args)
        {
            // Configurar AppDynamics ANTES de crear el host
            ConfigureAppDynamics();
            
            CreateHostBuilder(args).Build().Run();
        }
        
        private static void ConfigureAppDynamics()
        {
            // Configuración mediante variables de entorno (recomendado)
            // O mediante código:
            /*
            AppDynamicsConfig.Configure(new AppDynamicsConfig
            {
                ControllerHost = Environment.GetEnvironmentVariable("APPDYNAMICS_CONTROLLER_HOST_NAME"),
                ControllerPort = int.Parse(Environment.GetEnvironmentVariable("APPDYNAMICS_CONTROLLER_PORT") ?? "8090"),
                ControllerSslEnabled = bool.Parse(Environment.GetEnvironmentVariable("APPDYNAMICS_CONTROLLER_SSL_ENABLED") ?? "false"),
                AccountName = Environment.GetEnvironmentVariable("APPDYNAMICS_AGENT_ACCOUNT_NAME"),
                AccountAccessKey = Environment.GetEnvironmentVariable("APPDYNAMICS_AGENT_ACCOUNT_ACCESS_KEY"),
                ApplicationName = Environment.GetEnvironmentVariable("APPDYNAMICS_AGENT_APPLICATION_NAME"),
                TierName = Environment.GetEnvironmentVariable("APPDYNAMICS_AGENT_TIER_NAME"),
                NodeName = Environment.GetEnvironmentVariable("APPDYNAMICS_AGENT_NODE_NAME")
            });
            */
        }
        
        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                });
    }
}
```

### Paso 3: Configurar Variables de Entorno

**En appsettings.json (solo para referencia, usar variables de entorno en producción):**

```json
{
  "AppDynamics": {
    "ControllerHost": "controller.appdynamics.com",
    "ControllerPort": "8090",
    "ControllerSslEnabled": "false",
    "AccountName": "customer1",
    "AccountAccessKey": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "ApplicationName": "DotNet-Application",
    "TierName": "DotNet-WebAPI-Tier",
    "NodeName": "DotNet-Node-01"
  }
}
```

**Variables de entorno (recomendado):**

```bash
export APPDYNAMICS_CONTROLLER_HOST_NAME="controller.appdynamics.com"
export APPDYNAMICS_CONTROLLER_PORT="8090"
export APPDYNAMICS_CONTROLLER_SSL_ENABLED="false"
export APPDYNAMICS_AGENT_ACCOUNT_NAME="customer1"
export APPDYNAMICS_AGENT_ACCOUNT_ACCESS_KEY="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export APPDYNAMICS_AGENT_APPLICATION_NAME="DotNet-Application"
export APPDYNAMICS_AGENT_TIER_NAME="DotNet-WebAPI-Tier"
export APPDYNAMICS_AGENT_NODE_NAME="DotNet-Node-01"
```

---

## Automatización con GitLab CI/CD

### Concepto General

Los pipelines de GitLab pueden automatizar:
1. **Build** de la aplicación .NET
2. **Descarga** del agente AppDynamics (si es necesario)
3. **Configuración** automática del agente
4. **Instrumentación** durante el build o despliegue
5. **Despliegue** con el agente configurado

### Variables de CI/CD en GitLab

Configurar en: **Settings > CI/CD > Variables**

**Variables obligatorias:**

```
APPDYNAMICS_CONTROLLER_HOST
APPDYNAMICS_CONTROLLER_PORT
APPDYNAMICS_CONTROLLER_SSL (true/false)
APPDYNAMICS_ACCOUNT_NAME
APPDYNAMICS_ACCOUNT_ACCESS_KEY
APPDYNAMICS_APP_NAME
APPDYNAMICS_TIER_NAME
```

**Variables opcionales:**

```
APPDYNAMICS_NODE_NAME (puede usar $CI_COMMIT_REF_NAME o $CI_PIPELINE_ID)
```

---

## Configuración del Pipeline

### Ejemplo 1: .NET Framework con IIS (Windows Runner)

Ver archivo completo: [.gitlab-ci-framework.yml](.gitlab-ci-framework.yml)

**Estructura básica:**

```yaml
stages:
  - build
  - deploy

variables:
  BUILD_PATH: "./bin/Release"
  APPDYNAMICS_AGENT_PATH: "C:\\AppDynamics\\AppServerAgent"

build:
  stage: build
  tags:
    - windows
  script:
    - echo "Restoring NuGet packages"
    - nuget restore
    - echo "Building application"
    - msbuild MyApp.sln /p:Configuration=Release
    - echo "Configuring AppDynamics agent"
    - powershell -File configure-appdynamics.ps1
  artifacts:
    paths:
      - $BUILD_PATH
    expire_in: 1 hour

deploy:
  stage: deploy
  tags:
    - windows
  script:
    - echo "Deploying to IIS"
    - powershell -File deploy-iis.ps1
```

### Ejemplo 2: .NET Core con Docker

Ver archivo completo: [.gitlab-ci-dotnet-core.yml](.gitlab-ci-dotnet-core.yml)

**Estructura básica:**

```yaml
stages:
  - build
  - test
  - deploy

variables:
  DOTNET_VERSION: "6.0"
  IMAGE_NAME: $CI_REGISTRY_IMAGE
  IMAGE_TAG: $CI_COMMIT_REF_SLUG

build:
  stage: build
  image: mcr.microsoft.com/dotnet/sdk:$DOTNET_VERSION
  script:
    - dotnet restore
    - dotnet build --configuration Release
    - dotnet publish -c Release -o ./publish
  artifacts:
    paths:
      - ./publish
    expire_in: 1 hour

deploy:
  stage: deploy
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      docker build 
        --build-arg APPDYNAMICS_CONTROLLER_HOST=$APPDYNAMICS_CONTROLLER_HOST
        --build-arg APPDYNAMICS_CONTROLLER_PORT=$APPDYNAMICS_CONTROLLER_PORT
        --build-arg APPDYNAMICS_ACCOUNT_NAME=$APPDYNAMICS_ACCOUNT_NAME
        --build-arg APPDYNAMICS_ACCOUNT_ACCESS_KEY=$APPDYNAMICS_ACCOUNT_ACCESS_KEY
        --build-arg APPDYNAMICS_APP_NAME=$APPDYNAMICS_APP_NAME
        --build-arg APPDYNAMICS_TIER_NAME=$APPDYNAMICS_TIER_NAME
        --build-arg APPDYNAMICS_NODE_NAME="${CI_COMMIT_REF_NAME}-${CI_PIPELINE_ID}"
        -t $IMAGE_NAME:$IMAGE_TAG .
    - docker push $IMAGE_NAME:$IMAGE_TAG
```

### Ejemplo 3: .NET Core Standalone

Ver archivo completo: [.gitlab-ci-standalone.yml](.gitlab-ci-standalone.yml)

---

## Verificación

### Verificar en Logs del Pipeline

```bash
# Buscar mensajes de AppDynamics
grep -i "appdynamics\|instrumentation" pipeline.log

# Verificar configuración
grep -i "controller\|account" pipeline.log
```

### Verificar en el Controller de AppDynamics

1. Acceder al portal de AppDynamics
2. Navegar a: **Applications > [Nombre de tu aplicación]**
3. Verificar que el **Tier** y **Node** aparecen como **Up** (verde)
4. Verificar métricas en tiempo real

### Verificar en la Aplicación

- Revisar logs de la aplicación
- Verificar que no hay errores de instrumentación
- Verificar que las transacciones aparecen en AppDynamics

---

## Troubleshooting

### Problema: El agente no se carga

**Soluciones:**
1. Verificar que el agente está instalado
2. Verificar configuración en `web.config` o variables de entorno
3. Verificar logs del agente
4. Verificar que las DLLs del agente están en el PATH

### Problema: No se conecta al Controller

**Soluciones:**
1. Verificar variables de CI/CD están configuradas
2. Verificar conectividad de red
3. Verificar Access Key
4. Verificar firewall

### Problema: Errores en el Pipeline

**Soluciones:**
1. Verificar permisos del GitLab Runner
2. Verificar que las variables de CI/CD existen
3. Verificar logs del pipeline
4. Verificar que el agente se descarga/instala correctamente

---

**Última actualización:** Enero 2025
