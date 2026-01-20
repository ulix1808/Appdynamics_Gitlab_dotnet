# AppDynamics para .NET - Automatización con GitLab CI/CD

Este repositorio contiene documentación y ejemplos para automatizar la instrumentación del agente de AppDynamics en aplicaciones .NET mediante pipelines de GitLab CI/CD.

## 📋 Contenido

- [Requisitos](#requisitos)
- [Instalación Manual](#instalación-manual)
- [Automatización con GitLab CI/CD](#automatización-con-gitlab-cicd)
- [Configuración](#configuración)
- [Ejemplos de Pipelines](#ejemplos-de-pipelines)
- [Solución de Problemas](#solución-de-problemas)
- [Referencias](#referencias)

## 🚀 Quick Start

Para una guía rápida, consulta [QUICK_START.md](QUICK_START.md)

Para documentación detallada, consulta [INSTRUMENTACION_DOTNET.md](INSTRUMENTACION_DOTNET.md)

---

## Requisitos

### Requisitos del Sistema

- **.NET Framework** 4.5+ o **.NET Core** 3.1+ / **.NET** 5.0+
- **Sistema Operativo**: Windows Server, Linux (para .NET Core)
- **GitLab Runner**: Configurado para ejecutar pipelines CI/CD
- **Permisos**: Acceso para modificar archivos de configuración y ejecutar instrumentación

### Requisitos de AppDynamics

- **Controller AppDynamics**: Acceso al Controller (hostname, puerto, cuenta)
- **Account Name**: Nombre de cuenta de AppDynamics
- **Access Key**: Access Key de AppDynamics
- **Application Name**: Nombre de la aplicación en AppDynamics
- **Tier Name**: Nombre del tier (ej: "DotNet-WebAPI")
- **Node Name**: Nombre del nodo (único por instancia)

### Requisitos de GitLab

- **GitLab CI/CD**: Habilitado en el proyecto
- **GitLab Runner**: Configurado y funcionando
- **Variables de CI/CD**: Configuradas en GitLab (Controller info, Access Key, etc.)

---

## Instalación Manual

### Opción 1: .NET Framework

**Paso 1: Descargar el Agente .NET**

1. Acceder al portal de AppDynamics
2. **Settings > Downloads**
3. Descargar: **.NET Agent**
4. Extraer el archivo ZIP

**Paso 2: Instalar el Agente**

```powershell
# Ejecutar instalador
.\AppServerAgent-x64-Install.exe /S /D=C:\AppDynamics\AppServerAgent

# O instalar manualmente copiando archivos
```

**Paso 3: Configurar el Agente**

Editar: `C:\AppDynamics\AppServerAgent\conf\controller-info.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<controller-info>
    <controller-host>controller.appdynamics.com</controller-host>
    <controller-port>8090</controller-port>
    <controller-ssl-enabled>false</controller-ssl-enabled>
    <account-name>customer1</account-name>
    <account-access-key>xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx</account-access-key>
    <application-name>DotNet-Application</application-name>
    <tier-name>DotNet-WebAPI-Tier</tier-name>
    <node-name>DotNet-Node-01</node-name>
</controller-info>
```

**Paso 4: Instrumentar la Aplicación**

Modificar `web.config` o `app.config`:

```xml
<configuration>
  <system.web>
    <httpModules>
      <add name="AppDynamicsAgent" type="AppDynamics.Agent.AgentHttpModule" />
    </httpModules>
  </system.web>
  <system.webServer>
    <modules>
      <add name="AppDynamicsAgent" type="AppDynamics.Agent.AgentHttpModule" />
    </modules>
  </system.webServer>
</configuration>
```

### Opción 2: .NET Core / .NET 5+

**Paso 1: Instalar NuGet Package**

```bash
dotnet add package AppDynamics.Agent
```

**Paso 2: Configurar en `Program.cs` o `Startup.cs`**

```csharp
using AppDynamics.Agent;

public class Program
{
    public static void Main(string[] args)
    {
        // Configurar AppDynamics antes de crear el host
        AppDynamicsConfig.Configure();
        
        CreateHostBuilder(args).Build().Run();
    }
    
    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

**Paso 3: Configurar Variables de Entorno**

```bash
export APPDYNAMICS_AGENT_APPLICATION_NAME="DotNet-Application"
export APPDYNAMICS_AGENT_TIER_NAME="DotNet-WebAPI-Tier"
export APPDYNAMICS_AGENT_NODE_NAME="DotNet-Node-01"
export APPDYNAMICS_CONTROLLER_HOST_NAME="controller.appdynamics.com"
export APPDYNAMICS_CONTROLLER_PORT="8090"
export APPDYNAMICS_CONTROLLER_SSL_ENABLED="false"
export APPDYNAMICS_AGENT_ACCOUNT_NAME="customer1"
export APPDYNAMICS_AGENT_ACCOUNT_ACCESS_KEY="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

---

## Automatización con GitLab CI/CD

### Concepto General

Los pipelines de GitLab CI/CD pueden automatizar:
1. **Descarga del agente** desde el portal de AppDynamics
2. **Configuración automática** del agente con variables de CI/CD
3. **Instrumentación** de la aplicación durante el build
4. **Despliegue** con el agente ya configurado

### Variables de CI/CD Requeridas en GitLab

Configurar en: **Settings > CI/CD > Variables**

| Variable | Descripción | Protegida | Enmascarada |
|----------|-------------|-----------|-------------|
| `APPDYNAMICS_CONTROLLER_HOST` | Host del Controller | ✅ | ❌ |
| `APPDYNAMICS_CONTROLLER_PORT` | Puerto del Controller | ✅ | ❌ |
| `APPDYNAMICS_CONTROLLER_SSL` | SSL habilitado (true/false) | ✅ | ❌ |
| `APPDYNAMICS_ACCOUNT_NAME` | Nombre de cuenta | ✅ | ✅ |
| `APPDYNAMICS_ACCOUNT_ACCESS_KEY` | Access Key | ✅ | ✅ |
| `APPDYNAMICS_APP_NAME` | Nombre de la aplicación | ✅ | ❌ |
| `APPDYNAMICS_TIER_NAME` | Nombre del tier | ✅ | ❌ |
| `APPDYNAMICS_NODE_NAME` | Nombre del nodo (puede usar CI variables) | ✅ | ❌ |
| `APPDYNAMICS_RUM_APP_KEY` | RUM Application Key (opcional para pipelines con Server Agent, **requerido** para RUM-only) | ✅ | ✅ |
| `APPDYNAMICS_RUM_BEACON_URL` | RUM Beacon URL (opcional) | ✅ | ❌ |
| `IIS_APP_POOL_NAME` | Nombre del Application Pool de IIS (opcional, solo Windows) | ✅ | ❌ |

**Variables útiles de GitLab CI:**
- `CI_COMMIT_REF_NAME` - Nombre del branch
- `CI_JOB_NAME` - Nombre del job
- `CI_ENVIRONMENT_NAME` - Nombre del ambiente
- `CI_PIPELINE_ID` - ID del pipeline

---

## Ejemplos de Pipelines

### Ejemplo 1: ASP.NET con RUM (Recomendado)

Ver archivo: [.gitlab-ci-aspnet.yml](.gitlab-ci-aspnet.yml)

**Características:**
- ✅ Server Agent (monitoreo de backend)
- ✅ **RUM (Real User Monitoring)** (monitoreo de frontend/browser)
- ✅ Generación automática de scripts RUM
- ✅ Reinicio automático de Application Pool

**Ideal para:** Aplicaciones web ASP.NET que requieren monitoreo completo end-to-end.

**Variables requeridas:**
- `APPDYNAMICS_RUM_APP_KEY` - RUM Application Key de AppDynamics Controller

### Ejemplo 2: .NET Framework con IIS

Ver archivo: [.gitlab-ci-framework.yml](.gitlab-ci-framework.yml)

**Incluye RUM opcionalmente:**
- Si `APPDYNAMICS_RUM_APP_KEY` está configurado → RUM se configura automáticamente
- Si `APPDYNAMICS_RUM_APP_KEY` NO está configurado → Solo se configura Server Agent

### Ejemplo 3: .NET Core / .NET 5+ con Docker

Ver archivo: [.gitlab-ci-dotnet-core.yml](.gitlab-ci-dotnet-core.yml)

### Ejemplo 4: .NET Core / .NET 5+ Standalone

Ver archivo: [.gitlab-ci-standalone.yml](.gitlab-ci-standalone.yml)

### Ejemplo 5: SOLO RUM (Sin Server Agent)

Ver archivo: [.gitlab-ci-rum-only.yml](.gitlab-ci-rum-only.yml)

**Características:**
- ✅ **SOLO RUM** (monitoreo de frontend/browser)
- ✅ **NO requiere** AppDynamics Server Agent instalado
- ✅ Funciona con cualquier aplicación web (ASP.NET, .NET Core, HTML estático, etc.)
- ✅ Generación automática de scripts RUM
- ✅ Snippets de integración para ASP.NET y HTML genérico

**Ideal para:**
- Aplicaciones que solo necesitan monitoreo de frontend
- Aplicaciones estáticas o con backend no instrumentado
- Pruebas de RUM sin necesidad de Server Agent

**Variables requeridas:**
- `APPDYNAMICS_RUM_APP_KEY` - **Requerido** (RUM Application Key)
- `APPDYNAMICS_CONTROLLER_HOST` - **Requerido** (Controller hostname)
- `APPDYNAMICS_APP_NAME` - Opcional (default: "MyApplication")
- `APPDYNAMICS_TIER_NAME` - Opcional (default: "Frontend")

---

## Configuración

### Configuración del Agente

El agente puede configurarse mediante:
1. **Archivo XML**: `controller-info.xml`
2. **Variables de Entorno**: Para .NET Core
3. **Variables de CI/CD**: Automatizado en pipelines

### Configuración de la Aplicación

**Para .NET Framework:**
- Modificar `web.config` o `app.config`
- Agregar módulos HTTP de AppDynamics

**Para .NET Core:**
- Instalar NuGet package `AppDynamics.Agent`
- Configurar en `Program.cs` o `Startup.cs`
- Usar variables de entorno

---

## Verificación

### Verificar Instrumentación

1. **En Logs de Build:**
   ```bash
   # Buscar mensajes de AppDynamics
   grep -i "appdynamics\|instrumentation" build.log
   ```

2. **En el Controller de AppDynamics:**
   - Navegar a: **Applications > [Nombre de tu aplicación]**
   - Verificar que el **Tier** y **Node** aparecen como **Up** (verde)
   - Verificar métricas en tiempo real

3. **En la Aplicación:**
   - Verificar logs de la aplicación
   - Verificar que no hay errores de instrumentación

---

## Solución de Problemas

### Problema: El agente no se carga

**Soluciones:**
1. Verificar que el agente está instalado correctamente
2. Verificar configuración en `web.config` o `app.config`
3. Verificar variables de entorno para .NET Core
4. Revisar logs del agente: `AppServerAgent\logs\agent.log`

### Problema: No se conecta al Controller

**Soluciones:**
1. Verificar `controller-info.xml` o variables de entorno
2. Verificar conectividad de red al Controller
3. Verificar Access Key
4. Verificar firewall y reglas de seguridad

### Problema: Errores en el Pipeline

**Soluciones:**
1. Verificar que las variables de CI/CD están configuradas
2. Verificar permisos del GitLab Runner
3. Verificar que el agente se descarga correctamente
4. Revisar logs del pipeline en GitLab

---

## RUM (Real User Monitoring)

**RUM** monitorea la experiencia del usuario desde el navegador:
- ⏱️ Tiempo de carga de páginas
- 🔄 Flujos de navegación
- ❌ Errores de JavaScript
- 🌐 Llamadas AJAX/XHR

### Opciones para RUM:

**Opción 1: RUM con Server Agent (Monitoreo Completo)**
1. Configura `APPDYNAMICS_RUM_APP_KEY` en GitLab CI/CD Variables
2. Usa `.gitlab-ci-aspnet.yml` para aplicaciones ASP.NET (RUM + Server Agent automático)
3. O usa `.gitlab-ci-framework.yml` (RUM opcional con Server Agent)

**Opción 2: SOLO RUM (Sin Server Agent)**
1. Configura `APPDYNAMICS_RUM_APP_KEY` y `APPDYNAMICS_CONTROLLER_HOST` en GitLab CI/CD Variables
2. Usa `.gitlab-ci-rum-only.yml` (solo RUM, no requiere Server Agent)
3. Funciona con cualquier aplicación web (ASP.NET, .NET Core, HTML estático, etc.)

**Ver documentación completa:** [RUM_APPDYNAMICS.md](RUM_APPDYNAMICS.md)

## Documentación Adicional

- **Instrumentación Manual:** Ver [INSTRUMENTACION_DOTNET.md](INSTRUMENTACION_DOTNET.md)
- **Guía Rápida:** Ver [QUICK_START.md](QUICK_START.md)
- **RUM (Real User Monitoring):** Ver [RUM_APPDYNAMICS.md](RUM_APPDYNAMICS.md)
- **Reinicio de IIS:** Ver [REINICIO_IIS.md](REINICIO_IIS.md)

## Referencias

- [AppDynamics .NET Agent Documentation](https://docs.appdynamics.com/latest/en/application-monitoring/install-app-server-agents/net-agent)
- [AppDynamics RUM Documentation](https://docs.appdynamics.com/latest/en/end-user-monitoring/javascript-instrumentation)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [.NET Documentation](https://docs.microsoft.com/en-us/dotnet/)

---

**Última actualización:** Enero 2025
