
```bash
git clone https://gerrit.o-ran-sc.org/r/sim/e2-interface
```

```bash
nano Dockerfile_kpm
```
Uncomment the SLEEP command.

```bash
sudo docker build -f Dockerfile_kpm -t oransim:0.0.999 .
```

```bash
sudo docker run -d --name oransim -it oransim:0.0.999
```

```bash
sudo docker exec -it oransim /bin/bash
```

Get the IP address of e2term-alpha pod

kpm_sim \<e2term IP> \<port>