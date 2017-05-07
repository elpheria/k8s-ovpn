# Kubernetes OpenVPN
OpenVPN on a Kubernetes cluster. This implementation of OpenVPN simply let's you create your own secure VPN service on a cluster running on some cloud provider (this is tested on Google Cloud Platform). Other kuberized OpenVPN solutions right now aim to provide direct access to services inside the clister itself, but *this is not the aim* of **k8s-openvpn**.

**k8s-openvpn** relies on [excellent existing Docker implementations](https://github.com/kylemanna/docker-openvpn) of OpenVP and turns it into a reliable, scalable, and easy-to-deploy Kubernetes Deploymemt. It runs in a separate Namespace to isolate it from the rest of the cluster, and uses Secrets and ConfigMaps instead of Persistent Volumes to store configuration and PKI.

With **Kubernetes OpenVPN** you can roll your own secure VPN service with the ability to easily deploy multiple configurations and authorise friends and family too!

# Prerequisites

* You need decent Kubernetes skills if you want to understand what you're doing. Fortunately the [docs](https://kubernetes.io/docs/home/) are excellent.
* These instructions utilise Google Cloud Platform so [deploy your cluster there](https://kubernetes.io/docs/getting-started-guides/gce/) if you want to follow along verbatim.
* You should familiarise yourself with the documentation of the [OpenVPN container](https://github.com/chepurko/docker-openvpn/tree/master/docs) itself. Here we're primarily concerned with creating a working Kubernetes OpenVPN setup. The are many more features and configurations to OpenVPN, though.

# Installation

* Initialise the configuration files and ECC certificates
  * **Run these commands on your workstation.** You are creating a directory with OpenVPN configuration and sensitive PKI files. [Docker](https://docs.docker.com/engine/installation/) is required.

```bash
# docker
$ mkdir ovpn0 && cd ovpn0
# Modify the crypto algos to your liking and see documentation here
# https://github.com/kylemanna/docker-openvpn/blob/master/docs/paranoid.md
$ docker run --net=none --rm -it -v $PWD:/etc/openvpn chepurko/docker-openvpn ovpn_genconfig \
    -u udp://VPN.SERVERNAME.COM \
    -C 'AES-256-GCM' -a 'SHA384' -T 'TLS-ECDHE-ECDSA-WITH-AES-256-GCM-SHA384' \
    -b -n 185.121.177.177 -n 185.121.177.53 -n 87.98.175.85
$ docker run -e EASYRSA_ALGO=ec -e EASYRSA_CURVE=secp384r1 \
    --net=none --rm -it -v $PWD:/etc/openvpn chepurko/docker-openvpn ovpn_initpki
$ docker run --net=none --rm -it -v $PWD:/etc/openvpn chepurko/docker-openvpn ovpn_copy_server_files
```

* Generate client ECC certificate and retrieve client configuration with embedded certificates

```bash
$ export CLIENTNAME="your_client_name"
$ docker run -e EASYRSA_ALGO=ec -e EASYRSA_CURVE=secp384r1 \
    --net=none --rm -it -v $PWD:/etc/openvpn chepurko/docker-openvpn easyrsa build-client-full $CLIENTNAME
$ docker run --net=none --rm -v $PWD:/etc/openvpn chepurko/docker-openvpn ovpn_getclient $CLIENTNAME > $CLIENTNAME.ovpn
```

* Or generate client RSA certificates if your client doesn't support ECC

```bash
$ export CLIENTNAME="your_client_name"
$ docker run --net=none --rm -it -v $PWD:/etc/openvpn chepurko/docker-openvpn easyrsa build-client-full $CLIENTNAME
$ docker run --net=none --rm -v $PWD:/etc/openvpn chepurko/docker-openvpn ovpn_getclient $CLIENTNAME > $CLIENTNAME.ovpn
```

* Prepare the namespace and some file permissions.

```bash
$ kubectl apply -f ../00-namespace.yaml
$ kubectl config set-context $(kubectl config current-context) --namespace=ovpn
# Validate it
$ kubectl config view | grep namespace:
$ sudo chown -R $USER:$USER server/*
```

* Create ConfigMaps and Secrets.

```bash
$ kubectl create secret generic ovpn0-key --from-file=server/pki/private/VPN.SERVERNAME.COM.key
$ kubectl create secret generic ovpn0-cert --from-file=server/pki/issued/VPN.SERVERNAME.COM.crt
$ kubectl create secret generic ovpn0-pki \
    --from-file=server/pki/ca.crt --from-file=server/pki/dh.pem --from-file=server/pki/ta.key
$ kubectl create configmap ovpn0-conf --from-file=server/
$ kubectl create configmap ccd0 --from-file=server/ccd

```

* Bring up the OpenVPN server in your Kubernetes cluster.

```bash
$ kubectl apply -f ../ovpn-Service.yaml
$ kubectl apply -f ../ovpn-Deployment.yaml
```

# Usage

* See [USAGE.md](USAGE.md).

# TODO
- [X] Fix "Options error: Unrecognized option or missing or extra parameter(s) in /etc/openvpn/openvpn.conf:30: push (2.4.1)" - due to missing quotes in openvpn.conf
- [ ] Enable the `--tls-crypt` option in [`ovpn_genconfig`](https://github.com/chepurko/docker-openvpn/blob/master/bin/ovpn_genconfig) of the Docker image.

# Acknowledgements

This project relies on the very comprehensive [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn) image.
