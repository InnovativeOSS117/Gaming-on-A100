# How to Enable Gaming on NVIDIA GA100 GPU

This guide explains how to enable gaming support on an NVIDIA GA100 GPU. This enables WDDM suppport in windows for those GPU. 
It is based on the same method as on the P40 or P100 GPU.
Note that i have tested on my GPU, depending on GPU type it may not work.


## Step 1: Download and Install NVIDIA GRID Driver

1. Download the NVIDIA GRID driver from the official NVIDIA website or find a download link. I do not include it here and some others have explained it.
2. Run the installer and complete the installation process.

---

## Step 2: Modify Registry and Enable Gaming Mode

1. Open **windows regiester editor**.
2. Navigate to SYSTEM\CurrentControlSet\Control\Class\{4d36e968-e325-11ce-bfc1-08002be10318}
3. Open the folder corresponding to your GPU. 0001 for example.
3. Do the following changes:
   - `AdapterType` to `0` to treat the GPU as a gaming adapter.
   - Create `EnableMsHybrid` as Dword to `1` to disable the virtual display driver.

---

## Step 3: Enable the Changes

- Open **Device Manager** (`devmgmt.msc`).
- Find your GPU under **Display adapters**.
- Right-click the device and select **Disable device**, then right-click again and select **Enable device**.
- After this, the GPU should be ready for gaming. Check by using GPU-Z

---

## Troubleshooting

- If the GPU doesn’t appear correctly after these steps, try rebooting your computer.
- If your GPU’s device name differs, modify the device names list in the script accordingly.

---

## Important Notes

- Always backup your registry before making any changes.
- This method my put the GPU in a safe state. If so, put the register value or use DDU to remove the driver completely. After that it should be good again.

---

## PowerShell Script to Enable Gaming Mode

```powershell
# Requires running as Administrator

# Relaunch script as admin if not already running elevated
if (-NOT ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
    $arguments = "& '" + $myinvocation.mycommand.definition + "'"
    Start-Process powershell -Verb runAs -ArgumentList $arguments
    exit
}

# Device names to target for GA100 (Add or modify as needed)
$DevNames = @(
    "*NVIDIA GA100*",
    "*Tesla P40*",
    "*NVIDIA V100 OAM 32GB*",
    "*NVIDIA A100-SXM4-40GB*"
)

# Registry base path for Display Adapters
$RegBasePath = 'HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4d36e968-e325-11ce-bfc1-08002be10318}'

# Registry properties to set
$RegProps = @(
    @{ Name = 'AdapterType'; PropertyType = 'DWORD'; Value = 0x00 },         # Set adapter type to 0 (gaming)
    @{ Name = 'EnableMsHybrid'; PropertyType = 'DWORD'; Value = 0x01 }      # Disable virtual display
)

foreach ($DevName in $DevNames) {
    # Find matching devices in Device Manager
    $Devices = Get-PnpDevice -FriendlyName $DevName -ErrorAction SilentlyContinue
    if (-not $Devices) {
        Write-Host "No devices found matching: $DevName"
        continue
    }

    foreach ($Device in $Devices) {
        # Extract device instance key suffix
        $InstanceId = $Device.InstanceId
        $InstanceKey = ($InstanceId -split '\\')[-1]
        $RegPath = Join-Path $RegBasePath $InstanceKey

        # Set registry properties for the device
        foreach ($Prop in $RegProps) {
            try {
                New-ItemProperty -Path $RegPath -Name $Prop.Name -PropertyType $Prop.PropertyType -Value $Prop.Value -Force | Out-Null
                Write-Host "Set $($Prop.Name) to $($Prop.Value) on $DevName"
            } catch {
                Write-Warning "Failed to set $($Prop.Name): $($_.Exception.Message)"
            }
        }

        # Disable and enable the device to apply changes
        try {
            Disable-PnpDevice -InstanceId $InstanceId -Confirm:$false -ErrorAction Stop
            Start-Sleep -Seconds 2
            Enable-PnpDevice -InstanceId $InstanceId -Confirm:$false -ErrorAction Stop
            Write-Host "Device $DevName disabled and enabled successfully."
        } catch {
            Write-Warning "Failed to disable/enable device $DevName: $($_.Exception.Message)"
        }
    }
}

Write-Host "All done! Please reboot if changes are not yet effective."

