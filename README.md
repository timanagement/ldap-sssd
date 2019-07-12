# Autenticação com PAM + SSSD + OpenLDAP

Existem três máquinas com três sistemas diferentes:

| Servidor | IP          | Papel               |
|----------|-------------|---------------------|
| opensuse | 27.11.90.10 | LDAP Server, Client |
| debian   | 27.11.90.20 | Client              |
| centos   | 27.11.90.30 | Client              |

## openSUSE

Instale os pacotes referentes ao OpenLDAP, SSSD e suas dependências:

```
zypper install -y openldap2 openldap2-client sssd-ldap
```

Caso não utilize DNS:

```
echo '27.11.90.10 ldap.example.com' >> /etc/hosts
```

#### Criando somente o certificado

Crie um certificado TLS pois é considerado uma obrigatoriedade pelo SSSD:

```
mkdir -p /etc/openldap/tls/
openssl req -x509 -newkey rsa:4096 -nodes -keyout /etc/openldap/tls/key.pem -out /etc/openldap/tls/cert.pem -days 365
chown -R ldap: /etc/openldap/tls
```

#### Criando uma CA

```
mkdir -p /usr/share/ca/{certs,newcerts,private,crl}
echo 01 > serial
> index.txt

cd /usr/share/ca/
openssl req -config /etc/ssl/openssl.cnf -new -x509 -extensions v3_ca -keyout private/ca.key -out ca.cert -days 3650
...
Enter PEM pass phrase: 1234
Verifying - Enter PEM pass phrase: 1234
...
Country Name (2 letter code) [AU]:BR
State or Province Name (full name) [Some-State]:São Paulo
Locality Name (eg, city) []:Osasco
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Ldap Example Ltd 
Organizational Unit Name (eg, section) []:TI
Common Name (e.g. server FQDN or YOUR name) []:ldap.example.com
Email Address []:

chmod 0400 private/ca.key

mkdir -p /etc/openldap/tls/
openssl req -config /etc/ssl/openssl.cnf -newkey rsa:2048 -sha256 -nodes -out /etc/openldap/tls/ldap_cert.csr -outform PEM -keyout /etc/openldap/tls/ldap_key.pem
Country Name (2 letter code) [AU]:BR
State or Province Name (full name) [Some-State]:São Paulo
Locality Name (eg, city) []:São Paulo
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Ldap Example Ltd
Organizational Unit Name (eg, section) []:TI
Common Name (e.g. server FQDN or YOUR name) []:ldap.example.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:

openssl ca -config /etc/ssl/openssl.cnf -policy signing_policy -extensions signing_req -out /etc/openldap/tls/ldap_cert.pem -infiles /etc/openldap/tls/ldap_cert.csr
cp ca.cert /etc/openldap/tls/
chown -R ldap: /etc/openldap/
```

Modifique o arquivo **/etc/openldap/slapd.conf** para configurar os parâmetros iniciais do OpenLDAP:

```
pidfile         /run/slapd/slapd.pid
argsfile        /run/slapd/slapd.args

include /etc/openldap/schema/core.schema
include /etc/openldap/schema/cosine.schema
include /etc/openldap/schema/inetorgperson.schema
include /etc/openldap/schema/rfc2307bis.schema
include /etc/openldap/schema/yast.schema

modulepath /usr/lib64/openldap
moduleload back_mdb.la

access to dn.base=""
        by * read

access to dn.base="cn=Subschema"
        by * read

access to attrs=userPassword,userPKCS12
        by self write
        by * auth

access to attrs=shadowLastChange
        by self write
        by * read

access to *
        by * read

database     mdb
suffix       "dc=ldap,dc=example,dc=com"
rootdn       "cn=admin,dc=ldap,dc=example,dc=com"
rootpw       {SSHA}E7SBIzuFwXzf4AdYnWBhft9ynWnIWqww
directory    /var/lib/ldap
index        objectClass eq

TLSCACertificateFile /etc/openldap/tls/cert.pem
TLSCertificateFile /etc/openldap/tls/cert.pem
TLSCertificateKeyFile /etc/openldap/tls/key.pem
```

Inicie o openLDAP:

```
systemctl start slapd
```

#### Verificar conexão simples:

```
opensuse:~ # ldapsearch -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -LLL
No such object (32)
```

#### Verificar conexão TLS com STARTTLS:

Caso seja utilizado um certificado auto-assinado, você tem três opções:

Copiar a CA para a máquina:

```
opensuse:~ # cp ca.cert /usr/share/pki/trust/anchors/
opensuse:~ # update-ca-certificates -v
```

Alterando o arquivo de parâmetros padrões do client ldap **/etc/openldap/ldap.conf** - isto é necessário por conta do PAM e do SSSD:

```
opensuse:~ # echo -e 'TLS_REQCERT\tnever' > /etc/openldap/ldap.conf
opensuse:~ # ldapsearch -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -LLL -ZZ
No such object (32)
```

Ou sempre especificar a variável **LDAPTLS_REQCERT=never** antes do comando:

```
opensuse:~ # LDAPTLS_REQCERT=never ldapsearch -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -LLL -ZZ
No such object (32)
```

#### Popular o OpenLDAP:

Popule o OpenLDAP com os domínios, grupos e usuários iniciais disponíveis no diretório **/vagrant/files/ldap**:

```
ldapadd -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -f base.ldif
ldapadd -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -f users.ldif
ldapadd -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -f groups.ldif
# Este arquivo adiciona um membro a um grupo já existente
ldapadd -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -f groups-users.ldif
```

#### SSSD

Edite o arquivo **/etc/sssd/sssd.conf**:

```
[nss] 
filter_groups = root 
filter_users = root 
reconnection_retries = 3 

[pam] 
reconnection_retries = 3 
offline_credentials_expiration = 3 

[sssd] 
config_file_version = 2 
reconnection_retries = 3 
sbus_timeout = 30 
services = nss, pam 
domains = ldap 
debug_level = 5

[domain/ldap] 
chpass_provider = ldap 
auth_provider = ldap 
ldap_schema = rfc2307 
id_provider = ldap 
enumerate = true 
cache_credentials = true
# Necessário para autoassinados
ldap_tls_reqcert = never
# Opcional para limitar a base de busca
ldap_user_search_base = ou=users,dc=ldap,dc=example,dc=com
Caso seja utilizado um certificado auto-assinado, você tem duas opções:

Copiar a CA para a máquina:

```
cp ca.cert /usr/share/pki/trust/anchors/
update-ca-certificates -v
```

Ignorar o certificado auto-assinado alterando o arquivo de parâmetros padrões do client ldap **/etc/openldap/ldap.conf** - isto é necessário por conta do PAM e do SSSD:

```
[root@centos ~]# echo -e 'TLS_REQCERT\tnever' > /etc/openldap/ldap.conf
```
ldap_uri = ldap://ldap.example.com
```

Verifique se no arquivo **/etc/nsswitch.conf** ao menos passwd, group e shadow utilizem **sss**:

```
passwd: compat sss
group:  compat sss
shadow: compat sss
```

Verifique se o openSUSE consegue encontrar os usuários e grupos através do comando **getent**:

```
getent passwd
getent group
```

Configure o PAM para utilizar SSS e criar a home dos usuários que se logarem:

```
pam-config -a --sss --mkhomedir --mkhomedir-skel=/etc/skel/
```

Verifique se é possível se logar com o usuário:

```
# Local
su - hector.vido
# SSH
ssh hector.vido@127.0.0.1
```

## Debian

Instale os pacotes necessários para a configuração do SSSD:

```
apt-get install sssd libpam-sss libnss-sss vim ldap-utils
```

Caso não utilize DNS:

```
echo '27.11.90.10 ldap.example.com' >> /etc/hosts
```

Verifique se no arquivo **/etc/nsswitch.conf** ao menos passwd, group e shadow utilizem **sss**:

```
passwd: compat sss
group:  compat sss
shadow: compat sss
```

Uma boa configuração do SSSD inicial, com cache e tentativas pode ser a seguinte em **/etc/sssd/sssd.conf**:

```
[nss] 
filter_groups = root 
filter_users = root 
reconnection_retries = 3 

[pam] 
reconnection_retries = 3 
offline_credentials_expiration = 3 

[sssd] 
config_file_version = 2 
reconnection_retries = 3 
sbus_timeout = 30 
services = nss, pam 
domains = ldap 
debug_level = 5

[domain/ldap] 
chpass_provider = ldap 
auth_provider = ldap 
ldap_schema = rfc2307 
id_provider = ldap 
enumerate = true 
cache_credentials = true
# Necessário para autoassinados
ldap_tls_reqcert = never
# Opcional para limitar a base de busca
ldap_user_search_base = ou=users,dc=ldap,dc=example,dc=com

ldap_uri = ldap://ldap.example.com
```

Garanta que o arquivo seja acessado apenas pelo usuário root e reinicie o serviço:

```
chmod 0600 /etc/sssd/sssd.conf
systemctl restart sssd
```

Caso você queira que os usuários tenham um diretório **home**, você pode criar o arquivo */usr/share/pam-configs/mkhomedir* com o seguinte conteúdo:

```
Name: Create home directory during login
Default: yes
Priority: 900
Session-Type: Additional
Session:
        required        pam_mkhomedir.so umask=0022 skel=/etc/skel
```

Reconfigure o PAM para garantir que tudo esteja correto:

```
pam-auth-update
```

Caso seja utilizado um certificado auto-assinado, você tem duas opções:

Copiar a CA para a máquina:

```
vim /usr/local/share/ca-certificates/ldap.crt
update-ca-certificates -v
```

Alterando o arquivo de parâmetros padrões do client ldap **/etc/ldap/ldap.conf** - isto é necessário por conta do PAM e do SSSD:

```
root@debian:~# echo -e 'TLS_REQCERT\tnever' >> /etc/ldap/ldap.conf
```

Verifique se o Debian consegue encontrar os usuários e grupos através do comando **getent**:

```
getent passwd
getent group
```

Tente se logar com o usuário:

```
# Local
su - hector.vido
# SSH - liberar "PasswordAuthentication yes" em /etc/ssh/sshd_config
ssh hector.vido@127.0.0.1
```

## Centos

Instale os pacotes necessários para a configuração do SSSD:

```
yum install -y sssd vim openldap-clients
```

Caso não utilize DNS:

```
echo '27.11.90.10 ldap.example.com' >> /etc/hosts
```

Verifique se no arquivo **/etc/nsswitch.conf** ao menos passwd, group e shadow utilizem **sss**:

```
passwd: compat sss
group:  compat sss
shadow: compat sss
```

Uma boa configuração do SSSD inicial, com cache e tentativas pode ser a seguinte em **/etc/sssd/sssd.conf**:

```
[nss] 
filter_groups = root 
filter_users = root 
reconnection_retries = 3 

[pam] 
reconnection_retries = 3 
offline_credentials_expiration = 3 

[sssd] 
config_file_version = 2 
reconnection_retries = 3 
sbus_timeout = 30 
services = nss, pam 
domains = ldap 
debug_level = 5

[domain/ldap] 
chpass_provider = ldap 
auth_provider = ldap 
ldap_schema = rfc2307 
id_provider = ldap 
enumerate = true 
cache_credentials = true
# Necessário para autoassinados
ldap_tls_reqcert = never
# Opcional para limitar a base de busca
ldap_user_search_base = ou=users,dc=ldap,dc=example,dc=com

ldap_uri = ldap://ldap.example.com
```

Garanta que o arquivo seja acessado apenas pelo usuário root e reinicie o serviço:

```
chmod 0600 /etc/sssd/sssd.conf
systemctl restart sssd
```

Configure o PAM para utilizar SSS e criar a home dos usuários que se logarem:

```
authconfig --enablesssd --enablesssdauth --enablelocauthorize --enablemkhomedir --update
```

Caso seja utilizado um certificado auto-assinado, você tem duas opções:

Copiar a CA para a máquina:

```
vim /usr/share/pki/ca-trust-source/anchors/ldap.pem
update-ca-trust -v
```

Alterando o arquivo de parâmetros padrões do client ldap **/etc/openldap/ldap.conf** - isto é necessário por conta do PAM e do SSSD:

```
[root@centos ~]# echo -e 'TLS_REQCERT\tnever' > /etc/openldap/ldap.conf
```

Verifique se o CentOS consegue encontrar os usuários e grupos através do comando **getent**:

```
getent passwd
getent group
```

Tente se logar com o usuário:

```
# Local
su - hector.vido
# SSH - liberar "PasswordAuthentication yes" em /etc/ssh/sshd_config
ssh hector.vido@127.0.0.1
```
