

# OpenSSL Certificate Authority



---



## Generate Root Certificate
```
mkdir -p /root/CA
cd /root/CA
mkdir certs crl newcerts private OCSP
chmod 700 private
touch index.txt
echo 1000 > serial
echo 1000 > /root/CA/crlnumber
wget https://raw.githubusercontent.com/Pi-314159/Certificate-Authority/master/RootCA.cnf -O openssl.cnf
openssl ecparam -genkey -name secp384r1 | openssl ec -aes256 -out private/CA.key
```
```
chmod 400 private/CA.key
openssl req -config openssl.cnf -new -x509 -days 1826 -sha384 -extensions v3_ca -key private/CA.key -out certs/CA.crt
```
```
chmod 444 certs/CA.crt
```

> **Optional**
>
> *Generate the [OCSP Pair](#generate-the-ocsp-pair-root).*


---


## Intermediate Certificate

**Another Device.**
```
mkdir -p /root/IntermediateCA
cd /root/IntermediateCA
mkdir certs crl csr newcerts private OCSP
chmod 700 private
touch index.txt
echo 2000 > serial
wget https://raw.githubusercontent.com/Pi-314159/Certificate-Authority/master/IntmdCA.cnf -O openssl.cnf
echo 2000 > /root/IntermediateCA/crlnumber
openssl req -config openssl.cnf -new -newkey ec:<(openssl ecparam -name secp384r1) -keyout private/intermediate.key -out csr/intermediate.csr
```
```
chmod 400 private/intermediate.key
```

**Go Back.**
```
openssl ca -config openssl.cnf -extensions v3_intermediate_ca -days 730 -md sha384 -in intermediate.csr -out certs/intermediate.crt
```

> **Optional**
>
> *Verification:*
>
> `openssl verify -CAfile certs/CA.crt certs/intermediate.crt`
>
> *Enable [CRL](#generate-the-crl-root) and [Start the OCSP Server](#start-the-ocsp-server-root).*

**Another Device.**
```
chmod 444 /root/IntermediateCA/certs/intermediate.crt
cd /root/IntermediateCA/
```
> **Optional**
>
> *Generate the OCSP Pair.*
>
> *If You Have the CABundle:*
>
> `chmod 444 /root/IntermediateCA/certs/CABundle.crt`


---


## Sign Server and Client Certificates
```
openssl req -new -newkey rsa:2048 -out csr/domain.csr -keyout private/domain.key -subj "/C=KY/ST=/L=/O=/CN="
```
> **Optional**
>
> *ECC Certificate.*
>
> `openssl req -new -newkey ec:<(openssl ecparam -name secp384r1) -out csr/domain.csr -keyout private/domain.key -subj "/C=KY/ST=/L=/O=/CN="`
```
chmod 400 private/domain.key
```
*You Need to Modify openssl.cnf by Yourself.*
```
openssl ca -config openssl.cnf -extensions server_cert -days 366 -md sha384 -in csr/domain.csr -out certs/domain.crt
```
```
chmod 444 certs/domain.crt
```
> **Optional**
>
> *Enable CRL and Start the OCSP Server.*
>
> *Certificate Verification.*
>
> *Chain of Trust Verification:*
>
> `openssl verify -CAfile certs/CABundle.crt certs/domain.crt`


---


## Others 

### Generate the CRL (Root)
```
openssl ca -config openssl.cnf -gencrl -out crl/CA.crl
```
> **Optional**
>
> *Verification:*
>
> `openssl crl -in crl/CA.crl -noout -text `

### OCSP Server (Root)
#### Generate the OCSP Pair (Root)
```
openssl req -config openssl.cnf -new -newkey ec:<(openssl ecparam -name secp384r1) -keyout OCSP/key.pem -out OCSP/csr.pem
```
```
openssl ca -config openssl.cnf -extensions ocsp -days 1826 -notext -md sha384 -in OCSP/csr.pem -out OCSP/crt.pem
```
#### Start the OCSP Server (Root)
```
openssl ocsp -index index.txt -url http://example.com:56728 -rkey OCSP/key.pem -rsigner OCSP/crt.pem -CA certs/CA.crt -text -ndays 7 -out OCSP/log.txt -ignore_err
```
> **Optional**
>
> *Verification:*
>
> `openssl ocsp -CAfile certs/CA.crt -url http://127.0.0.1:4572 -resp_text -issuer certs/CA.crt -cert certs/intermediate.crt `
>
> *Continue to Run After Close the Terminal:*
>
> `^Z`
>
> `bg`
>
> `disown`

### Revoke Certificate (Root)
*This Works on Both OCSP and CRL.*

*Re-Generate CRL Is Required.*
```
openssl ca -config openssl.cnf -revoke certs/intermediate.crt -crl_reason superseded
```

### Generate CSR from Private Key
```
openssl req -new -key key.pem -out csr.pem -sha256 -subj "/CN=" -config config.cnf
```

### Convert PEM to DER
```
openssl x509 -inform PEM -in crt.pem -outform DER -out crt.cer
```

### Convert PEM to PFX
```
openssl pkcs12 -export -out crt.pfx -inkey key.pem -in crt.pem -certfile CA.crt
```

### PowerShell Code Signing
```
Set-AuthenticodeSignature .\PATH_TO_FILE.exe -Certificate (Get-ChildItem "Cert:\CurrentUser\My" -CodeSigningCert) -IncludeChain All -TimestampServer http://timestamp.verisign.com/scripts/timstamp.dll -HashAlgorithm SHA256
```

### Verify the Certificate
```
openssl x509 -noout -text -in crt.crt
```

### Extended Validation Certificate
#### OpenSSL Configuration
*Add Following Lines to the OpenSSL Config File If They Are Missing, Do Not Delete Existing Lines!*

[ new_oids ]
```
trustList = 2.16.840.1.113730.1.900
businessCategory = 2.5.4.15
jurisdictionOfIncorporationLocalityName = 1.3.6.1.4.1.311.60.2.1.1
jurisdictionOfIncorporationStateOrProvinceName = 1.3.6.1.4.1.311.60.2.1.2
jurisdictionOfIncorporationCountryName = 1.3.6.1.4.1.311.60.2.1.3
```
[ policy ]
```
businessCategory        = supplied
countryName             = supplied
stateOrProvinceName     = supplied
localityName            = supplied
streetAddress           = optional
organizationName        = supplied
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
jurisdictionL           = optional
jurisdictionST          = optional
jurisdictionC           = supplied
serialNumber            = supplied
```
[ all_cert ]
```
basicConstraints = CA:FALSE
nsCertType = server, client, email
nsComment = "PI Intermediate Certificate Authority Generated Certificate."
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth, emailProtection
crlDistributionPoints = @crl_sec
authorityInfoAccess = @OCSP_Info
certificatePolicies=@PICertTrust,2.23.140.1.1
subjectAltName=@alt_names
```
#### Generate CSR and KEY
*serialNumber Is Issued by Government.*
```
openssl req -config openssl.cnf -new -newkey rsa:2048 -days 366 -out csr/domain.csr -keyout private/domain.key -subj "/serialNumber=09358867287202/jurisdictionC=KY/jurisdictionST=/jurisdictionL=/C=KY/ST=/L=/O=/businessCategory=Private Organization/OU=/CN="
```
> **Optional**
>
> *ECC Certificate.*
>
> `openssl req -config openssl.cnf -new -newkey ec:<(openssl ecparam -name secp384r1) -days 366 -out csr/domain.csr -keyout private/domain.key -subj "/serialNumber=09358867287202/jurisdictionC=KY/jurisdictionST=/jurisdictionL=/C=KY/ST=/L=/O=/businessCategory=Private Organization/OU=/CN="`
#### Sign the Certificate
```
openssl ca -config openssl.cnf -verbose -startdate 19700101000001Z -enddate 19710101235959Z -extensions all_cert -md sha384 -in csr/domain.csr -out certs/domain.crt
```

### Certificate Policy
#### Certificate Type
| Type | OID |
|:-----:|:-----:|
| Domain Validated | 2.23.140.1.2.1 |
| Organization Validated | 2.23.140.1.2.2 |
| Extended Validation | 2.23.140.1.1 |
#### Certificate Authority
| Issuer | OID |
|:-----:|:-----:|
| Comodo | 1.3.6.1.4.1.6449.1.2.1.5.1 |
| DigiCert | 2.16.840.1.114412.2.1 |
| Entrust | 2.16.840.1.114028.10.1.2 |
| GlobalSign | 1.3.6.1.4.1.4146.1.1 |
| GoDaddy | 2.16.840.1.114413.1.7.23.3 |
| QuoVadis | 1.3.6.1.4.1.8024.0.2.100.1.2 |
