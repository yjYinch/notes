<font color=red size=4>***Docker***</font>

```bash
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/json: dial unix /var/run/docker.sock: connect: permission denied
```

解决：

```bash
sudo chmod a+rw /var/run/docker.sock
```

