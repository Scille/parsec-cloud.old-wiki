Sequester feature requires two pairs of asymetric rsa keys: 
 - Service signature key: used at organization boostrap and is used to certified all services
 - Sequester encryption key: used by a sequester service to encrypt manifest.

First, generate those two pairs:
```
openssl genrsa -out encrypt_private_key.pem 4096
openssl rsa -in encrypt_private_key.pem -pubout > encrypt_public_key.pem

openssl genrsa -out signing_private_key.pem 4096
openssl rsa -in signing_private_key.pem -pubout > signing_public_key.pem
```

Then, create a fresh organizaton:
```
parsec.cli core  create_organization "TestSequester-XXX-rcXXX" --administration-token "XXXXXXX"  -B "XXXXX"
```

This command returns a bootstrap link. The public service signature key need to be provided during organizaton bootstrap
```
parsec.cli core bootstrap_organization "BOOTSTRAP_ADDR"  --sequester-verify-key signing_public_key.pem
```

The organization is ready. We are now using the backend command line interface that talks directly to the parsec DB. The database address shall be set as an env variable:
```
export PARSEC_DB=PARSEC_DB
```

Register a new sequester storage service:
```
parsec.cli backend sequester create_service --organization "TestSequester-XXX-rcXXXX" --service-label "TestServiceSequester" --service-public-key encryption_public_key.pem  --authority-private-key signing_private_key.pem --service-type storage
```
It requires the servicve public encryption key (use for manifest encryption) and also the service signing private key, to sign the encryption key.

New service shall be now in the service list:
```
python -m parsec.cli backend sequester list_services --organization "TestSequester-2-13-rc1"
```

A service can be enabled/disabled (service id is given from the `list_services` command:
```
python -m parsec.cli backend sequester update_service --disable  --organization "TestSequester-XXX-rcXXXX" --service SERVICE_ID
```

(Create some data into a realm to get exportable data)


Sequester data can be dump for a realm can be dump by an administrator. The realm id can be found using the following command:
```
python -m parsec.cli backend human_accesses  --organization "TestSequester-XXX-rcXXXX"
```

Data can be exported to an encrypted sqllite file:
```
python -m parsec.cli backend sequester export_realm --organization "TestSequester-XXX-rcXXXX" --realm=REALM_ID --output EXPORT_DIR -b BLOCK_STORAGE_ADDR
```

The workspace can be retrived with the encryption private key:
```
python -m parsec.cli backend sequester extract_realm_export --service-decryption-key ~/keys-sequester/encrypt_private_key.pem --input  EXPORT_DIR/SQLITE_FILE --output EXPORT_DIR
```






