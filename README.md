# GetMachineInfo
#Powershell Get-Machineinfo Script

Function Get-MachineInfo{

Param(
[string]$MachineName  = (throw "Name required"),
[Switch]$gm = $false,
[Switch]$gmc = $false,
[Switch]$gml = $false
)

ipmo PSExcel
$Content = GetAllForest

$Forest = {$Content}.Invoke()

$Forest.Remove("nam06")  | Out-Null
$Forest.Remove("FSProd") | Out-Null 
$Forest.Remove("Sonar")  | Out-Null
$Forest.Remove("GLS01")  | Out-Null 
$Forest.Remove("GLS06")  | Out-Null

$Payload


if($MachineName -like '*FFO11*'){
$i = 'FSPROD'
$Payload = Get-CentralAdminDropBoxPayload -Filter "Name -like '*EopStateless*$i*'" | Select Name, PayloadStatus, ID, StartTime
$Payload=(($Payload | Sort StartTime -Descending)[0]).Name
Write-Host ""
Write-Host "$Payload is Running in FSPROD"
}
elseif($MachineName -like '*TP*'){
$i = 'Sonar'
$Payload = Get-CentralAdminDropBoxPayload -Filter "Name -like '*EopStateless*$i*'" | Select Name, PayloadStatus, ID, StartTime
$Payload=(($Payload | Sort StartTime -Descending)[0]).Name
Write-Host ""
Write-Host "$Payload is Running in Sonar."
Write-Host ""
Return
}
elseif($MachineName -like '*gls*'){
$i = 'GLS'
$Payload = Get-CentralAdminDropBoxPayload -Filter "Name -like '*EopStateless*$i*'" | Select Name, PayloadStatus, ID, StartTime
$Payload=(($Payload | Sort StartTime -Descending)[0]).Name
Write-Host ""
Write-Host "$Payload is Running in GLS."
Write-Host ""
}
elseif($MachineName -like '*NAM06*'){
$i = 'SDF'
$Payload = Get-CentralAdminDropBoxPayload -Filter "Name -like '*EopSDF-NAM06-*'" | Select Name, PayloadStatus, ID, StartTime
$Payload=(($Payload | Sort StartTime -Descending)[0]).Name
Write-Host ""
Write-Host "$Payload is Running in NAM06"
}
elseif($MachineName -like '*FFO30*'){
Write-Host ""
Write-Host "Setting ServiceInstace to Gallatin" -ForegroundColor DarkGray
Set-MyServiceInstance Gallatin
$Payload = Get-CentralAdminDropBoxPayload -Filter "Name -like '*EopStateless-FFO30*'" | Select Name, PayloadStatus, ID, StartTime
$Payload=(($Payload | Sort StartTime -Descending)[0]).Name
Write-Host ""
Write-Host "$Payload is Running in Gallatin"
}
else{
foreach($i in $Forest){
if($MachineName -like "*$($i)*"){
$i = $i.ToUpper()
$Payload = Get-CentralAdminDropBoxPayload -Filter "Name -like '*EopStateless*$i*'" | Select Name, PayloadStatus, ID, StartTime
$Payload=(($Payload | Sort StartTime -Descending)[0]).Name
Write-Host ""
Write-Host "$Payload is Running in $i"
}
}
}

Write-Host ""

Write-Host "Machine Info:" -ForegroundColor Blue

$ProvisioningState = (gcam $MachineName).ProvisioningState

$RedundancyGroup = (gcam $MachineName).RedundancyGroup

$AV = (gcam $MachineName).ActualVersion

$DM = (gcam $MachineName).DeploymentMode

$MS = (gcam $MachineName).MaintenanceState

if($ProvisioningState -eq 'Provisioned'){
Write-Host ""
Write-Host "Machine is Provisioned with $($AV)"

$CA=$MachineName.ToCharArray()
$CA1=$MachineName.ToCharArray()[3..$CA.Count]
$GetPMString=[String]::new($CA1)


$PairedMachines = (gcam *$($GetPMString)).Name
$Count = $PairedMachines.Count
Write-Host ""

Write-Host "DM: $DM`nMS: $MS"

Write-Host ""

Write-Host "Machine has $($Count) Paired Machines. Getting Status of Paired Machine.." -ForegroundColor DarkGray
gcam *$GetPMString  | Sort ProvisioningState
Write-Host ""

Write-Host "Payload History on this Machine:"

Get-CentralAdminDropBoxMachineEntry -Filter "Machine -eq '$MachineName'" |select -property id, machine, payloadstatus, payload, LastModifiedTime,  WorkflowId |sort LastModifiedTime -desc| ft -a

}
else{


Write-Host ""
Write-Host "Machine is not Provisioned" -ForegroundColor Red
Write-Host ""

Write-Host "DM: $DM`nMS: $MS"

$CA=$MachineName.ToCharArray()
$CA1=$MachineName.ToCharArray()[3..$CA.Count]
$GetPMString=[String]::new($CA1)


$PairedMachines = (gcam *$($GetPMString)).Name
$Count = $PairedMachines.Count
Write-Host ""
Write-Host "Machine has $($Count) Paired Machines. Getting Status of Paired Machine.." -ForegroundColor DarkGray
gcam *$GetPMString  | Sort ProvisioningState

Write-Host ""

Write-Host "Payload History on this Machine:"


Get-CentralAdminDropBoxMachineEntry -Filter "Machine -eq '$MachineName'" |select -property id, machine, payloadstatus, payload, LastModifiedTime,  WorkflowId |sort LastModifiedTime -desc| ft -a

}


Write-Host "POD Info:" -ForegroundColor Blue
Write-Host ""

$CA=$MachineName.ToCharArray()
$CA1=$MachineName.ToCharArray()[0..($CA.Count-4)]
$GetPMString=[String]::new($CA1)

$CA2 = $MachineName.ToCharArray()[0..5]
$TempString = [String]::new($CA2)


Write-Host "RedundancyGroup = $($RedundancyGroup)" -ForegroundColor Yellow

Write-Host ""

Write-Host "Details:"

$PodTemp=Get-CentralAdminDropBoxPodEntry -Filter "Payload -like '$Payload' -and Scope -NotLike '*E2EComplete' -and Scope -NotLike '*EopDataInsights' -and Scope -NotLike '*-Template' -and $_.OrchestrationUnit -eq '$RedundancyGroup' -and $_.POD -like '$TempString*'" | Select Id, Pod, Scope, @{n="OU";e={$_.OrchestrationUnitName}}, PayloadStatus, RetryCount, @{n="Green";e={$_.SucceededMachines}}, @{n="Yellow";e={$_.MachinesInProgress}}, @{n="Red";e={$_.FailedMachines}},@{n="Total";e={$_.ApproxTotalMachineCount}},WorkflowId, LastModifiedTime, AdditionalStatusInfo

$PodTemp | ft

$PODID = (Get-CentralAdminDropBoxPodEntry -Filter "Payload -like '$Payload' -and Scope -NotLike '*E2EComplete' -and Scope -NotLike '*EopDataInsights' -and Scope -NotLike '*-Template' -and $_.OrchestrationUnit -eq '$RedundancyGroup' -and $_.POD -like '$TempString*'").ID
if($PODID.Count -eq 0){
Write-Host ""
Write-Host "Deployment on this Machine or on the POD might have not Started yet, Please check.." -ForegroundColor Red
Write-Host ""

##

if($gm){

Write-Host "Running .\gm.ps1.." -ForegroundColor Blue
Write-Host ""
.\gm.ps1 $MachineName
}
if($gmc){
Write-Host "Running .\gm.ps1 with CheckHealthy.." -ForegroundColor Blue
Write-Host ""
.\gm.ps1 $MachineName -CheckHealthy
}
if($gml){
Write-Host "Running .\gm.ps1 and fetching Machine logs.. " -ForegroundColor Blue
Write-Host ""
.\gm.ps1 $MachineName

########
if($Machine -like '*nam06*'){$Payload=(Get-CentralAdminDropBoxMachineEntry -Filter "Machine -eq '$MachineName' -And Payload -like '*EopSDF*' " |select -property id, machine, payloadstatus, payload, LastModifiedTime,  WorkflowId |sort LastModifiedTime -desc)
 }
 else{
 $Payload=(Get-CentralAdminDropBoxMachineEntry -Filter "Machine -eq '$MachineName' -And Payload -like '*EopStateless*' " |select -property id, machine, payloadstatus, payload, LastModifiedTime,  WorkflowId |sort LastModifiedTime -desc)
 }
 if($Payload.Count -eq 0){
 Write-Host "No Recent Stateless Payload Found." -ForegroundColor Red
 Write-Host ""
 }else{
 $PL = $Payload[0].Payload
 $PS = $Payload[0].PayloadStatus
 $WF = $Payload[0].WorkFlowID
 Write-Host "The Most recent Payload on this Machine: $PL and it is $PS. Its Workflow is $WF" -ForegroundColor Yellow
 Write-Host ""
 Write-Host "Checking its Workflow.." -ForegroundColor Blue
 Write-Host ""
 Write-Host "gorc Results:" -ForegroundColor Yellow
 Write-Host ""
 gorc $WF
 ot $WF -ShowAll | Clip
 Write-Host ""
 Write-Host "Complete Log has been copied to clipboard" -ForegroundColor Yellow
 Write-Host ""
 Write-Host "Error: " -ForegroundColor Yellow
 Write-Host ""
 ot $WF -ErrorsOnly
 Write-Host ""

}

}

Set-MyServiceInstance MultiTenant
##

Return
}
elseif($PODID.Count -eq 1){

[Array]$OUMachines=@()
[Array]$PodMachines=@()

$OUMachines=gcam -ShowAll -Filter "Redundancygroup -like '$RedundancyGroup' -and ActivityState -eq 'DotBuildUpgrade'" -ErrorAction SilentlyContinue |select Name, @{n="Version";e={$_.ActualVersion.SubString(0,14)}}, ActualMachineDefinition, ActivityState, @{n="PS";e={$_.ProvisioningState}}, @{n="MM";e={$_.MaintenanceState}}, @{n="DM";e={$_.DeploymentMode}}
$PodMachines=$OUMachines | Where {$_.Name -like "$TempString*"}

[Array]$PayloadMachines=@()
[Array]$PayloadMachines=Get-CentralAdminDropBoxMachineEntry -Filter "PodEntryId -eq '$PODID'" | Select Id, Machine, PayloadStatus, @{n="Ret";e={$_.RetryCount}}, WorkflowId, StartTime, LastModifiedTime 
Write-Host "Machines in this Pod:" -ForegroundColor Yellow
$JoinedMachines=@()
If(!$PayloadMachines){
$JoinedMachines=$PodMachines    
}
else{
$JoinedMachines=Join-Object -Left $PodMachines -Right $PayloadMachines -LeftJoinProperty Name -RightJoinProperty Machine -Type AllInLeft
}

$JoinedMachines |Sort Name |ft ID, Name, PayloadStatus,Version,ActivityState, PS, MM, DM,Ret,WorkflowId

}
else{

Write-Host "$($PODID.Count) PODS are present under Ochestration Unit $($RedundancyGroup)"
Write-Host ""
foreach($i in $PODID){

[Array]$OUMachines=@()
[Array]$PodMachines=@()

$OUMachines=gcam -ShowAll -Filter "Redundancygroup -like '$RedundancyGroup' -and ActivityState -eq 'DotBuildUpgrade'" -ErrorAction SilentlyContinue |select Name, @{n="Version";e={$_.ActualVersion.SubString(0,14)}}, ActualMachineDefinition, ActivityState, @{n="PS";e={$_.ProvisioningState}}, @{n="MM";e={$_.MaintenanceState}}, @{n="DM";e={$_.DeploymentMode}}
$PodMachines=$OUMachines | Where {$_.Name -like "$TempString*"}

[Array]$PayloadMachines=@()
[Array]$PayloadMachines=Get-CentralAdminDropBoxMachineEntry -Filter "PodEntryId -eq '$i'" | Select Id, Machine, PayloadStatus, @{n="Ret";e={$_.RetryCount}}, WorkflowId, StartTime, LastModifiedTime 
Write-Host "Machines in the Pod: $i" -ForegroundColor Yellow
$JoinedMachines=@()
If(!$PayloadMachines){
$JoinedMachines=$PodMachines    
}
else{
$JoinedMachines=Join-Object -Left $PodMachines -Right $PayloadMachines -LeftJoinProperty Name -RightJoinProperty Machine -Type AllInLeft
}

$JoinedMachines |Sort Name |ft ID, Name, PayloadStatus,Version,ActivityState, PS, MM, DM,Ret,WorkflowId

}
}

Write-Host "Scope Info:" -ForegroundColor Blue
Write-Host ""

$Scope = (Get-CentralAdminDropBoxPodEntry -Filter "Payload -like '$Payload' -and $_.OrchestrationUnit -eq '$RedundancyGroup' -and $_.POD -like '$TempString*'").Scope

if($Scope.Count -eq 1){
 Get-CentralAdminDropBoxScopeEntry -Filter "Payload -like '$Payload' -and Scope -like '$Scope' " | Select Id, Scope, PayloadStatus, ApprovalStatus ,CreateTime, LastModifiedTime, Info | ft
}
else{
foreach($i in $Scope){
Get-CentralAdminDropBoxScopeEntry -Filter "Payload -like '$Payload' -and Scope -like '$i' " | Select Id, Scope, PayloadStatus, ApprovalStatus ,CreateTime, LastModifiedTime, Info | ft
}
}

Write-Host "Summary of Machine: $MachineName" -ForegroundColor Blue
Write-Host ""

$POD= (Get-CentralAdminDropBoxPodEntry -Filter "Payload -like '$Payload' -and Scope -NotLike '*E2EComplete' -and Scope -NotLike '*EopDataInsights' -and Scope -NotLike '*-Template' -and $_.OrchestrationUnit -eq '$RedundancyGroup' -and $_.POD -like '$TempString*'").POD
if($POD.Count -eq 1){
$Scope = (Get-CentralAdminDropBoxPodEntry -Filter "Payload -like '$Payload' -and Scope -NotLike '*E2EComplete' -and Scope -NotLike '*EopDataInsights' -and Scope -NotLike '*-Template' -and $_.OrchestrationUnit -eq '$RedundancyGroup' -and $_.POD -like '$TempString*'").Scope
Write-Host "$($MachineName) is present in $($POD) POD which is in turn present in $($Scope) Scope" -ForegroundColor Yellow
Write-Host ""
}
else{
$Scope = (Get-CentralAdminDropBoxPodEntry -Filter "Payload -like '$Payload' -and Scope -NotLike '*E2EComplete' -and Scope -NotLike '*EopDataInsights' -and Scope -NotLike '*-Template' -and $_.OrchestrationUnit -eq '$RedundancyGroup' -and $_.POD -like '$TempString*'").Scope
Write-Host "$($MachineName) is present in $($POD[0]) POD which is in turn present in both $($Scope) Scopes" -ForegroundColor Yellow
Write-Host ""
}


if($gm){

Write-Host "Running .\gm.ps1.." -ForegroundColor Blue
Write-Host ""
.\gm.ps1 $MachineName
}
if($gmc){
Write-Host "Running .\gm.ps1 with CheckHealthy.." -ForegroundColor Blue
Write-Host ""
.\gm.ps1 $MachineName -CheckHealthy
}
if($gml){
Write-Host "Running .\gm.ps1 and fetching Machine logs.. " -ForegroundColor Blue
Write-Host ""
.\gm.ps1 $MachineName

########
if($Machine -like '*nam06*'){$Payload=(Get-CentralAdminDropBoxMachineEntry -Filter "Machine -eq '$MachineName' -And Payload -like '*EopSDF*' " |select -property id, machine, payloadstatus, payload, LastModifiedTime,  WorkflowId |sort LastModifiedTime -desc)
 }
 else{
 $Payload=(Get-CentralAdminDropBoxMachineEntry -Filter "Machine -eq '$MachineName' -And Payload -like '*EopStateless*' " |select -property id, machine, payloadstatus, payload, LastModifiedTime,  WorkflowId |sort LastModifiedTime -desc)
 }
 if($Payload.Count -eq 0){
 Write-Host "No Recent Stateless Payload Found." -ForegroundColor Red
 Write-Host ""
 }else{
 $PL = $Payload[0].Payload
 $PS = $Payload[0].PayloadStatus
 $WF = $Payload[0].WorkFlowID
 Write-Host "The Most recent Payload on this Machine: $PL and it is $PS. Its Workflow is $WF" -ForegroundColor Yellow
 Write-Host ""
 Write-Host "Checking its Workflow.." -ForegroundColor Blue
 Write-Host ""
 Write-Host "gorc Results:" -ForegroundColor Yellow
 Write-Host ""
 gorc $WF
 ot $WF -ShowAll | Clip
 Write-Host ""
 Write-Host "Complete Log has been copied to clipboard" -ForegroundColor Yellow
 Write-Host ""
 Write-Host "Error: " -ForegroundColor Yellow
 Write-Host ""
 ot $WF -ErrorsOnly
 Write-Host ""

}

}

Set-MyServiceInstance MultiTenant
}

