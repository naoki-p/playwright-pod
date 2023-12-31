---
podmanでGUIアプリを動かすまでの軌跡
---

## Windows版Podmanのインストール
* exeからインストール
* podman machine init
* podman machine start

---
## ここからはWSL:podman-machine-defaultで
---

## chromeを動かす
```
$ vi Containerfile
$ podman build -t test:v1 .
$ vi seccomp_profile.json
$ podman run -it --rm -e DISPLAY=${DISPLAY} -v /tmp/.X11-unix:/tmp/.X11-unix --ipc=host --user pwuser --security-opt seccomp=seccomp_profile.json localhost/test:v1 google-chrome
```

* Containerfile
```
FROM mcr.microsoft.com/playwright:jammy
RUN apt-get update && apt-get install -y ca-certificates curl git ssh \
 && curl -fsSL https://dl-ssl.google.com/linux/linux_signing_key.pub | gpg --dearmour -o /usr/share/keyrings/google-keyring.gpg \
 && echo "deb [arch=amd64 signed-by=/usr/share/keyrings/google-keyring.gpg] http://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google-chrome.list \
 && apt-get update -y && apt-get install -y --no-install-recommends google-chrome-stable \
 && rm -rf /var/lib/apt/lists/*
```

* seccomp_profile.json
```
{
  "comment": "Allow create user namespaces",
  "names": [
    "clone",
    "setns",
    "unshare"
  ],
  "action": "SCMP_ACT_ALLOW",
  "args": [],
  "includes": {},
  "excludes": {}
}
```

## TODO
* WSL:UbuntuからPodmanをコントロールしたい

## 参考文献
* PodmanでCUIアプリ
> https://qiita.com/karosuwindam/items/7aa0e55168d91cfcfbfe
* DockerコンテナでPlaywright
> https://playwright.dev/docs/docker


---
## Podを生成してみる
---

```
$ podman pod create pw-pod
$ podman container run --pod=pw-pod -it -e DISPLAY=${DISPLAY} -v /tmp/.X11-unix:/tmp/.X11-unix --ipc=host --user pwuser --security-opt seccomp=seccomp_profile.json localhost/test:v1 google-chrome
$ podman generate kube pw-pod
```

```
# Save the output of this file and use kubectl create -f to import
# it into Kubernetes.
#
# Created with podman-4.7.2
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2023-11-29T10:37:17Z"
  labels:
    app: pw-pod
  name: pw-pod
spec:
  containers:
  - command:
    - google-chrome
    env:
    - name: TERM
      value: xterm
    - name: DISPLAY
      value: :0
    image: localhost/test:v1
    name: vibrantsinoussi
    securityContext:
      runAsGroup: 1000
      runAsUser: 1000
    stdin: true
    tty: true
    volumeMounts:
    - mountPath: /tmp/.X11-unix
      name: tmp-.X11-unix-host-0
  volumes:
  - hostPath:
      path: /tmp/.X11-unix
      type: Directory
    name: tmp-.X11-unix-host-0
```

## Containerfileはイメージ名のディレクトリの下に配置
```
$ mkdir test
$ mv Containerfile ./test/
$ podman pod rm -f pw-pod
$ podman system prune --volumes
```

## 起動
```
$ podman play kube pw-pod.yml
```

## WSLの別のディストリビューションから繋ぐ
> https://zenn.dev/dozo/articles/39d90d1e47f7ae
* Ubuntuのdevパッケージからpodmanをインストール
* /mnt/wsl/に生えてるpodman-socketに繋ぐ
```
export CONTAINER_HOST=unix:///mnt/wsl/podman-sockets/podman-machine-default/podman-user.sock
```
* podmanの-r(--remote)オプション必須

## generateで抜けたsecoptとか
* Kubernetes と seccomp を組み合わせてシステムコールを制限する
> https://kakakakakku.hatenablog.com/entry/2021/07/05/095346
* 公式仕様書
> https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#pod-v1-core

```
    hostIPC: true
    securityContext:
      ...
      seccompProfile:
        type: Localhost
        localhostProfile: playwright/seccomp_profile.json
```
