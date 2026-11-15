1. Menjalankan cAdvisor dengan Docker Run
Jalankan perintah berikut di server Ubuntu:
```
docker run -d \
  --name=cadvisor \
  --restart=always \
  -p 8080:8080 \
  --privileged \
  -v /:/rootfs:ro \
  -v /var/run:/var/run:ro \
  -v /sys:/sys:ro \
  -v /var/lib/docker:/var/lib/docker:ro \
  -v /dev/disk:/dev/disk:ro \
  gcr.io/cadvisor/cadvisor:latest
```
cek container 
```
docker ps
```

  
