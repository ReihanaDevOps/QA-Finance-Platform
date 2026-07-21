docker context inspect desktop-linux --format "{{.Endpoints.docker.Host}}"; [System.IO.Directory]::GetFiles("\\.\pipe\") | Select-String -Pattern "docker"
