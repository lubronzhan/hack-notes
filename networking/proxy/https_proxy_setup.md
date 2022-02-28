# How to use HTTPS Proxy for Kubernetes to work in Airgapped environment

## Desciption

Usually we use Http Proxy to route traffic for both HTTPS and HTTP request.

HTTP proxy gets a plain-text request and [in most but not all cases] sends a different HTTP request to the remote server, then returns information to the client.

HTTPS proxy is a relayer, which receives special HTTP request (CONNECT verb) and builds an opaque tunnel to the destination server (which is not necessarily even an HTTPS server). Then the client sends SSL/TLS request to the server and they continue with SSL handshake and then with HTTPS (if requested).

As you see, these are two completely different proxy types with different behavior and different design goals. HTTPS proxy can't cache anything as it doesn't see the request sent to the server. With HTTPS proxy you have a channel to the server and the client receives and validates server's certificate (and optionally vice versa). HTTP proxy, on the other hand, sees and has control over the request it received from the client.

While HTTPS request can be sent via HTTP proxy, this is almost never done because in this scenario the proxy will validate server's certificate, but the client will be able to receive and validate only proxy's certificate, and as name in the proxy's certificate will not match the address the socket connected to, in most cases an alert will be given and SSL handshake won't succeed (I am not going into details of how to try to address this).

## Steps for experiment

### Install MITMProxy

> MITM 7+ supports https as proxy protocol now: [Secure Web Proxy (TLS-over-TLS)](https://mitmproxy.org/posts/releases/mitmproxy7/)

URL for downloads: https://mitmproxy.org/downloads/#

Certficiate is generated at `~/.mitmproxy/mitmproxy-ca-cert.pem`

```sh
kubo@jumper:~$ cat ~/.mitmproxy/mitmproxy-ca-cert.pem
-----BEGIN CERTIFICATE-----
MIIDoTCCAomgAwIBAgIGDvZC9y6UMA0GCSqGSIb3DQEBCwUAMCgxEjAQBgNVBAMM
CW1pdG1wcm94eTESMBAGA1UECgwJbWl0bXByb3h5MB4XDTIyMDIxNTA4MDAyM1oX
DTI1MDIxNjA4MDAyM1owKDESMBAGA1UEAwwJbWl0bXByb3h5MRIwEAYDVQQKDAlt
aXRtcHJveHkwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQC7RlNFSP9e
8oHA1ICgbyuYeNKs+nhnp6kHz2lew4BPNV5SqpOMquK7qnQEseuvDmcZH52ggABL
0L1V2dK9Pi0SeNjO092R5GgreEB1q9gK+0gP/pTe53pf7jHMJRhsTTM+1fK4ceW8
hpB8iZ+D9obbYwH2p2gxhqqA0YR15XPVPtii1AHMi9mc27hR+9LYHqtcmVUqSNi9
g7Bwj7NmITZHkWlZ8cEJ3DGtnifswu6EJK6Vk6LngDI3sOOa/CzTSryPXDfL+zkD
SCYCm2PP30AeN2pyVWpiB/43BBlEOoCkmHERFa8N4VcpHi305ZVYGzlxx5m9/Nbv
51ERD2wKW7NLAgMBAAGjgdAwgc0wDwYDVR0TAQH/BAUwAwEB/zARBglghkgBhvhC
AQEEBAMCAgQweAYDVR0lBHEwbwYIKwYBBQUHAwEGCCsGAQUFBwMCBggrBgEFBQcD
BAYIKwYBBQUHAwgGCisGAQQBgjcCARUGCisGAQQBgjcCARYGCisGAQQBgjcKAwEG
CisGAQQBgjcKAwMGCisGAQQBgjcKAwQGCWCGSAGG+EIEATAOBgNVHQ8BAf8EBAMC
AQYwHQYDVR0OBBYEFHd08IOa82FCLqD6EHK52rlGipfGMA0GCSqGSIb3DQEBCwUA
A4IBAQCtX5MaCo4tdO0hFs4nLgnvG86UqfL1PXyPioiE9iK6dCasy3HOsJZwFqn6
sNKr8Vt5rW29/7u5KcKZ+r/l7+TBB7uQB5CNZoADn0P81bm2+FwIduKbUd+zbaE6
u6OZxpJupCq4H+RSXDPFDmTBQFCSyr6E159TMDlbouWnKu3YzDx7o68fUvS3rj6P
81DzEBtoeqCLOXx3sQHxm7ahxtzhZ1d/dwbT3laAo3ar7L1fNpooOKc0TEx8JLYq
X+2mH0fwoKWPwcpXw1Wtgu6ZgNCw67nX5f+RGQ9a3+MJUTHt4Brjl4GH8YLBV1ul
aX4fVirYzA0Yq55LqDnM5TsyHUAO
-----END CERTIFICATE-----
```

Starts the mitmproxy

```sh
## -p specify the port
mitmproxy -p 80
```

Test the proxy

```sh
kubo@jumper:~$ curl --proxy http://10.206.98.187:80 --cacert ~/.mitmproxy/mitmproxy-ca-cert.pem https://githubstatus.com
<html>
  <head>
    <meta http-equiv="refresh" content="0;url=https://www.githubstatus.com/">
  </head>
  <body></body>
</html>
```


### Trust the Cert in VM

***For Ubuntu***

* Go to /usr/local/share/ca-certificates/
* Create a new folder, i.e. `sudo mkdir example`
* Copy the .crt file into the `example` folder
* Make sure the permissions are OK (755 for the folder, 644 for the file)
* Run `sudo update-ca-certificates`

***For Photon***

* Convert the cert to PEM

```sh
openssl x509 -in cert -out mitmproxy.pem -outform PEM
```

* Put into this folder

```
cp mitmproxy.pem /etc/ssl/certs/
```

* Run `rehash_ca_certificates.sh`

### Make pod or container trust the cert

* Add proxy env to the pod

```yaml
        env:
        - name: "HTTP_PROXY"
          value: http://10.83.3.29:80
        - name: "HTTPS_PROXY"
          value: http://10.83.3.29:80
        - name: "NO_PROXY"
          value: ""
```

* Spin up a dev [pod](https://raw.githubusercontent.com/lubronzhan/k8s-networking-debug-tools/main/manifest/dnstools-daemonset.yaml):


* Ssh into the pod and try curl google, got error. This is because the pod also needs to trust the proxy cert

```sh
$ curl https://google.com -v

* Rebuilt URL to: https://google.com/
* Uses proxy env variable HTTPS_PROXY == 'http://10.83.3.29:80'
*   Trying 10.83.3.29...
...
curl: (60) SSL certificate problem: self signed certificate in certificate chain
More details here: https://curl.haxx.se/docs/sslcerts.html
curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

* You can either change the program to load the CA from a path, change dockerfile to read it.

"Or you can mount the pod through a configMap" <---- turns out only adding the cert to the `/etc/ssl/certs` doesn't work.
Because you also need to append the cert to the `ca-certificate.crt`

In this example, we mount the cert through configMap.

```sh
kubectl create configmap ca-pemstore --from-file=my-cert.pem
```

* Change pod spec

```yaml
        volumeMounts:
        - name: ca-pemstore
          mountPath: /etc/ssl/certs/my-cert.pem
          subPath: my-cert.pem
          readOnly: false
      volumes:
      - name: ca-pemstore
        configMap:
          name: ca-pemstore

```

* Ssh into pod and find the file `/etc/ssl/certs/my-cert.pem`

* curl still fail

```sh
dnstools# curl https://google.com
curl: (60) SSL certificate problem: self signed certificate in certificate chain
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.

```

* but if you change the `ca-certificate.crt` as well

```sh
cat /etc/ssl/certs/ca-cert-my-cert.pem.pem  >> /etc/ssl/certs/ca-certificates.crt
```

* Now it works
```sh
dnstools# curl https://google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="https://www.google.com/">here</A>.
</BODY></HTML>
```


## Related reading:

`update-ca-certificates` does more than just add a file, but also modifies ca-certificates.crt

https://www.unix.com/man-page/linux/8/update-ca-certificates/


>       update-ca-certificates  is  a  program that updates the directory /etc/ssl/certs to hold SSL certificates and generates certificates.crt, a
>
>       concatenated single-file list of certificates.
>
>       It reads the file /etc/ca-certificates.conf. Each line gives a pathname of a CA certificate under /usr/share/ca-certificates that should be
>       trusted.  Lines that begin with "#" are comment lines and thus ignored.	Lines that begin with "!" are deselected, causing the deactivation
>       of the CA certificate in question.
>
>       Furthermore all certificates found below /usr/local/share/ca-certificates are also included as implicitly trusted.
>
>       Before terminating, update-ca-certificates invokes run-parts on /etc/ca-certificates/update.d and calls each hook with a list  of  certifi-
>       cates: those added are prefixed with a +, those removed are prefixed with a -.
>
