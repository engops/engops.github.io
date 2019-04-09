---
layout: default
title: OTP
nav_order: 2
parent: Keystone
gran_parent: OpenStack
permalink: /openstack/keystone/otp-keystone
---

```
DOMIN=bob
osdomain=${DOMIN}-domain
osproject=${DOMIN}-project
osuserproject=${DOMIN}-user
osuserpassword=${DOMIN}-123456
```



```
openstack domain create $osdomain
openstack project create --domain $osdomain $osproject
openstack user create --domain $osdomain --password $osuserpassword $osuserproject
openstack role add --project $osproject --project-domain $osdomain --user-domain $osdomain --user $osuserproject user
```




```
SECRET=$(
python << EOF
import base64
message = '	'
print base64.b32encode(message).rstrip('=')
EOF
)
```


echo $SECRET

USER_ID=$(openstack user show --domain $osdomain $osuserproject | awk ' /id/ && ! /domain/ {print $4}')

echo $USER_ID

PEGAR TOKEN DO USER ADMIN === ELE QUE ALTERA AS PORRA TODA
OS_TOKEN=$(openstack token issue | awk 'match($2,/^id$/) {print $4}')

echo $OS_TOKEN

curl -i -H "X-Auth-Token: $OS_TOKEN"  -H "Content-Type: application/json"   -d '
{
    "credential": {
        "blob": "'$SECRET'",
        "type": "totp",
        "user_id": "'$USER_ID'"
    }
}'   https://api-keystone.spo1.flexcloud.com.br:5000/v3/credentials ; echo



python << EOF
import qrcode
osuserproject="$osuserproject"
secret="$SECRET"
uri = 'otpauth://totp/{name}?secret={secret}&issuer={issuer}'.format(
    name=osuserproject,
    secret=secret,
    issuer='Keystone')

img = qrcode.make(uri)
img.save('osuserproject.png')
EOF



VALIDA COM SENHA ####
################
curl -i   -H "Content-Type: application/json"   -d '
{ "auth": {
    "identity": {
      "methods": ["password"],
      "password": {

        "user": {
          "name": "'$osuserproject'",
          "domain": { "name": "'$osdomain'" },
          "password": "'$osuserpassword'"
        }
      }
    },
    "scope": {
      "project": {
        "name": "'$osproject'",
        "domain": { "name": "'$osdomain'" }
      }
    }
  }
}'   "https://api-keystone.spo1.flexcloud.com.br:5000/v3/auth/tokens" ; echo



Usando o OTP ####
curl -i   -H "Content-Type: application/json"   -d '
{ "auth": {
    "identity": {
      "methods": ["totp"],
      "totp": {
        "user": {
          "name": "'$osuserproject'",
          "domain": { "name": "'$osdomain'" },
          "passcode": "523220"
        }
      }
    },
    "scope": {
      "project": {
        "name": "'$osproject'",
        "domain": { "name": "'$osdomain'" }
      }
    }
  }
}'   "https://api-keystone.spo1.flexcloud.com.br:5000/v3/auth/tokens" ; echo




