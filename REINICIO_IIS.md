# Reinicio de IIS en Pipelines de GitLab CI/CD

Esta guía explica cómo reiniciar IIS (Application Pool y sitios web) dentro de un pipeline de GitLab CI/CD cuando se despliega una aplicación .NET Framework.

## 📋 Opciones de Reinicio

### Opción 1: Reiniciar Application Pool (Recomendado)

Es la opción **más común y menos disruptiva**. Solo reinicia el Application Pool sin afectar otros sitios web.

```powershell
# Detener Application Pool
Import-Module WebAdministration
Stop-WebAppPool -Name "MyAppPool"
Start-Sleep -Seconds 3

# Copiar archivos de aplicación
Copy-Item -Path ".\bin\Release\*" -Destination "C:\inetpub\wwwroot\MyApp" -Recurse -Force

# Reiniciar Application Pool
Start-WebAppPool -Name "MyAppPool"
Start-Sleep -Seconds 5

# Verificar que se inició correctamente
$poolState = (Get-WebAppPoolState -Name "MyAppPool").Value
if ($poolState -eq "Started") {
    echo "✅ Application Pool iniciado correctamente"
}
```

### Opción 2: Reiniciar Sitio Web Completo

Más agresivo, reinicia todo el sitio web (puede afectar todas las aplicaciones en ese sitio).

```powershell
# Detener sitio web
Stop-Website -Name "MyWebsite"

# Copiar archivos
Copy-Item -Path ".\bin\Release\*" -Destination "C:\inetpub\wwwroot\MyApp" -Recurse -Force

# Reiniciar sitio web
Start-Website -Name "MyWebsite"
```

### Opción 3: Reinicio con Appcmd (Alternativa)

Usando `appcmd.exe` de IIS:

```powershell
# Reiniciar Application Pool
& C:\Windows\System32\inetsrv\appcmd.exe recycle apppool "MyAppPool"

# O reiniciar sitio web
& C:\Windows\System32\inetsrv\appcmd.exe recycle apppool /apppool.name:"MyAppPool"
```

## 🔧 Configuración en Pipeline

### Variables de CI/CD Requeridas

Configurar en GitLab: **Settings > CI/CD > Variables**

```
IIS_APP_POOL_NAME = "MyAppPool"          # Nombre del Application Pool
IIS_WEBSITE_NAME = "MyWebsite"            # Nombre del sitio web (opcional)
IIS_DEPLOY_PATH = "C:\inetpub\wwwroot\MyApp"  # Ruta de despliegue
```

### Pipeline Actualizado

El pipeline `.gitlab-ci-framework.yml` ya incluye:

1. **Detener Application Pool** antes de copiar archivos
2. **Copiar archivos** de la aplicación
3. **Reiniciar Application Pool** después de copiar

**Ejemplo del stage deploy:**

```yaml
deploy:
  stage: deploy
  tags:
    - windows
  script:
    - $appPoolName = $env:IIS_APP_POOL_NAME
    - $deployPath = $env:IIS_DEPLOY_PATH
    
    # Detener Application Pool
    - Import-Module WebAdministration
    - Stop-WebAppPool -Name $appPoolName
    - Start-Sleep -Seconds 3
    
    # Copiar archivos
    - Copy-Item -Path "$BUILD_PATH\*" -Destination $deployPath -Recurse -Force
    
    # Reiniciar Application Pool
    - Start-WebAppPool -Name $appPoolName
    - Start-Sleep -Seconds 5
    
    # Verificar estado
    - $poolState = (Get-WebAppPoolState -Name $appPoolName).Value
    - if ($poolState -eq "Started") { echo "✅ Application Pool iniciado" }
```

## ⚠️ Consideraciones Importantes

### ¿Cuándo se reinicia?

**El reinicio ocurre automáticamente en el stage `deploy`** después de:
1. Detener el Application Pool
2. Copiar los archivos nuevos
3. Reiniciar el Application Pool

### Permisos Requeridos

El usuario que ejecuta el GitLab Runner debe tener:
- Permisos para **detener/iniciar Application Pools** en IIS
- Permisos de **escritura** en la ruta de despliegue
- Módulo **WebAdministration** disponible en PowerShell

### Tiempo de Inactividad

- **Application Pool**: ~5-10 segundos de inactividad durante el reinicio
- **Sitio Web Completo**: ~10-30 segundos (depende de la aplicación)

### Mejores Prácticas

1. **Usar Application Pool** en lugar de sitio web completo (menos disruptivo)
2. **Verificar estado** después del reinicio
3. **Agregar delays** (`Start-Sleep`) para asegurar que los procesos terminen
4. **Manejar errores** si el Application Pool no existe o falla

## 🔍 Verificación Post-Deploy

```powershell
# Verificar estado del Application Pool
$poolState = (Get-WebAppPoolState -Name "MyAppPool").Value
echo "Estado del Application Pool: $poolState"

# Verificar sitio web
$siteState = (Get-Website -Name "MyWebsite").State
echo "Estado del sitio web: $siteState"

# Verificar que la aplicación responde
$response = Invoke-WebRequest -Uri "http://localhost/MyApp" -UseBasicParsing -TimeoutSec 10
if ($response.StatusCode -eq 200) {
    echo "✅ Aplicación respondiendo correctamente"
}
```

## 📝 Personalización

Para personalizar el nombre del Application Pool, agrega una variable de CI/CD:

1. **En GitLab**: Settings > CI/CD > Variables
2. **Variable**: `IIS_APP_POOL_NAME`
3. **Value**: El nombre de tu Application Pool (ej: "MyAppPool")

Luego en el pipeline:

```powershell
$appPoolName = $env:IIS_APP_POOL_NAME
# O usar valor por defecto
$appPoolName = if ($env:IIS_APP_POOL_NAME) { $env:IIS_APP_POOL_NAME } else { "MyAppPool" }
```

---

**Nota:** El pipeline actual incluye el reinicio automático del Application Pool. Solo necesitas ajustar las variables `IIS_APP_POOL_NAME` y `IIS_DEPLOY_PATH` según tu configuración.
