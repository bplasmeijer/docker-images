# escape=`
ARG BUILD_IMAGE
ARG BASE_IMAGE

FROM $BUILD_IMAGE as builder

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

COPY . /install/

# Expand zips, prepare SIF config and WDP package
RUN $env:SIF_CONFIG = 'sitecore-XM1-cm.json'; `
    $env:SIF_PACKAGE = 'Sitecore*_cm.scwdp.zip'; `
    $env:SITENAME = 'sc'; `
    $env:INSTALL_TEMP = 'C:\\install'; `    
    Expand-Archive -Path (Join-Path $env:INSTALL_TEMP '*.zip') -DestinationPath $env:INSTALL_TEMP; `
    Expand-Archive -Path (Join-Path $env:INSTALL_TEMP '*Configuration files*.zip') -DestinationPath $env:INSTALL_TEMP; `
    $config = Get-Content (Join-Path $env:INSTALL_TEMP $env:SIF_CONFIG) | ConvertFrom-Json; `
    $config.Tasks.InstallWDP.Params.Arguments | Add-Member -Name 'Skip' -Value @(@{'ObjectName' = 'dbDacFx'}, @{'ObjectName' = 'dbFullSql'}) -MemberType NoteProperty; `
    $config.Variables.'Site.PhysicalPath' = 'C:\inetpub\{0}' -f $env:SITENAME; `
    ConvertTo-Json $config -Depth 50 | Set-Content -Path (Join-Path $env:INSTALL_TEMP $env:SIF_CONFIG); `
    Rename-Item -Path (Resolve-Path (Join-Path $env:INSTALL_TEMP $env:SIF_CONFIG)).Path -NewName 'config.json'; `
    Rename-Item -Path (Resolve-Path (Join-Path $env:INSTALL_TEMP $env:SIF_PACKAGE)).Path -NewName 'package.zip';

FROM $BASE_IMAGE

# BUG: Target path needs to exist when building with Docker Engine 19.03.1, see https://github.com/docker/for-win/issues/4349. Should be removed when fixed.
RUN MKDIR C:\install
COPY --from=builder ["/install/config.json", "/install/package.zip", "/install/license.xml", "/install/"]
COPY --from=builder ["/install/Sitecore", "/Sitecore"]

# Install Sitecore, apply tweaks and cleanup
RUN $env:SITENAME = 'sc'; `
    $env:INSTALL_TEMP = 'C:\\install'; `
    $env:SITECORE_ADMIN_PASSWORD = 'b'; `
    $env:SQL_ADMIN_USER = 'sa'; `
    $env:SQL_ADMIN_PASSWORD = 'HASH-epsom-sunset-cost7!'; `
    $env:SQL_HOST = 'sql'; `
    $env:SQL_DB_PREFIX = 'sc'; `
    $env:SOLR_URL = 'http://solr:8983/solr'; `
    $env:IDENTITYSERVER_URL = 'https://127.0.0.1'; `
    $env:IDENTITYSERVER_SECRET = 'R6mYfCbM4Ci0niMBuzKVHIKc'; `
    Install-SitecoreConfiguration -Path (Join-Path $env:INSTALL_TEMP '\\config.json') -Package (Join-Path $env:INSTALL_TEMP '\\package.zip') `
    -LicenseFile (Join-Path $env:INSTALL_TEMP '\\license.xml') `
    -Sitename $env:SITENAME `
    -SitecoreAdminPassword $env:SITECORE_ADMIN_PASSWORD `
    -SitecoreIdentityAuthority $env:IDENTITYSERVER_URL `
    -SitecoreIdentitySecret $env:IDENTITYSERVER_SECRET `
    -SolrUrl $env:SOLR_URL `
    -SolrCorePrefix $env:SQL_DB_PREFIX `
    -SqlServer $env:SQL_HOST `
    -SqlDbPrefix $env:SITENAME `
    -SqlAdminUser $env:SQL_ADMIN_USER -SqlAdminPassword $env:SQL_ADMIN_PASSWORD `
    -SqlSecurityUser $env:SQL_ADMIN_USER -SqlSecurityPassword $env:SQL_ADMIN_PASSWORD `
    -SqlCoreUser $env:SQL_ADMIN_USER -SqlCorePassword $env:SQL_ADMIN_PASSWORD `
    -SqlMasterUser $env:SQL_ADMIN_USER -SqlMasterPassword $env:SQL_ADMIN_PASSWORD `
    -SqlWebUser $env:SQL_ADMIN_USER -SqlWebPassword $env:SQL_ADMIN_PASSWORD `
    -SqlFormsUser $env:SQL_ADMIN_USER -SqlFormsPassword $env:SQL_ADMIN_PASSWORD `
    -Skip "RemoveDefaultBinding", "CreateBindingsWithThumbprint", "CreateHostHeader", "CreateBindingsWithDevelopmentThumbprint", "UpdateSolrSchema", "DisplayPassword"; `
    $iisPath = ('IIS:\Sites\{0}' -f $env:SITENAME); `
    Clear-WebConfiguration -PSPath $iisPath -Filter '/system.webserver/rewrite/rules/rule'; `
    Set-WebConfiguration -PSPath $iisPath -Filter '/system.web/customErrors/@mode' -Value 'Off'; `
    Add-WebConfigurationProperty -PSPath $iisPath -Filter '/system.webServer/rewrite/outboundRules' -Name '.' -Value @{name = 'MakeLocationHeaderRelative' ; preCondition = 'IsSitecoreAbsoluteRedirect'; match = @{serverVariable = 'RESPONSE_LOCATION'; pattern = '(https?://[^:/]+):?([0-9]+)?(.*)'}; action = @{type = 'Rewrite'; value = '{R:3}'}}; `
    Add-WebConfigurationProperty -PSPath $iisPath -Filter '/system.webServer/rewrite/outboundRules/preConditions' -Name '.' -Value @{name = 'IsSitecoreAbsoluteRedirect'}; `
    Add-WebConfigurationProperty -PSPath $iisPath -Filter 'system.webServer/rewrite/outboundRules/preConditions/preCondition[@name=''IsSitecoreAbsoluteRedirect'']' -Name '.' -Value @{input = '{RESPONSE_LOCATION}'; pattern = '(https?://[^:/]+):?([0-9]+)?/sitecore/(.*)'}; `
    Add-WebConfigurationProperty -PSPath $iisPath -Filter 'system.webServer/rewrite/outboundRules/preConditions/preCondition[@name=''IsSitecoreAbsoluteRedirect'']' -Name '.' -Value @{input = '{RESPONSE_STATUS}'; pattern = '3[0-9][0-9]'}; `
    Add-LocalGroupMember -Group 'Performance Monitor Users' -Member ('IIS AppPool\{0}' -f $env:SITENAME); `
    Remove-Item -Path 'C:\\*.log'; `
    Remove-Item -Path ('C:\\inetpub\\{0}\\App_Data\\logs' -f $env:SITENAME) -Force -Recurse; `
    Remove-Item -Path $env:INSTALL_TEMP -Force -Recurse;

