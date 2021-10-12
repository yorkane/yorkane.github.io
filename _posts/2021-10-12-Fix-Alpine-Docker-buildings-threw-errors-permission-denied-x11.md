# [Fix Alpine-Docker buildings threw errors permission denied](https://github.com/yorkane/yorkane.github.io/issues/11)

## Download the moby default seccomp profile and make modification
```sh
curl "https://raw.githubusercontent.com/moby/moby/master/profiles/seccomp/default.json" -o seccomp.json
sed -i "s/\"defaultAction\": .*,/\"defaultAction\": \"SCMP_ACT_TRACE\",/" seccomp.json
```
## Within docker-compose.yml:
```yaml
    security_opt:
      - seccomp:seccomp.json
```
## With Docker run command
```sh
docker run  --seccomp-profile=seccomp.json alpine:3.19 ...
```

### Source
> https://wiki.alpinelinux.org/wiki/Release_Notes_for_Alpine_3.14.0#faccessat2

1. runc v1.0.0-rc93
 * if using Docker's Debian repositories, this is part of containerd.io 1.4.3-2
 * if using Docker Desktop for Windows or Mac, this is part of Docker Desktop 3.3.0
 2. Docker 20.10.0 (which contains moby commit a181391) or greater, AND libseccomp 2.4.4 (which contains backported libseccomp commit 5696c89) or greater. In this case, to check if your host libseccomp is faccessat2-compatible, invoke scmp_sys_resolver faccessat2. If 439 is returned, faccessat2 is supported. If -1 is returned, faccessat2 is not supported. Note that if runc is older than v1.0.0-rc93, Docker must still be at least version 20.10.0, regardless of the result of this command.
3. As a workaround, in order to run under old Docker or libseccomp versions, the moby default seccomp profile should be downloaded and on line 2, defaultAction changed to SCMP_ACT_TRACE, then --seccomp-profile=default.json can be passed to dockerd, or --security-opt=seccomp=default.json passed to docker create or docker run. This will cause the system calls to return ENOSYS instead of EPERM, allowing the container to fall back to faccessat.