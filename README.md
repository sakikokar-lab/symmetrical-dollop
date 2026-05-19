name: RDP with NetBird

on:
  workflow_dispatch:

jobs:
  secure-rdp:
    runs-on: windows-latest
    timeout-minutes: 3600

    steps:
      - name: Install NetBird
        run: |
          # NetBird MSI download URL (make sure this is the latest version)
          $nbUrl = "https://github.com/netbirdio/netbird/releases/download/v0.65.1/netbird_installer_0.65.1_windows_amd64.msi"
          $installerPath = "$env:TEMP\netbird.msi"
          
          # Download the NetBird installer using curl
          Write-Host "Downloading NetBird installer from $nbUrl"
          curl -L $nbUrl -o $installerPath
          
          # Install NetBird silently
          Write-Host "Installing NetBird..."
          Start-Process msiexec.exe -ArgumentList "/i", "$installerPath", "/quiet", "/norestart" -Wait
          
          # Clean up the installer file
          Remove-Item $installerPath -Force
          Write-Host "NetBird installation complete."

      - name: Bring up NetBird with the provided setup key
        run: |
          # Path to NetBird executable
          $netbird = "$env:ProgramFiles\NetBird\netbird.exe"

          # Ensure that the NetBird executable exists
          if (-not (Test-Path $netbird)) {
              Write-Error "NetBird executable not found!"
              exit 1
          }

          # Ensure setup key is provided (no longer using --authkey flag)
          if (-not "${{ secrets.NETBIRD_SETUP_KEY }}") {
              Write-Error "NETBIRD_SETUP_KEY secret is missing!"
              exit 1
          }

          # Bring up the NetBird connection (use correct flags based on CLI)
          Write-Host "Connecting to NetBird..."
          & $netbird up --setup-key "${{ secrets.NETBIRD_SETUP_KEY }}" --hostname="gh-runner-$env:GITHUB_RUN_ID"
          
          # Wait for NetBird to assign an IP address (check if it’s assigned correctly)
          Write-Host "Waiting for NetBird to assign IP..."
          $nbIP = $null
          $retries = 0

          while (-not $nbIP -and $retries -lt 10) {
              Start-Sleep -Seconds 5
              $statusOutput = & $netbird status
              $ipMatch = $statusOutput | Select-String -Pattern '\b\d{1,3}(\.\d{1,3}){3}\b'

              if ($ipMatch) {
                  $nbIP = $ipMatch.Matches[0].Value
              }

              $retries++
          }

          if (-not $nbIP) {
              Write-Error "NetBird IP not assigned after retries."
              exit 1
          }

          echo "NETBIRD_IP=$nbIP" >> $env:GITHUB_ENV
          Write-Host "Detected NetBird IP: $nbIP"

      - name: Create RDP User and Password
        run: |
          # Generate a random secure password
          Add-Type -AssemblyName System.Security
          $charSet = @(
              [char[]](65..90),    # Uppercase letters
              [char[]](97..122),   # Lowercase letters
              [char[]](48..57),    # Numbers
              [char[]](33..47),    # Special characters
              [char[]](58..64),
              [char[]](91..96),
              [char[]](123..126)
          )

          $passwordChars = @()
          $passwordChars += $charSet[0] | Get-Random -Count 4   # Uppercase
          $passwordChars += $charSet[1] | Get-Random -Count 4   # Lowercase
          $passwordChars += $charSet[2] | Get-Random -Count 4   # Numbers
          $passwordChars += $charSet[3] | Get-Random -Count 4   # Special chars
          $password = -join ($passwordChars | Sort-Object { Get-Random })
