# Certificates Folder

You can use the provided tg binary to create a CA and corresponding
certificates. Simple instructions:

## Create a private CA

```bash
tg cert --name myca --org "Acme Enterprises" --common-name root --is-ca --pass secret
```

## Create a certificate for your server

```bash
tg cert --name myclient --org "Acme Enterprises" \
   --common-name demo-server \
   --auth-server --auth-client \
   --signing-cert myca-cert.pem  \
   --signing-cert-key myca-key.pem \
   --signing-cert-key-pass secret \
   --ip <your server IP> \
   --dns <your server DNS>
```