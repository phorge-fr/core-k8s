# Encryption

Create [age](https://age-encryption.org/) keys:

```bash
apt install age -y
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
```

Install sops:

```bash
VERSION="v3.9.1"
curl -LO https://github.com/getsops/sops/releases/download/$VERSION/sops-$VERSION.linux.amd64
mv sops-$VERSION.linux.amd64 /usr/local/bin/sops
chmod +x /usr/local/bin/sops
```

Create kubernetes secret:

```bash
age-keygen -o age.agekey # This should be a different key than yours
cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin
rm age.agekey
```