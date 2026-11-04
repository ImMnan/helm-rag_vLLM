Here are the steps. The CA cert is already exported to `~/Downloads/orca-infer-ca.crt` from earlier, but step 1 re-extracts it so the steps are self-contained.

**1. Extract the CA cert from the cluster**
```bash
kubectl get secret orca-infer-ca -n hawk -o jsonpath='{.data.ca\.crt}' | base64 -d > ~/Downloads/orca-infer-ca.crt
```

**2. Verify it's the right thing before trusting it**
```bash
openssl x509 -in ~/Downloads/orca-infer-ca.crt -noout -subject -issuer -dates -fingerprint -sha256
```
Expect:
```
subject=CN=orca-infer-ca
issuer=CN=orca-infer-ca
notAfter=Oct 31 09:18:14 2036 GMT
sha256 Fingerprint=EE:4F:C3:6D:66:77:1F:B3:8A:0F:A1:25:EE:34:B6:54:70:3F:90:AF:FA:0E:DA:7D:A7:E5:A5:98:28:C7:A8:78
```

**3. Confirm the gateway's leaf is actually signed by it**
```bash
echo | openssl s_client -connect orca-infer.ai:443 -servername orca-infer.ai -CAfile ~/Downloads/orca-infer-ca.crt 2>/dev/null | grep -E "Verify return code|issuer="
```
Expect `Verify return code: 0 (ok)` and `issuer=CN=orca-infer-ca`.

**4. Add the CA to the System keychain as trusted root** (prompts for your password)
```bash
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/Downloads/orca-infer-ca.crt
```
- `-d` → system (admin) trust domain, applies to all users
- `-r trustRoot` → trust as a root CA
- If you'd rather keep it user-only (no sudo): `security add-trusted-cert -r trustRoot -k ~/Library/Keychains/login.keychain-db ~/Downloads/orca-infer-ca.crt`

**5. Remove the stale old leaf entry**
Open **Keychain Access** → *System* (or *login*) → *Certificates*. The old self-signed leaf had no subject, so it shows with a blank or odd name. Find it by expiry **Jan 21 2027** (or search `orca-infer`), right-click → Delete. Alternatively from the terminal, by its SHA-1:
```bash
security find-certificate -a -Z /Library/Keychains/System.keychain | grep -B1 -A3 "orca-infer"   # locate it
sudo security delete-certificate -Z <SHA-1-hash> /Library/Keychains/System.keychain
```

**6. Verify trust from macOS's point of view**
```bash
security verify-cert -c ~/Downloads/orca-infer-ca.crt
```
Expect `...certificate verification successful.`

**7. Restart the browser** (fully quit Chrome — ⌘Q — not just close the tab; Chrome caches cert verdicts). Then hit `https://orca-infer.ai/ui` → it should redirect to `/auth` and both load with a valid padlock.

**Going forward:** the leaf under `orca-infer-tls` will keep rotating every ~60 days, but since it's now signed by the 10-year CA you just trusted, you won't need to repeat any of this. The only time you'd redo steps 1–4 is if the `orca-infer-ca` secret is ever deleted/recreated (new CA key).