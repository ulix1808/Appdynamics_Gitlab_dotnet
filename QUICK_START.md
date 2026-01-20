# Quick Start: AppDynamics para .NET con GitLab CI/CD

Guía rápida para automatizar la instrumentación de AppDynamics en aplicaciones .NET mediante GitLab CI/CD.

## ⚡ Inicio Rápido (5 Pasos)

### 1. Configurar Variables de CI/CD en GitLab

Ir a: **Settings > CI/CD > Variables**

Agregar variables:
- `APPDYNAMICS_CONTROLLER_HOST` - Host del Controller
- `APPDYNAMICS_CONTROLLER_PORT` - Puerto (8090 o 443)
- `APPDYNAMICS_CONTROLLER_SSL` - true/false
- `APPDYNAMICS_ACCOUNT_NAME` - Nombre de cuenta
- `APPDYNAMICS_ACCOUNT_ACCESS_KEY` - Access Key
- `APPDYNAMICS_APP_NAME` - Nombre de aplicación
- `APPDYNAMICS_TIER_NAME` - Nombre del tier

### 2. Elegir Pipeline Según tu Proyecto

- **.NET Framework + IIS**: Usar `.gitlab-ci-framework.yml`
- **.NET Core con Docker**: Usar `.gitlab-ci-dotnet-core.yml`
- **.NET Core Standalone**: Usar `.gitlab-ci-standalone.yml`

### 3. Instalar Package AppDynamics (Para .NET Core)

```bash
dotnet add package AppDynamics.Agent
```

### 4. Configurar en Código (.NET Core)

En `Program.cs`:

```csharp
using AppDynamics.Agent;

public class Program
{
    public static void Main(string[] args)
    {
        AppDynamicsConfig.Configure(); // Configura desde variables de entorno
        CreateHostBuilder(args).Build().Run();
    }
}
```

### 5. Configurar Pipeline en GitLab

1. Copiar el pipeline apropiado a `.gitlab-ci.yml` en tu proyecto
2. Ajustar según tu configuración (paths, nombres de proyecto, etc.)
3. Hacer commit y push
4. El pipeline se ejecutará automáticamente

## 📚 Documentación Completa

- [README.md](README.md) - Documentación general
- [INSTRUMENTACION_DOTNET.md](INSTRUMENTACION_DOTNET.md) - Guía detallada paso a paso

## 🔍 Verificar Instrumentación

Después del despliegue:

1. **En GitLab CI/CD**: Revisar logs del pipeline
2. **En AppDynamics Controller**: Verificar que Tier y Node aparecen como "Up"
3. **En la Aplicación**: Verificar logs y métricas

---

**Nota:** Los pipelines están configurados para usar variables de CI/CD, asegúrate de configurarlas en GitLab antes de ejecutar.
