# AppDynamics RUM (Real User Monitoring) - Guía de Configuración

Esta guía explica cómo configurar y usar **AppDynamics RUM (Real User Monitoring)** con los pipelines de GitLab CI/CD para aplicaciones .NET.

## 📋 ¿Qué es RUM?

**Real User Monitoring (RUM)** es el agente JavaScript de AppDynamics que se ejecuta en el navegador del usuario final para monitorear:

- ⏱️ **Tiempo de carga de páginas**
- 🔄 **Flujos de navegación del usuario**
- ❌ **Errores de JavaScript**
- 🌐 **Llamadas AJAX/XHR**
- 📊 **Métricas de rendimiento del frontend**
- 👥 **Experiencia del usuario desde el navegador**

## 🎯 Diferencias: Server Agent vs RUM

| Aspecto | Server Agent | RUM (JavaScript Agent) |
|---------|-------------|------------------------|
| **Ubicación** | Servidor (IIS) | Navegador del usuario |
| **Monitorea** | Backend (.NET) | Frontend (JavaScript) |
| **Instalación** | `controller-info.xml` en servidor | Script JavaScript en HTML |
| **Datos** | Métricas de aplicación, transacciones | Tiempo de carga, navegación, errores JS |

## 🔧 Configuración en Pipelines

### Variables de CI/CD Requeridas para RUM

Configurar en GitLab: **Settings > CI/CD > Variables**

#### Variables Requeridas:

```
APPDYNAMICS_RUM_APP_KEY = "tu-rum-app-key-aqui"
```

#### Variables Opcionales:

```
APPDYNAMICS_RUM_BEACON_URL = "https://controller.example.com/eumcollector"
```

**Nota:** Si `APPDYNAMICS_RUM_BEACON_URL` no se configura, se usará: `https://{APPDYNAMICS_CONTROLLER_HOST}/eumcollector`

### Cómo Obtener el RUM App Key

1. **Inicia sesión en AppDynamics Controller**
2. **Ve a Settings > Instrumentation > RUM**
3. **Selecciona tu aplicación** (o crea una nueva)
4. **Copia el "RUM Application Key"** (también llamado "EUM Application Key")

El RUM App Key tiene un formato como: `AD-AAB-AAA-12345`

## 📦 Pipelines que Incluyen RUM

### 1. `.gitlab-ci-rum-only.yml` (SOLO RUM - Sin Server Agent) ⭐ Nuevo

Este pipeline **configura SOLO RUM** sin requerir AppDynamics Server Agent instalado:

```yaml
# RUM es configurado automáticamente en el stage "configure_rum"
# El script RUM se genera en: ./bin/Release/Scripts/appdynamics-rum.js
# NO requiere Server Agent instalado
```

**Características:**
- ✅ **SOLO RUM** (monitoreo de frontend/browser)
- ✅ **NO requiere** AppDynamics Server Agent instalado
- ✅ Funciona con cualquier aplicación web (ASP.NET, .NET Core, HTML estático, etc.)
- ✅ Genera `appdynamics-rum.js` automáticamente
- ✅ Genera snippets de integración para ASP.NET y HTML genérico
- ✅ Verifica que el script RUM se copie durante el deploy

**Variables Requeridas:**
- `APPDYNAMICS_RUM_APP_KEY` - **Requerido** (RUM Application Key)
- `APPDYNAMICS_CONTROLLER_HOST` - **Requerido** (Controller hostname)

**Variables Opcionales:**
- `APPDYNAMICS_APP_NAME` - Nombre de la aplicación (default: "MyApplication")
- `APPDYNAMICS_TIER_NAME` - Nombre del tier (default: "Frontend")
- `APPDYNAMICS_RUM_BEACON_URL` - URL del beacon (default: `https://{CONTROLLER_HOST}/eumcollector`)
- `IIS_APP_POOL_NAME` - Nombre del Application Pool de IIS (solo Windows/IIS)

**Cuándo Usar Este Pipeline:**
- ✅ Solo necesitas monitoreo de frontend/browser
- ✅ No tienes o no necesitas Server Agent instalado
- ✅ Aplicaciones estáticas o con backend no instrumentado
- ✅ Pruebas de RUM sin necesidad de Server Agent
- ✅ Frontend independiente del backend

**Ver archivo completo:** [.gitlab-ci-rum-only.yml](.gitlab-ci-rum-only.yml)

### 2. `.gitlab-ci-aspnet.yml` (ASP.NET - RUM + Server Agent)

Este pipeline **incluye RUM automáticamente** junto con Server Agent para aplicaciones ASP.NET:

```yaml
# RUM es configurado automáticamente en el stage "configure_appdynamics"
# El script RUM se genera en: ./bin/Release/Scripts/appdynamics-rum.js
# Server Agent también se configura para monitoreo completo end-to-end
```

**Características:**
- ✅ Server Agent (monitoreo de backend)
- ✅ RUM (monitoreo de frontend/browser)
- ✅ Genera `appdynamics-rum.js` automáticamente
- ✅ Incluye configuración completa de RUM
- ✅ Genera snippet de integración para ASP.NET
- ✅ Verifica que el script RUM se copie durante el deploy
- ✅ Reinicio automático de Application Pool

**Variables Requeridas:**
- `APPDYNAMICS_RUM_APP_KEY` - **Requerido** (RUM Application Key)
- `APPDYNAMICS_CONTROLLER_HOST` - **Requerido** (Controller hostname)
- `APPDYNAMICS_ACCOUNT_ACCESS_KEY` - **Requerido** (Account Access Key para Server Agent)

**Ideal para:** Aplicaciones web ASP.NET que requieren monitoreo completo end-to-end (backend + frontend).

**Ver archivo completo:** [.gitlab-ci-aspnet.yml](.gitlab-ci-aspnet.yml)

### 3. `.gitlab-ci-framework.yml` (Framework - RUM Opcional)

Este pipeline **incluye RUM opcionalmente** junto con Server Agent:

- Si `APPDYNAMICS_RUM_APP_KEY` está configurado → RUM se configura automáticamente
- Si `APPDYNAMICS_RUM_APP_KEY` NO está configurado → Solo se configura Server Agent

**Variables Requeridas (Server Agent):**
- `APPDYNAMICS_CONTROLLER_HOST` - **Requerido**
- `APPDYNAMICS_ACCOUNT_ACCESS_KEY` - **Requerido**

**Variables Opcionales (RUM):**
- `APPDYNAMICS_RUM_APP_KEY` - Si está configurado, RUM se agrega automáticamente

**Ver archivo completo:** [.gitlab-ci-framework.yml](.gitlab-ci-framework.yml)

## 🚀 Integración en Aplicaciones .NET

### Opción 1: Integración en ASP.NET MVC / Razor Pages

**En tu `_Layout.cshtml` (o Master Page):**

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>@ViewBag.Title - My ASP.NET App</title>
    
    <!-- AppDynamics RUM JavaScript Agent -->
    <!-- Agregar ANTES del cierre de </head> -->
    <script src="~/Scripts/appdynamics-rum.js"></script>
    
    <!-- O usando CDN directamente: -->
    <!--
    <script>
        window['adrum-start-time'] = new Date().getTime();
        (function(config){
            config.appKey = '@ConfigurationManager.AppSettings["AppDynamics.RUM.AppKey"]';
            config.adrumExtUrlHttp = 'http://@ConfigurationManager.AppSettings["AppDynamics.Controller.Host"]/eumcollector';
            config.adrumExtUrlHttps = 'https://@ConfigurationManager.AppSettings["AppDynamics.Controller.Host"]/eumcollector';
            config.applicationName = '@ConfigurationManager.AppSettings["AppDynamics.Application.Name"]';
            config.tierName = '@ConfigurationManager.AppSettings["AppDynamics.Tier.Name"]';
        })(window['adrum-config'] || (window['adrum-config'] = {}));
    </script>
    <script src="https://@ConfigurationManager.AppSettings["AppDynamics.Controller.Host"]/adrum/adrum-latest.js"></script>
    -->
</head>
<body>
    <!-- Tu contenido aquí -->
</body>
</html>
```

### Opción 2: Integración en Web Forms

**En tu `Site.Master` o página principal:**

```html
<head runat="server">
    <title>My Web Forms App</title>
    
    <!-- AppDynamics RUM JavaScript Agent -->
    <script src="~/Scripts/appdynamics-rum.js"></script>
</head>
```

### Opción 3: Integración usando web.config (Global)

**En `web.config`, agregar en `<system.webServer>`:**

```xml
<system.webServer>
    <handlers>
        <!-- Otros handlers -->
    </handlers>
    
    <!-- Agregar RUM script a todas las páginas -->
    <rewrite>
        <outboundRules>
            <rule name="AppDynamics RUM Script" preCondition="IsHTML">
                <match filterByTags="None" pattern="(&lt;/head&gt;)" />
                <action type="Rewrite" value="&lt;script src=&quot;~/Scripts/appdynamics-rum.js&quot;&gt;&lt;/script&gt;$0" />
            </rule>
            <preConditions>
                <preCondition name="IsHTML">
                    <add input="{RESPONSE_CONTENT_TYPE}" pattern="^text/html" />
                </preCondition>
            </preConditions>
        </outboundRules>
    </rewrite>
</system.webServer>
```

**Nota:** Esta opción requiere el módulo URL Rewrite de IIS.

## 📁 Archivos Generados por el Pipeline

### 1. `appdynamics-rum.js`

Ubicación: `./bin/Release/Scripts/appdynamics-rum.js`

Este archivo contiene:
- Configuración de RUM con variables de CI/CD
- Código para cargar el agente RUM desde la CDN de AppDynamics
- Configuración de beacon URLs y application/tier names

### 2. `AppDynamics-RUM-Snippet.txt`

Ubicación: `./bin/Release/AppDynamics-RUM-Snippet.txt`

Este archivo contiene:
- Snippet de código listo para copiar/pegar
- Ejemplos de integración
- Instrucciones básicas

## ✅ Verificación Post-Deploy

### 1. Verificar que el Script RUM está Cargado

**En el navegador (DevTools > Console):**

```javascript
// Verificar que la configuración RUM está presente
console.log(window['adrum-config']);

// Verificar que el script se cargó
console.log(typeof window.ADRUM);
```

### 2. Verificar en Network Tab

- Buscar requests a `/eumcollector` o `/adrum/adrum-latest.js`
- Deben tener status `200 OK`

### 3. Verificar en AppDynamics Controller

1. **Inicia sesión en AppDynamics Controller**
2. **Ve a User Experience > Browser Real User Monitoring**
3. **Selecciona tu aplicación**
4. **Deberías ver métricas de usuarios navegando tu aplicación**

## 🔍 Troubleshooting

### RUM No Está Enviando Datos

**Posibles causas:**

1. **RUM App Key incorrecto**
   - Verifica que `APPDYNAMICS_RUM_APP_KEY` esté correcto en GitLab CI/CD Variables
   - Verifica que el App Key coincida con la aplicación en AppDynamics Controller

2. **Script RUM no está incluido en la página**
   - Verifica que `appdynamics-rum.js` esté en `~/Scripts/appdynamics-rum.js`
   - Verifica que el script esté referenciado en `_Layout.cshtml` o Master Page
   - Verifica en DevTools > Network que el script se carga correctamente

3. **Beacon URL incorrecta**
   - Verifica que `APPDYNAMICS_RUM_BEACON_URL` esté correcto
   - Verifica que el Controller sea accesible desde internet (para que los navegadores puedan enviar datos)

4. **CORS o Firewall bloqueando**
   - Verifica que el Controller permita requests desde navegadores externos
   - Verifica configuración de CORS si es necesario

### Script RUM No se Genera

**Si el script no se genera durante el build:**

1. Verifica que `APPDYNAMICS_RUM_APP_KEY` esté configurado en GitLab CI/CD Variables
2. Verifica los logs del stage `configure_appdynamics` en GitLab CI/CD
3. Verifica que el directorio `./bin/Release/Scripts/` exista después del build

### RUM Se Carga pero No Veo Datos en Controller

**Esperar 5-10 minutos:**
- Los datos de RUM pueden tardar algunos minutos en aparecer en el Controller

**Verificar configuración:**
- Asegúrate de que la Application Name y Tier Name en RUM coincidan con la configuración del Server Agent
- Verifica que la aplicación esté activa en AppDynamics Controller

## 📊 Métricas Disponibles en RUM

Una vez configurado, podrás ver en AppDynamics Controller:

- **Page Load Time:** Tiempo total de carga de páginas
- **AJAX Request Time:** Tiempo de llamadas AJAX/XHR
- **JavaScript Errors:** Errores de JavaScript en el navegador
- **Browser Types:** Distribución de navegadores
- **Geographic Distribution:** Ubicación geográfica de usuarios
- **User Sessions:** Sesiones de usuarios individuales

## 🔗 Recursos Adicionales

- **Documentación oficial de AppDynamics RUM:**
  - https://docs.appdynamics.com/latest/en/application-monitoring/install-app-server-agents/java-agent

- **Ejemplos de configuración RUM:**
  - https://docs.appdynamics.com/latest/en/end-user-monitoring/javascript-instrumentation

## 💡 Mejores Prácticas

1. **Siempre incluir RUM en producción** para monitoreo completo
2. **Usar el script generado** (`appdynamics-rum.js`) en lugar de hardcodear configuración
3. **Verificar que RUM esté funcionando** después de cada deploy
4. **Monitorear métricas de RUM** regularmente para detectar problemas de frontend
5. **Combinar Server Agent + RUM** para monitoreo end-to-end (backend + frontend)

---

**Nota:** El pipeline `.gitlab-ci-aspnet.yml` incluye RUM automáticamente y es la opción recomendada para aplicaciones ASP.NET que requieren monitoreo completo.
