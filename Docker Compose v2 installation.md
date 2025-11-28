## Working Install Method for Docker Compose v2 (for Amazon Linux 2023)

Run these steps:

1. Create plugin directory
```
sudo mkdir -p /usr/local/lib/docker/cli-plugins
```
2. Download docker compose v2
```
sudo curl -SL https://github.com/docker/compose/releases/download/v2.29.7/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
```
3. Make it executable
```
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
```
4. Verify
```
docker compose version
```

If you see a version output.
```
Docker Compose version v2.29.7
```
 you’re good
 
start build:
```
docker compose up --build -d
```
