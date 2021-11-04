# external-dns
External-dns orchestrator for Kubernetes


# Configuring RFC2136 provider
This tutorial describes how to use the RFC2136 with either BIND or Windows DNS.

## Using with BIND
To use external-dns with BIND: generate/procure a key, configure DNS and add a
deployment of external-dns.

### Server credentials:
- RFC2136 was developed for and tested with
[BIND](https://www.isc.org/downloads/bind/) DNS server. This documentation
assumes that you already have a configured and working server. If you don't,
please check BIND documents or tutorials.
- If your DNS is provided for you, ask for a TSIG key authorized to update and
transfer the zone you wish to update. The key will look something like below.
Skip the next steps wrt BIND setup.
```text
key "externaldns-key" {
	algorithm hmac-sha256;
	secret "96Ah/a2g0/nLeFGK+d/0tzQcccf9hCEIy34PoXX2Qg8=";
};
```
- If you are your own DNS administrator create a TSIG key. Use
`tsig-keygen -a hmac-sha256 externaldns` or on older distributions
`dnssec-keygen -a HMAC-SHA256 -b 256 -n HOST externaldns`. You will end up with
a key printed to standard out like above (or in the case of dnssec-keygen in a
file called `Kexternaldns......key`).

### BIND Configuration:
If you do not administer your own DNS, skip to RFC provider configuration

- Edit your named.conf file (or appropriate included file) and add/change the
following.
  - Make sure You are listening on the right interfaces. At least whatever
  interface external-dns will be communicating over and the interface that
  faces the internet.
  - Add the key that you generated/was given to you above. Copy paste the four
  lines that you got (not the same as the example key) into your file.
  - Create a zone for kubernetes. If you already have a zone, skip to the next
  step. (I put the zone in it's own subdirectory because named,
  which shouldn't be running as root, needs to create a journal file and the
  default zone directory isn't writeable by named).
  ```text
  zone "k8s.example.org" {
      type master;
      file "/etc/bind/pri/k8s/k8s.zone";
  };
  ```
  - Add your key to both transfer and update. For instance with our previous
  zone.
  ```text
  zone "k8s.example.org" {
      type master;
      file "/etc/bind/pri/k8s/k8s.zone";
      allow-transfer {
          key "externaldns-key";
      };
      update-policy {
          grant externaldns-key zonesub ANY;
      };
  };
  ```
  - Create a zone file (k8s.zone):
  ```text
  $TTL 60 ; 1 minute
  k8s.example.org         IN SOA  k8s.example.org. root.k8s.example.org. (
                                  16         ; serial
                                  60         ; refresh (1 minute)
                                  60         ; retry (1 minute)
                                  60         ; expire (1 minute)
                                  60         ; minimum (1 minute)
                                  )
                          NS      ns.k8s.example.org.
  ns                      A       123.456.789.012
  ```
  - Reload (or restart) named


### Using external-dns
To use external-dns add an ingress or a LoadBalancer service with a host that
is part of the domain-filter. For example both of the following would produce
A records.
```text
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: svc.example.org
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: my-ingress
spec:
    rules:
    - host: ingress.example.org
      http:
          paths:
          - path: /
            backend:
                serviceName: my-service
                servicePort: 8000
```

### Custom TTL

The default DNS record TTL (Time-To-Live) is 0 seconds. You can customize this value by setting the annotation `external-dns.alpha.kubernetes.io/ttl`. e.g., modify the service manifest YAML file above:

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: nginx.external-dns-test.my-org.com
    external-dns.alpha.kubernetes.io/ttl: 60
spec:
    ...
```

This will set the DNS record's TTL to 60 seconds.

A default TTL for all records can be set using the the flag with a time in seconds, minutes or hours, such as `--rfc2136-min-ttl=60s`

There are other annotation that can affect the generation of DNS records, but these are beyond the scope of this
tutorial and are covered in the main documentation.

### Test with external-dns installed on local machine (optional)
You may install external-dns and test on a local machine by running:
```external-dns --txt-owner-id k8s --provider rfc2136 --rfc2136-host=192.168.0.1 --rfc2136-port=53 --rfc2136-zone=k8s.example.org --rfc2136-tsig-secret=96Ah/a2g0/nLeFGK+d/0tzQcccf9hCEIy34PoXX2Qg8= --rfc2136-tsig-secret-alg=hmac-sha256 --rfc2136-tsig-keyname=externaldns-key --rfc2136-tsig-axfr --source ingress --once --domain-filter=k8s.example.org --dry-run```
- host should be the IP of your master DNS server.
- tsig-secret should be changed to match your secret.
- tsig-keyname needs to match the keyname you used (if you changed it).
- domain-filter can be used as shown to filter the domains you wish to update.

### RFC2136 provider configuration:
In order to use external-dns with your cluster you need to add a deployment
with access to your ingress and service resources. The following are two
example manifests with and without RBAC respectively.

- With RBAC:
```text
apiVersion: v1
kind: Namespace
metadata:
  name: external-dns
  labels:
    name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
  namespace: external-dns
rules:
- apiGroups:
  - ""
  resources:
  - services
  - endpoints
  - pods
  - nodes
  verbs:
  - get
  - watch
  - list
- apiGroups:
  - extensions
  - networking.k8s.io
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  namespace: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
  namespace: external-dns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: external-dns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: external-dns
spec:
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: k8s.gcr.io/external-dns/external-dns:v0.7.6
        args:
        - --registry=txt
        - --txt-prefix=external-dns-
        - --txt-owner-id=k8s
        - --provider=rfc2136
        - --rfc2136-host=192.168.0.1
        - --rfc2136-port=53
        - --rfc2136-zone=k8s.example.org
        - --rfc2136-tsig-secret=96Ah/a2g0/nLeFGK+d/0tzQcccf9hCEIy34PoXX2Qg8=
        - --rfc2136-tsig-secret-alg=hmac-sha256
        - --rfc2136-tsig-keyname=externaldns-key
        - --rfc2136-tsig-axfr
        - --source=ingress
        - --domain-filter=k8s.example.org
```

- Without RBAC:
```text
apiVersion: v1
kind: Namespace
metadata:
  name: external-dns
  labels:
    name: external-dns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: external-dns
spec:
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      containers:
      - name: external-dns
        image: k8s.gcr.io/external-dns/external-dns:v0.7.6
        args:
        - --registry=txt
        - --txt-prefix=external-dns-
        - --txt-owner-id=k8s
        - --provider=rfc2136
        - --rfc2136-host=192.168.0.1
        - --rfc2136-port=53
        - --rfc2136-zone=k8s.example.org
        - --rfc2136-tsig-secret=96Ah/a2g0/nLeFGK+d/0tzQcccf9hCEIy34PoXX2Qg8=
        - --rfc2136-tsig-secret-alg=hmac-sha256
        - --rfc2136-tsig-keyname=externaldns-key
        - --rfc2136-tsig-axfr
        - --source=ingress
        - --domain-filter=k8s.example.org
```

## Microsoft DNS (Insecure Updates)

While `external-dns` was not developed or tested against Microsoft DNS, it can be configured to work against it. YMMV.

### Insecure Updates

#### DNS-side configuration

1. Create a DNS zone
2. Enable insecure dynamic updates for the zone
3. Enable Zone Transfers to all servers

#### `external-dns` configuration

You'll want to configure `external-dns` similarly to the following:

```text
...
        - --provider=rfc2136
        - --rfc2136-host=192.168.0.1
        - --rfc2136-port=53
        - --rfc2136-zone=k8s.example.org
        - --rfc2136-insecure
        - --rfc2136-tsig-axfr # needed to enable zone transfers, which is required for deletion of records.
...
```

### Secure Updates Using RFC3645 (GSS-TSIG)

### DNS-side configuration

1. Create a DNS zone
2. Enable secure dynamic updates for the zone
3. Enable Zone Transfers to all servers

If you see any error messages which indicate that `external-dns` was somehow not able to fetch
existing DNS records from your DNS server, this could mean that you forgot about step 3.

#### Kerberos Configuration

DNS with secure updates relies upon a valid Kerberos configuration running within the `external-dns` container.  At this time, you will need to create a ConfigMap for the `external-dns` container to use and mount it in your deployment.  Below is an example of a working Kerberos configuration inside a ConfigMap definition.  This may be different depending on many factors in your environment:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: krb5.conf
data:
  krb5.conf: |
    [logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log

    [libdefaults]
    dns_lookup_realm = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false
    pkinit_anchors = /etc/pki/tls/certs/ca-bundle.crt
    default_ccache_name = KEYRING:persistent:%{uid}

    default_realm = YOUR-REALM.COM

    [realms]
    YOUR-REALM.COM = {
      kdc = dc1.yourdomain.com
      admin_server = dc1.yourdomain.com
    }

    [domain_realm]
    yourdomain.com = YOUR-REALM.COM
    .yourdomain.com = YOUR-REALM.COM
```
In most cases, the realm name will probably be the same as the domain name, so you can simply replace
`YOUR-REALM.COM` with something like `YOURDOMAIN.COM`.

Once the ConfigMap is created, the container `external-dns` container needs to be told to mount that ConfigMap as a volume at the default Kerberos configuration location.  The pod spec should include a similar configuration to the following:

```yaml
...
    volumeMounts:
    - mountPath: /etc/krb5.conf
      name: kerberos-config-volume
      subPath: krb5.conf
...
  volumes:
  - configMap:
      defaultMode: 420
      name: krb5.conf
    name: kerberos-config-volume
...
```

#### `external-dns` configuration

You'll want to configure `external-dns` similarly to the following:

```text
...
        - --provider=rfc2136
	 - --rfc2136-gss-tsig
        - --rfc2136-host=dns-host.yourdomain.com
        - --rfc2136-port=53
        - --rfc2136-zone=your-zone.com
        - --rfc2136-kerberos-username=your-domain-account
        - --rfc2136-kerberos-password=your-domain-password
        - --rfc2136-kerberos-realm=your-domain.com
        - --rfc2136-tsig-axfr # needed to enable zone transfers, which is required for deletion of records.
...
```

As noted above, the `--rfc2136-kerberos-realm` flag is completely optional and won't be necessary in many cases.
Most likely, you will only need it if you see errors similar to this: `KRB Error: (68) KDC_ERR_WRONG_REALM Reserved for future use`.

The flag `--rfc2136-host` can be set to the host's domain name or IP address.
However, it also determines the name of the Kerberos principal which is used during authentication.
This means that Active Directory might only work if this is set to a specific domain name, possibly leading to errors like this:
`KDC_ERR_S_PRINCIPAL_UNKNOWN Server not found in Kerberos database`.
To fix this, try setting `--rfc2136-host` to the "actual" hostname of your DNS server.
