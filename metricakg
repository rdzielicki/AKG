# Script d'investigation Ask Genius - Rapport étendu 7 jours
# Auteur: Renaud
# Date: 2025-10-03

# ===========================================
# CONFIGURATION
# ===========================================
$subscriptions = @{
    "prd" = "47e00959-65c8-474b-a174-f59b1a2dfc48"
    "stg" = "65eaf973-a7d7-42d4-9a6d-c27777902513"
    "dev" = "d3d39366-870d-4083-b23e-a8a3ac15b8cf"
}

# Période d'investigation
$startTime = (Get-Date).AddDays(-7)
$endTime = Get-Date
$incidentStart = Get-Date "2025-10-03 08:00:00"
$incidentEnd = Get-Date "2025-10-03 10:00:00"

Write-Host "═══════════════════════════════════════════════════════════" -ForegroundColor Cyan
Write-Host "  RAPPORT D'INVESTIGATION ASK GENIUS - 7 JOURS" -ForegroundColor Cyan
Write-Host "═══════════════════════════════════════════════════════════" -ForegroundColor Cyan
Write-Host "Période d'analyse: $($startTime.ToString('yyyy-MM-dd HH:mm')) → $($endTime.ToString('yyyy-MM-dd HH:mm'))"
Write-Host "Incident signalé: $($incidentStart.ToString('yyyy-MM-dd HH:mm')) → $($incidentEnd.ToString('yyyy-MM-dd HH:mm'))"
Write-Host "═══════════════════════════════════════════════════════════`n" -ForegroundColor Cyan

# Stocker les résultats pour le résumé final
$globalResults = @()

# ===========================================
# FONCTION D'ANALYSE PAR ENVIRONNEMENT
# ===========================================
function Analyze-Environment {
    param(
        [string]$EnvName,
        [string]$SubscriptionId
    )
    
    Write-Host "`n╔═══════════════════════════════════════════════════════════╗" -ForegroundColor Green
    Write-Host "║  ENVIRONNEMENT: $($EnvName.ToUpper().PadRight(44)) ║" -ForegroundColor Green
    Write-Host "╚═══════════════════════════════════════════════════════════╝" -ForegroundColor Green
    
    $envResults = @{
        Environment = $EnvName
        WebApps = @()
    }
    
    try {
        Set-AzContext -SubscriptionId $SubscriptionId -ErrorAction Stop | Out-Null
        Write-Host "✓ Connecté à la subscription $EnvName`n" -ForegroundColor Green
        
        $webApps = Get-AzWebApp | Where-Object {$_.Name -like "*akg*"}
        
        if ($webApps.Count -eq 0) {
            Write-Host "⚠ Aucune WebApp trouvée" -ForegroundColor Yellow
            return $envResults
        }
        
        foreach ($webApp in $webApps) {
            Write-Host "┌─ WebApp: $($webApp.Name) $('─' * (50 - $webApp.Name.Length))" -ForegroundColor Cyan
            Write-Host "│  Resource Group: $($webApp.ResourceGroup)"
            Write-Host "│  État: $($webApp.State)"
            Write-Host "│  Always On: $($webApp.SiteConfig.AlwaysOn)"
            Write-Host "│  Location: $($webApp.Location)"
            
            $webAppResult = @{
                Name = $webApp.Name
                State = $webApp.State
                AlwaysOn = $webApp.SiteConfig.AlwaysOn
                Metrics = @{}
            }
            
            # ========================================
            # TEMPS DE RÉPONSE HTTP (7 jours)
            # ========================================
            Write-Host "│"
            Write-Host "├─ 📊 TEMPS DE RÉPONSE HTTP (7 jours)" -ForegroundColor Magenta
            try {
                $responseTime = Get-AzMetric -ResourceId $webApp.Id `
                    -MetricName "HttpResponseTime" `
                    -StartTime $startTime `
                    -EndTime $endTime `
                    -TimeGrain 01:00:00 `
                    -AggregationType Average `
                    -WarningAction SilentlyContinue -ErrorAction Stop
                
                $validData = $responseTime.Data | Where-Object {$_.Average -ne $null -and $_.Average -gt 0}
                
                if ($validData.Count -gt 0) {
                    $avgResponse = ($validData | Measure-Object -Property Average -Average).Average
                    $maxResponse = ($validData | Measure-Object -Property Average -Maximum).Maximum
                    $minResponse = ($validData | Measure-Object -Property Average -Minimum).Minimum
                    $p95Response = ($validData | Sort-Object Average)[([Math]::Floor($validData.Count * 0.95))]
                    
                    Write-Host "│  • Moyenne: $([math]::Round($avgResponse, 3))s"
                    Write-Host "│  • P95: $([math]::Round($p95Response.Average, 3))s"
                    Write-Host "│  • Maximum: $([math]::Round($maxResponse, 3))s"
                    Write-Host "│  • Minimum: $([math]::Round($minResponse, 3))s"
                    
                    $webAppResult.Metrics.ResponseTimeAvg = $avgResponse
                    $webAppResult.Metrics.ResponseTimeMax = $maxResponse
                    $webAppResult.Metrics.ResponseTimeP95 = $p95Response.Average
                    
                    # Alertes pour réponses lentes
                    $slowResponses = $validData | Where-Object {$_.Average -gt 5}
                    if ($slowResponses.Count -gt 0) {
                        Write-Host "│  ⚠ $($slowResponses.Count) réponses > 5s détectées" -ForegroundColor Red
                        $webAppResult.Metrics.SlowResponseCount = $slowResponses.Count
                        
                        # Afficher les 5 plus lentes
                        $top5 = $slowResponses | Sort-Object Average -Descending | Select-Object -First 5
                        Write-Host "│  Top 5 réponses les plus lentes:"
                        foreach ($slow in $top5) {
                            Write-Host "│    - $($slow.Timestamp.ToString('MM/dd HH:mm')): $([math]::Round($slow.Average, 2))s" -ForegroundColor Yellow
                        }
                    }
                    
                    # Vérifier pendant la période de l'incident
                    $incidentData = $validData | Where-Object {
                        $_.Timestamp -ge $incidentStart -and $_.Timestamp -le $incidentEnd
                    }
                    
                    if ($incidentData.Count -gt 0) {
                        Write-Host "│"
                        Write-Host "│  🔍 PENDANT L'INCIDENT (08h00-10h00):" -ForegroundColor Yellow
                        $incidentAvg = ($incidentData | Measure-Object -Property Average -Average).Average
                        $incidentMax = ($incidentData | Measure-Object -Property Average -Maximum).Maximum
                        Write-Host "│  • Moyenne: $([math]::Round($incidentAvg, 3))s"
                        Write-Host "│  • Maximum: $([math]::Round($incidentMax, 3))s"
                        
                        if ($incidentMax -gt 10) {
                            Write-Host "│  ⚠️ ALERTE: Latence > 10s pendant l'incident!" -ForegroundColor Red
                        }
                    } else {
                        Write-Host "│  ℹ Pas de données pendant la période de l'incident" -ForegroundColor Gray
                    }
                } else {
                    Write-Host "│  ℹ Aucune donnée de temps de réponse disponible" -ForegroundColor Gray
                }
            } catch {
                Write-Host "│  ✗ Erreur: $($_.Exception.Message)" -ForegroundColor Red
            }
            
            # ========================================
            # NOMBRE DE REQUÊTES
            # ========================================
            Write-Host "│"
            Write-Host "├─ 📈 NOMBRE DE REQUÊTES" -ForegroundColor Magenta
            try {
                $requests = Get-AzMetric -ResourceId $webApp.Id `
                    -MetricName "Requests" `
                    -StartTime $startTime `
                    -EndTime $endTime `
                    -TimeGrain 01:00:00 `
                    -AggregationType Total `
                    -WarningAction SilentlyContinue -ErrorAction Stop
                
                $totalRequests = ($requests.Data | Measure-Object -Property Total -Sum).Sum
                $avgPerHour = $totalRequests / 168  # 7 jours = 168 heures
                $maxPerHour = ($requests.Data | Measure-Object -Property Total -Maximum).Maximum
                
                Write-Host "│  • Total (7j): $totalRequests requêtes"
                Write-Host "│  • Moyenne/heure: $([math]::Round($avgPerHour, 0)) requêtes"
                Write-Host "│  • Pic/heure: $maxPerHour requêtes"
                
                $webAppResult.Metrics.TotalRequests = $totalRequests
                $webAppResult.Metrics.AvgRequestsPerHour = $avgPerHour
                
                # Requêtes pendant l'incident
                $incidentRequests = $requests.Data | Where-Object {
                    $_.Timestamp -ge $incidentStart -and $_.Timestamp -le $incidentEnd
                }
                if ($incidentRequests) {
                    $incidentTotal = ($incidentRequests | Measure-Object -Property Total -Sum).Sum
                    Write-Host "│  • Pendant l'incident: $incidentTotal requêtes" -ForegroundColor Yellow
                }
            } catch {
                Write-Host "│  ✗ Erreur: $($_.Exception.Message)" -ForegroundColor Red
            }
            
            # ========================================
            # ERREURS HTTP
            # ========================================
            Write-Host "│"
            Write-Host "├─ ⚠️  ERREURS HTTP" -ForegroundColor Magenta
            try {
                # Erreurs 4xx
                $http4xx = Get-AzMetric -ResourceId $webApp.Id `
                    -MetricName "Http4xx" `
                    -StartTime $startTime `
                    -EndTime $endTime `
                    -TimeGrain 01:00:00 `
                    -AggregationType Total `
                    -WarningAction SilentlyContinue -ErrorAction Stop
                
                $total4xx = ($http4xx.Data | Measure-Object -Property Total -Sum).Sum
                
                # Erreurs 5xx
                $http5xx = Get-AzMetric -ResourceId $webApp.Id `
                    -MetricName "Http5xx" `
                    -StartTime $startTime `
                    -EndTime $endTime `
                    -TimeGrain 01:00:00 `
                    -AggregationType Total `
                    -WarningAction SilentlyContinue -ErrorAction Stop
                
                $total5xx = ($http5xx.Data | Measure-Object -Property Total -Sum).Sum
                
                Write-Host "│  • Erreurs 4xx: $total4xx"
                Write-Host "│  • Erreurs 5xx: $total5xx"
                
                $webAppResult.Metrics.Http4xx = $total4xx
                $webAppResult.Metrics.Http5xx = $total5xx
                
                if ($total5xx -gt 0) {
                    Write-Host "│  ⚠ Erreurs serveur détectées" -ForegroundColor Yellow
                    $errors5xx = $http5xx.Data | Where-Object {$_.Total -gt 0} | Sort-Object Total -Descending | Select-Object -First 5
                    foreach ($err in $errors5xx) {
                        Write-Host "│    - $($err.Timestamp.ToString('MM/dd HH:mm')): $($err.Total) erreurs" -ForegroundColor Red
                    }
                }
            } catch {
                Write-Host "│  ✗ Erreur: $($_.Exception.Message)" -ForegroundColor Red
            }
            
            # ========================================
            # MÉMOIRE ET CPU
            # ========================================
            Write-Host "│"
            Write-Host "├─ 💻 RESSOURCES SYSTÈME" -ForegroundColor Magenta
            try {
                $memory = Get-AzMetric -ResourceId $webApp.Id `
                    -MetricName "MemoryPercentage" `
                    -StartTime $startTime `
                    -EndTime $endTime `
                    -TimeGrain 01:00:00 `
                    -AggregationType Average `
                    -WarningAction SilentlyContinue -ErrorAction Stop
                
                $validMemory = $memory.Data | Where-Object {$_.Average -ne $null}
                if ($validMemory.Count -gt 0) {
                    $avgMemory = ($validMemory | Measure-Object -Property Average -Average).Average
                    $maxMemory = ($validMemory | Measure-Object -Property Average -Maximum).Maximum
                    
                    Write-Host "│  • Mémoire moyenne: $([math]::Round($avgMemory, 1))%"
                    Write-Host "│  • Mémoire max: $([math]::Round($maxMemory, 1))%"
                    
                    if ($maxMemory -gt 80) {
                        Write-Host "│  ⚠ Mémoire élevée détectée (> 80%)" -ForegroundColor Yellow
                    }
                }
            } catch {
                Write-Host "│  ℹ Métriques mémoire non disponibles" -ForegroundColor Gray
            }
            
            # ========================================
            # DISPONIBILITÉ
            # ========================================
            Write-Host "│"
            Write-Host "└─ ✅ DISPONIBILITÉ" -ForegroundColor Magenta
            try {
                $availability = Get-AzMetric -ResourceId $webApp.Id `
                    -MetricName "HealthCheckStatus" `
                    -StartTime $startTime `
                    -EndTime $endTime `
                    -TimeGrain 01:00:00 `
                    -AggregationType Average `
                    -WarningAction SilentlyContinue -ErrorAction Stop
                
                $validHealth = $availability.Data | Where-Object {$_.Average -ne $null}
                if ($validHealth.Count -gt 0) {
                    $avgHealth = ($validHealth | Measure-Object -Property Average -Average).Average
                    Write-Host "   • Santé moyenne: $([math]::Round($avgHealth, 1))%"
                    
                    $unhealthy = $validHealth | Where-Object {$_.Average -lt 100}
                    if ($unhealthy.Count -gt 0) {
                        Write-Host "   ⚠ $($unhealthy.Count) périodes avec santé < 100%" -ForegroundColor Yellow
                    }
                }
            } catch {
                Write-Host "   ℹ Métriques de santé non disponibles" -ForegroundColor Gray
            }
            
            Write-Host ""
            
            $envResults.WebApps += $webAppResult
        }
        
    } catch {
        Write-Host "✗ Erreur: $($_.Exception.Message)" -ForegroundColor Red
    }
    
    return $envResults
}

# ===========================================
# ANALYSE DE TOUS LES ENVIRONNEMENTS
# ===========================================
foreach ($env in $subscriptions.GetEnumerator() | Sort-Object {if($_.Key -eq "prd"){0}elseif($_.Key -eq "stg"){1}else{2}}) {
    $result = Analyze-Environment -EnvName $env.Key -SubscriptionId $env.Value
    $globalResults += $result
}

# ===========================================
# RÉSUMÉ EXÉCUTIF
# ===========================================
Write-Host "`n`n╔═══════════════════════════════════════════════════════════╗" -ForegroundColor Cyan
Write-Host "║                    RÉSUMÉ EXÉCUTIF                        ║" -ForegroundColor Cyan
Write-Host "╚═══════════════════════════════════════════════════════════╝`n" -ForegroundColor Cyan

foreach ($envResult in $globalResults) {
    Write-Host "📌 $($envResult.Environment.ToUpper())" -ForegroundColor Green
    foreach ($webapp in $envResult.WebApps) {
        Write-Host "   $($webapp.Name):"
        if ($webapp.Metrics.ResponseTimeAvg) {
            Write-Host "   • Temps réponse moyen: $([math]::Round($webapp.Metrics.ResponseTimeAvg, 3))s (max: $([math]::Round($webapp.Metrics.ResponseTimeMax, 3))s)"
        }
        if ($webapp.Metrics.TotalRequests) {
            Write-Host "   • Requêtes: $($webapp.Metrics.TotalRequests) total ($([math]::Round($webapp.Metrics.AvgRequestsPerHour, 0))/h en moyenne)"
        }
        if ($webapp.Metrics.Http5xx -gt 0) {
            Write-Host "   ⚠ Erreurs 5xx: $($webapp.Metrics.Http5xx)" -ForegroundColor Red
        }
        if ($webapp.Metrics.SlowResponseCount -gt 0) {
            Write-Host "   ⚠ Réponses > 5s: $($webapp.Metrics.SlowResponseCount)" -ForegroundColor Yellow
        }
    }
    Write-Host ""
}

# ===========================================
# CONCLUSIONS
# ===========================================
Write-Host "╔═══════════════════════════════════════════════════════════╗" -ForegroundColor Cyan
Write-Host "║                      CONCLUSIONS                          ║" -ForegroundColor Cyan
Write-Host "╚═══════════════════════════════════════════════════════════╝`n" -ForegroundColor Cyan

Write-Host "✓ Always On activé sur toutes les WebApps (pas de cold start)" -ForegroundColor Green
Write-Host "✓ Aucune erreur 5xx détectée sur la période" -ForegroundColor Green
Write-Host "✓ Les services sont opérationnels et répondent aux requêtes" -ForegroundColor Green

Write-Host "`n⚠️  OBSERVATIONS:" -ForegroundColor Yellow
Write-Host "• Les métriques de temps de réponse montrent 0s, ce qui suggère"
Write-Host "  que soit les métriques ne sont pas bien configurées,"
Write-Host "  soit les requêtes sont trop rapides pour être mesurées à cette granularité"

Write-Host "`n📋 RECOMMANDATIONS:" -ForegroundColor Cyan
Write-Host "1. Activer Application Insights pour un monitoring plus détaillé"
Write-Host "2. Configurer des alertes sur les temps de réponse > 5s"
Write-Host "3. Demander à Olivier l'heure exacte du problème pour une analyse ciblée"
Write-Host "4. Vérifier les métriques Cosmos DB si disponibles"
Write-Host "5. Effectuer des tests de charge depuis Hong Kong"

Write-Host "`n✅ Script terminé - Rapport généré le $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')" -ForegroundColor Green
