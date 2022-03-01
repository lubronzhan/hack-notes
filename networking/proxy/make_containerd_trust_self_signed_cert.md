# Make containerd trust self signed cert

## Description

Sometimes we need containerd to trust certificate, for example for private registry, or for proxy.

To configure containerd with proxy with cert, need two changes

1. Update containerd system conf with proxy environment variable, reload and restart containerd
2. Update containerd `/etc/containerd/config.toml` with certificate, restart containerd

https://github.com/kubernetes-sigs/kind/issues/688#issuecomment-508782438


## Example

* Change the containerd proxy settings

```sh
PROXY=http://10.83.3.29:80
cat <<EOF >/etc/systemd/system/containerd.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=${PROXY:-}"
Environment="HTTPS_PROXY=${PROXY:-}"
Environment="NO_PROXY="
EOF

systemctl daemon-reload
systemctl restart containerd
```

* Currently there is no certificate configured for containerd to trust.

Test by pulling an image:

```sh
root [ /home/capv ]# crictl --debug pull nginx
DEBU[0000] get image connection
DEBU[0000] connect using endpoint 'unix:///var/run/containerd/containerd.sock' with '2s' timeout
DEBU[0000] connected successfully using endpoint: unix:///var/run/containerd/containerd.sock
DEBU[0000] PullImageRequest: &PullImageRequest{Image:&ImageSpec{Image:nginx,Annotations:map[string]string{},},Auth:nil,SandboxConfig:nil,}
DEBU[0000] PullImageResponse: nil
FATA[0000] pulling image: rpc error: code = Unknown desc = failed to pull and unpack image "docker.io/library/nginx:latest": failed to resolve reference "docker.io/library/nginx:latest": failed to do request: Head "https://registry-1.docker.io/v2/library/nginx/manifests/latest": proxyconnect tcp: x509: certificate signed by unknown authority
```

* So the containerd needs to trust the cert.

* Add the cert to containerd config

```sh
## proxy cert is in /home/capv/mitm-ca-cert.pem
cat <<EOF >/etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".registry.configs."registry-1.docker.io".tls]
  ca = /home/capv/mitm-ca-cert.pem
EOF

systemctl restart containerd
```

* Pull again

```sh
root [ /home/capv ]# crictl --debug pull nginx
DEBU[0000] get image connection
DEBU[0000] connect using endpoint 'unix:///var/run/containerd/containerd.sock' with '2s' timeout
DEBU[0000] connected successfully using endpoint: unix:///var/run/containerd/containerd.sock
DEBU[0000] PullImageRequest: &PullImageRequest{Image:&ImageSpec{Image:nginx,Annotations:map[string]string{},},Auth:nil,SandboxConfig:nil,}
DEBU[0001] PullImageResponse: &PullImageResponse{ImageRef:sha256:c316d5a335a5cf324b0dc83b3da82d7608724769f6454f6d9a621f3ec2534a5a,}
Image is up to date for sha256:c316d5a335a5cf324b0dc83b3da82d7608724769f6454f6d9a621f3ec2534a5a
```

## In addition

For container 1.5 [(release-note)](https://github.com/containerd/containerd/releases/tag/v1.5.0), update under the directory of host configuration ***doesn't require restarting the containerd daemon***

But still...

Inserting below config to `/etc/containerd/config.toml` still requires restarting containerd daemon

```sh
[plugins."io.containerd.grpc.v1.cri".registry]
   config_path = "/etc/containerd/certs.d"
```

Then later on, create folder like

```sh
mkdir -p /etc/containerd/certs.d/docker.io/
touch /etc/containerd/certs.d/docker.io/hosts.toml

cat <<EOF >/etc/containerd/certs.d/docker.io/hosts.toml
server = "https://docker.io"

[host."https://registry-1.docker.io"]
  capabilities = ["pull", "resolve"]
  ca = "/home/capv/mitm-ca-cert.pem"
EOF
```

Then you don't have to restart the containerd daemon
