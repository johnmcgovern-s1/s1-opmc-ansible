# certificates/

Put your console TLS material here, on the **control node**:

```
certificates/certificate.pem     the certificate (PEM encoded)
certificates/certificate.key     the private key (PEM encoded, NO passphrase)
```

The playbook copies both to `/etc/sentinelone/certs` on the target (root-only,
`0700`) and points `sentinel_pem_file` / `sentinel_key_file` at them.

`*.pem`, `*.key` and `*.crt` are gitignored — real certificates will not be
committed by accident. This README exists so the directory survives a clone.

Use different filenames by setting `s1_key_src` / `s1_pem_src` in
`group_vars/all/main.yml`.

See [INSTALL.md](../INSTALL.md) for PEM conversion commands and how to strip a
passphrase.
