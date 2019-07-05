---
layout: default
title: Federation Keystone
parent: Installation
gran_parent: OpenStack
nav_order: 2
---

#  Implementando federeação entre keystones de diferentes regiões.


Para isto será nessario:
  - Certificado Wildcard para cada região
  - Instalação e configuração do Shibboleth nas maquinas de keystone
  - Haproxy com persistência de sessão no backend do keystone
  - Configurando o Keystone para trabalhar como SP
  - Configuração do keystone para funcionar como IDP
  - Horizon versão Pike ou superior
  

## Exemplo
Neste vamos supor que temos 3x regiões, sendo que cada região tem 1x zona e 3x hosts de keystones:
Regiões e Zonas:
 - region-1
  - r1-az1
 - region-2
  - r2-az1
 - region-3
  - r3-az1

Keystones p/ region
 - keystone-1
 - keystone-2
 - keystone-3

---

## Criação do certificado auto-assinado do tipo wildcard para cada região

Primeiramente será necessário criar um certificado wildcard para cada região com todas as possíveis domínios que você tenha em sua infraestrutura.
Abaixo está uma breve instrução de como criar um certificado auto-assinado do tipo wildcard. Execute o mesmo para cada região.

Execute o comando seguinte para definir as variaveis de região e zona (Obs.: Altere conforme a região e zona que se encontra)
~~~bash
domain_internal=domaincompany.corp
domain_external=domaincompany.com
region=region-1
zone=r1-az1
~~~

Execute o comando seguinte para gerar o arquivo com os dados do certificado.
~~~bash
echo "[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no
[req_distinguished_name]
C = BR
ST = SAO_PAULO
L = SAO_PAULO
O  = company
CN = *.$region.$domain
[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = *.$region.$domain_external
DNS.2 = *.$region.$domain_internal
DNS.3 = *.$zone.$domain_external
DNS.4 = *.$zone.$domain_internal" > /etc/CA/$region-wildcard.cfg
~~~

Execute agora o comando seguinte para gerar o certificado com base no conf criado.
~~~bash
openssl req -new -newkey rsa:2048 -sha256 -nodes -out /etc/CA/certs/$region-wildcard.csr -keyout /etc/CA/certs/$region-wildcard.key -config /etc/CA/$region-wildcard.cfg
~~~

Para finalizar, execute o seguinte passo para assinar o certificado com CA interna
~~~bash
openssl ca -batch -config /etc/CA/openssl.cnf -days 9999 -in /etc/CA/certs/$region-wildcard.csr -out /etc/CA/certs/$region-wildcard.crt -keyfile /etc/CA/private/rootCA.key -cert /etc/CA/certs/rootCA.crt -policy policy_anything
~~~


Transfira cada certificado região para todas as máquinas de keystone da devida região
~~~bash
scp /etc/CA/certs/$region-wildcard* keystone-1.$region.$domain_internal:/usr/local/ssl/
scp /etc/CA/certs/$region-wildcard* keystone-2.$region.$domain_internal:/usr/local/ssl/
scp /etc/CA/certs/$region-wildcard* keystone-3.$region.$domain_internal:/usr/local/ssl/
~~~

---

## Instalando e configurando o shibboleth no Keystone:

Instale os seguintes pacotes em todos os keystones: 
~~~bash
yum install -y shibboleth xmlsec1 xmlsec1-openssl
~~~

Edite o sequinte arquivo em todos os keystones, alterando algumas informações de região. (Neste exemplo abaixo eu farei para "region-1", repita o passo para as outras regiões)
vim /etc/shibboleth/shibboleth2.xml 
~~~bash
<SPConfig xmlns="urn:mace:shibboleth:2.0:native:sp:config"
    xmlns:conf="urn:mace:shibboleth:2.0:native:sp:config"
    xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
    xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"    
    xmlns:md="urn:oasis:names:tc:SAML:2.0:metadata"
    clockSkew="180">

    <RequestMapper type="Native">
        <RequestMap applicationId="default">
            <Host name="vip-keystoneendpoint.region-1.internal_domain">
                <Path name="idp-region-2" applicationId="idp-region-2" authType="shibboleth" requireSession="true"/>
                <Path name="idp-region-3" applicationId="idp-region-3" authType="shibboleth" requireSession="true"/>
            </Host>
        </RequestMap>
    </RequestMapper>

    <ApplicationDefaults entityID="https://vip-keystoneendpoint.region-1.internal_domain:5000/shibboleth" REMOTE_USER="eppn persistent-id targeted-id">

        <Sessions lifetime="28800" timeout="3600" relayState="ss:mem"
                  checkAddress="false" handlerSSL="false" cookieProps="https">
            <SSO discoveryProtocol="SAMLDS" discoveryURL="https://vip-keystoneendpoint.region-1.internal_domain:5000/DS">
                SAML2 SAML1
            </SSO>
        </Sessions>

        <Errors supportContact="root@localhost"
            helpLocation="/about.html"
            styleSheet="/shibboleth-sp/main.css"/>
        
        <MetadataProvider type="XML" validate="true"
	      uri="http://example.org/federation-metadata.xml"
              backingFilePath="federation-metadata.xml" reloadInterval="7200">
            <MetadataFilter type="RequireValidUntil" maxValidityInterval="2419200"/>
            <MetadataFilter type="Signature" certificate="fedsigner.pem"/>
            <DiscoveryFilter type="Blacklist" matcher="EntityAttributes" trimTags="true" 
              attributeName="http://macedir.org/entity-category"
              attributeNameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"
              attributeValue="http://refeds.org/category/hide-from-discovery" />
        </MetadataProvider>
    
        <ApplicationOverride id="idp-region-2">
            <Sessions lifetime="28800" timeout="3600" relayState="ss:mem"
                checkAddress="false" handlerSSL="true" cookieProps="https"
                handlerURL="/idp-region-2/Shibboleth.sso">

                <SSO discoveryProtocol="SAMLDS" entityID="https://vip-keystoneendpoint.idp-region-2.internal_domain:5000/v3/OS-FEDERATION/saml2/idp" forceAuthn="true"
                    discoveryURL="https://vip-keystoneendpoint.idp-region-2.internal_domain:5000/DS">
                    SAML2 SAML1
                </SSO>
                <Logout>SAML2 Local</Logout>
                <Handler type="MetadataGenerator" Location="/Metadata" signing="false"/>
                <Handler type="Status" Location="/Status" acl="127.0.0.1 ::1"/>
                <Handler type="Session" Location="/Session" showAttributeValues="false"/>
                <Handler type="DiscoveryFeed" Location="/DiscoFeed"/>
            </Sessions>
            <MetadataProvider type="Chaining">
                <MetadataProvider type="XML" uri="https://vip-keystoneendpoint.idp-region-2.internal_domain:5000/v3/OS-FEDERATION/saml2/metadata"/>
            </MetadataProvider>
        </ApplicationOverride>

        <ApplicationOverride id="idp-region-3">
            <Sessions lifetime="28800" timeout="3600" relayState="ss:mem"
                checkAddress="false" handlerSSL="true" cookieProps="https"
                handlerURL="/idp-region-3/Shibboleth.sso">

                <SSO discoveryProtocol="SAMLDS" entityID="https://vip-keystoneendpoint.idp-region-3.internal_domain:5000/v3/OS-FEDERATION/saml2/idp" forceAuthn="true"
                    discoveryURL="https://vip-keystoneendpoint.idp-region-3.internal_domain:5000/DS">
                    SAML2 SAML1
                </SSO>
                <Logout>SAML2 Local</Logout>
                <Handler type="MetadataGenerator" Location="/Metadata" signing="false"/>
                <Handler type="Status" Location="/Status" acl="127.0.0.1 ::1"/>
                <Handler type="Session" Location="/Session" showAttributeValues="false"/>
                <Handler type="DiscoveryFeed" Location="/DiscoFeed"/>
            </Sessions>
            <MetadataProvider type="Chaining">
                <MetadataProvider type="XML" uri="https://vip-keystoneendpoint.idp-region-3.internal_domain:5000/v3/OS-FEDERATION/saml2/metadata"/>
            </MetadataProvider>
        </ApplicationOverride>
    </ApplicationDefaults>   
</SPConfig>
~~~

Crie o sequinte arquivo em todos os keystones. (Neste exemplo o arquivo será igual para todas as regiões)
vim /etc/shibboleth/attribute-map.xml
~~~bash
<Attributes xmlns="urn:mace:shibboleth:2.0:attribute-map" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <Attribute name="openstack_user" id="openstack_user"/>
    <Attribute name="openstack_roles" id="openstack_roles"/>
    <Attribute name="openstack_project" id="openstack_project"/>
    <Attribute name="openstack_user_domain" id="openstack_user_domain"/>
    <Attribute name="openstack_project_domain" id="openstack_project_domain"/>
    <Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.6" id="eppn">
        <AttributeDecoder xsi:type="ScopedAttributeDecoder"/>
    </Attribute>
    <Attribute name="urn:mace:dir:attribute-def:eduPersonPrincipalName" id="eppn">
        <AttributeDecoder xsi:type="ScopedAttributeDecoder"/>
    </Attribute>

    <Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.9" id="affiliation">
        <AttributeDecoder xsi:type="ScopedAttributeDecoder" caseSensitive="false"/>
    </Attribute>
    <Attribute name="urn:mace:dir:attribute-def:eduPersonScopedAffiliation" id="affiliation">
        <AttributeDecoder xsi:type="ScopedAttributeDecoder" caseSensitive="false"/>
    </Attribute>

    <Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.1" id="unscoped-affiliation">
        <AttributeDecoder xsi:type="StringAttributeDecoder" caseSensitive="false"/>
    </Attribute>
    <Attribute name="urn:mace:dir:attribute-def:eduPersonAffiliation" id="unscoped-affiliation">
        <AttributeDecoder xsi:type="StringAttributeDecoder" caseSensitive="false"/>
    </Attribute>

    <Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.7" id="entitlement"/>
    <Attribute name="urn:mace:dir:attribute-def:eduPersonEntitlement" id="entitlement"/>
    <Attribute name="urn:mace:dir:attribute-def:eduPersonTargetedID" id="targeted-id">
        <AttributeDecoder xsi:type="ScopedAttributeDecoder"/>
    </Attribute>
     
    <Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.10" id="persistent-id">
        <AttributeDecoder xsi:type="NameIDAttributeDecoder" formatter="$NameQualifier!$SPNameQualifier!$Name" defaultQualifiers="true"/>
    </Attribute>

    <Attribute name="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent" id="persistent-id">
        <AttributeDecoder xsi:type="NameIDAttributeDecoder" formatter="$NameQualifier!$SPNameQualifier!$Name" defaultQualifiers="true"/>
    </Attribute>
</Attributes>
~~~


Crie o sequinte arquivo em todos os keystones. (Neste exemplo o arquivo será igual para todas as regiões)
vim /usr/local/sbin/rules.json
~~~bash
[{
    "remote": [
    {
            "type": "openstack_user"
    },
    {
            "type": "openstack_roles"
    },
    {
            "type": "openstack_project"
    },
    {
            "type": "openstack_project_domain"
    },
    {
            "type": "openstack_user_domain"
    }
    ],
    "local": [{
            "user": {
                    "name": "{0}",
                    "type": "local",
                    "domain": {
                       "name": "{3}"
            }
            }
    }]
}]
~~~

---

## Criando o mapiamento e relacionamento entre IDPs e SPs no Openstack de cada região:

Agora iremos criar mapiamentos e o relacionamentos entre os IDPs e SPs em cada região.
Vamos executar primeiro na region-1, pegue o source de Admin do openstack da region-1 e execute os seguintes comandos:

~~~bash
openstack mapping create --rules rules.json keystone_idp_mapping

openstack identity provider create --remote-id https://vip-keystoneendpoint.region-2.cloudevolution.corp:5000/v3/OS-FEDERATION/saml2/idp idp-region-2
openstack identity provider create --remote-id https://vip-keystoneendpoint.region-3.cloudevolution.corp:5000/v3/OS-FEDERATION/saml2/idp idp-region-3

openstack federation protocol create saml2 --mapping keystone_idp_mapping --identity-provider idp-region-2
openstack federation protocol create saml2 --mapping keystone_idp_mapping --identity-provider idp-region-3

openstack service provider create --service-provider-url '"https://vip-keystoneendpoint.idp-region-2.internal_domain:5000/idp-region-1/Shibboleth.sso/SAML2/ECP' --auth-url https://vip-keystoneendpoint.idp-region-2.internal_domain:5000/v3/OS-FEDERATION/identity_providers/idp-region-1/protocols/saml2/auth r2-az1
openstack service provider create --service-provider-url '"https://vip-keystoneendpoint.idp-region-3.internal_domain:5000/idp-region-1/Shibboleth.sso/SAML2/ECP' --auth-url https://vip-keystoneendpoint.idp-region-3.internal_domain:5000/v3/OS-FEDERATION/identity_providers/idp-region-1/protocols/saml2/auth r3-az1
~~~

Agora na region-2, pegue o source de Admin do openstack da region-2 e execute os seguintes comandos:
~~~bash
openstack mapping create --rules rules.json keystone_idp_mapping

openstack identity provider create --remote-id https://vip-keystoneendpoint.region-1.cloudevolution.corp:5000/v3/OS-FEDERATION/saml2/idp idp-region-1
openstack identity provider create --remote-id https://vip-keystoneendpoint.region-3.cloudevolution.corp:5000/v3/OS-FEDERATION/saml2/idp idp-region-3

openstack federation protocol create saml2 --mapping keystone_idp_mapping --identity-provider idp-region-1
openstack federation protocol create saml2 --mapping keystone_idp_mapping --identity-provider idp-region-3

openstack service provider create --service-provider-url '"https://vip-keystoneendpoint.idp-region-1.internal_domain:5000/idp-region-1/Shibboleth.sso/SAML2/ECP' --auth-url https://vip-keystoneendpoint.idp-region-1.internal_domain:5000/v3/OS-FEDERATION/identity_providers/idp-region-1/protocols/saml2/auth r1-az1
openstack service provider create --service-provider-url '"https://vip-keystoneendpoint.idp-region-3.internal_domain:5000/idp-region-1/Shibboleth.sso/SAML2/ECP' --auth-url https://vip-keystoneendpoint.idp-region-3.internal_domain:5000/v3/OS-FEDERATION/identity_providers/idp-region-1/protocols/saml2/auth r3-az1
~~~

Por fim na region-3, pegue o source de Admin do openstack da region-3 e execute os seguintes comandos:
~~~bash
openstack mapping create --rules rules.json keystone_idp_mapping

openstack identity provider create --remote-id https://vip-keystoneendpoint.region-1.cloudevolution.corp:5000/v3/OS-FEDERATION/saml2/idp idp-region-1
openstack identity provider create --remote-id https://vip-keystoneendpoint.region-3.cloudevolution.corp:5000/v3/OS-FEDERATION/saml2/idp idp-region-2

openstack federation protocol create saml2 --mapping keystone_idp_mapping --identity-provider idp-region-1
openstack federation protocol create saml2 --mapping keystone_idp_mapping --identity-provider idp-region-2

openstack service provider create --service-provider-url '"https://vip-keystoneendpoint.idp-region-1.internal_domain:5000/idp-region-1/Shibboleth.sso/SAML2/ECP' --auth-url https://vip-keystoneendpoint.idp-region-1.internal_domain:5000/v3/OS-FEDERATION/identity_providers/idp-region-1/protocols/saml2/auth r1-az1
openstack service provider create --service-provider-url '"https://vip-keystoneendpoint.idp-region-2.internal_domain:5000/idp-region-1/Shibboleth.sso/SAML2/ECP' --auth-url https://vip-keystoneendpoint.idp-region-2.internal_domain:5000/v3/OS-FEDERATION/identity_providers/idp-region-1/protocols/saml2/auth r2-az1
~~~

---

## Configurando o Haproxy para trabalhar com persistencia de sessão:

Edite o arquivo do haproxy
vim /etc/haproxy/haproxy.conf
~~~
frontend Front_Keystone-Public-5000
        mode http
        bind 0.0.0.0:5000
        reqadd X-Forwarded-Proto:\ https
        default_backend Back_Keystone-Public-5000

backend Back_Keystone-Public-5000
        mode http
        cookie SERVERID insert indirect nocache
        option httpchk OPTIONS / HTTP/1.0
        http-check expect rstatus (2|3)[0-9][0-9]
        server keystone-1 keystone-1:5000 check ssl verify none weight 1 cookie k1
        server keystone-2 keystone-2:5000 check ssl verify none weight 1 cookie k2
        server keystone-3 keystone-3:5000 check ssl verify none weight 1 cookie k3

frontend Front_Keystone-Admin-35357
        mode http
        bind 0.0.0.0:35357
        reqadd X-Forwarded-Proto:\ https
        default_backend Back_Keystone-Admin-35357

backend Back_Keystone-Admin-35357
        mode http
        cookie SERVERID insert indirect nocache
        option httpchk OPTIONS / HTTP/1.0
        http-check expect rstatus (2|3)[0-9][0-9]
        server keystone-1 keystone-1:35357 check ssl verify none weight 1 cookie k1
        server keystone-2 keystone-2:35357 check ssl verify none weight 1 cookie k2
        server keystone-3 keystone-3:35357 check ssl verify none weight 1 cookie k3
~~~

Reinie o serviços do Haproxy

~~~bash
systemctl restart haproxy
~~~
---

## Configurando o Keystone para trabalhar como SP:

Edite o arquivo de conf. do apache responsável por subir o WSGI do Keystone. (/etc/httpd/conf.d/wsgi-keystone.conf)

Dentro do virtualhost do admin (35357), adicione as linhas:
~~~bash
    WSGIScriptAliasMatch ^(/v3/OS-FEDERATION/identity_providers/.*?/protocols/.*?/auth)$ /usr/bin/keystone-wsgi-public/$1

    <Location /idp-region-2/Shibboleth.sso>
      SetHandler shib
      ShibRequestSetting applicationId idp-region-2
      Require all granted
    </Location>
    
    <Location /v3/OS-FEDERATION/identity_providers/idp-region-2/protocols/saml2/auth>
      ShibRequestSetting applicationId idp-region-2
      ShibRequestSetting requireSession 1
      AuthType shibboleth
      ShibExportAssertion Off
      Require valid-user

      <IfVersion < 2.4>
        ShibRequireSession On
        ShibRequireAll On
      </IfVersion>
    </Location>
    

    <Location /idp-region-3/Shibboleth.sso>
      SetHandler shib
      ShibRequestSetting applicationId idp-region-3
      Require all granted
    </Location>
    
    <Location /v3/OS-FEDERATION/identity_providers/idp-region-3/protocols/saml2/auth>
      ShibRequestSetting applicationId idp-region-3
      ShibRequestSetting requireSession 1
      AuthType shibboleth
      ShibExportAssertion Off
      Require valid-user

      <IfVersion < 2.4>
        ShibRequireSession On
        ShibRequireAll On
      </IfVersion>
    </Location>
~~~

---

## Configurando o Keystone para trabalhar como IDP:

Edite o arquivo de configuração do keystone com as seguintes informações:
vim /etc/keystone/jeystone.conf
~~~bash
[auth]
methods = external,password,token,saml2

[saml2]
remote_id_attribute = Shib-Identity-Provider

[saml]
certfile = /usr/local/ssl/region-1-wildcard.crt
keyfile = /usr/local/ssl/region-1-wildcard.key
idp_entity_id = https://vip-keystoneendpoint.region-1.internal_domain:5000/v3/OS-FEDERATION/saml2/idp
idp_sso_endpoint =https://vip-keystoneendpoint.region-1.internal_domain/v3/OS-FEDERATION/saml2/sso
idp_organization_name = Company
idp_organization_display_name = Company
idp_organization_url = https://domain_external.com
idp_contact_company = Engineering
idp_contact_name = Openstack Cloud
idp_contact_surname = Openstack Cloud 
idp_contact_email = cloudevolution-infra@domain_external.com
idp_contact_telephone = +55119999999
idp_contact_type = technical
idp_metadata_path = /etc/keystone/saml2_idp_metadata.xml
~~~

Restart todos os keystones de cada região, para que eles subam as novas confs.

~~~bash
systemctl restart httpd
~~~

Agora gere o metadata com base no certificado do IDP (certificado wildcard).
~~~bash
keystone-manage saml_idp_metadata > /etc/keystone/saml2_idp_metadata.xml
~~~


Para finaliza, realize o restart todos os keystones e shibboletes de cada região, para que eles possam se conectar um a outro e baixem o metadado de cada região.
Faça isso região por região e repita 2x este procedimento.

~~~bash
systemctl restart httpd shibd
~~~

---

## Configuração do Horizon (Pike ou superior):

Edite o arquivo de configuração de todos os horizon com a seguinte informação e em todos as regiões, apenas altere a zona conforme a região:

vim /etc/openstack-dashboard/local_settings
~~~bash
KEYSTONE_PROVIDER_IDP_NAME = "r1-az1"
~~~

Agora realize o restart do serviço em todos os horizon.
~~~bash
systemctl restart httpd
~~~