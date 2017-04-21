# Nguồn gốc

Do nhu cầu xác thực tập trung, tránh việc người dùng phải ghi nhớ hàng trăm username/password cho hàng trăm dịch vụ. Các công ty bắt đầu triển khai SSO hoặc tích hợp SSO vào 
hệ thống đang triển khai.

Khi bạn gõ "SSO là gì" vào ô tìm kiếm của google thì hiện ra vô số kết quả, đều định nghĩa SSO là gì, làm gì, hoạt động ra sao? Nhưng chả có cái hướng dẫn ra hồn nào để 
bạn dựng được một hệ thống SSO. Vì thực tế SSO có rất nhiều cách triển khai. Đầu tiên bạn phải hiểu cách dựng một hệ thống SSO, các thành phần bên trong, cơ chế giao tiếp, 
xác thực,...

# Cấu trúc

Một hệ thống Web-based Single Sign-On sẽ có 03 phần:
- Principal (typically a user)
- Service Provider (SP)
- Identity Provider (IdP)

Principal ở đây có thể là người dùng hoặc một thực thể, một module nào đó muốn đăng nhập vào hệ thống.

Service Provider tiếp nhận yêu cầu đăng nhập từ Principal, chuyển hướng người dùng sang một phiên đăng nhập do Identity Provider cung cấp, sau khi đăng nhập thành công, IdP 
sẽ gửi lại một identity assertion để SP lưu giữ cho phiên làm việc của người dùng.

Identity Provider có nhiệm vụ xác thực thông tin đăng nhập và sinh ra một identity assertion gửi về cho SP.

Bước đầu tìm hiểu SSO, tôi ứng dụng SSO cho xác thực người dùng trong OpenStack luôn.

# SSO vào OpenStack

Trong phần này, chúng ta cấu hình hệ thống Web-based Single Sign-On gồm OpenStack Keystone, Shibboleth Identity Provider and OpenStack Horizon.

Tôi thực hiện cài đặt một hệ thống thử nghiệm như [link](https://github.com/TrongTan124/docker-keystone-federation)

Mô hình

![Screenshot_from_2017_03_20_11_19_08](/images/Screenshot_from_2017_03_20_11_19_08.png)

Một số lưu ý khi cài đặt
- Chuẩn bị một hệ điều hành có cài sẵn docker và docker-compose theo [link](https://github.com/TrongTan124/ghi-chep-docker/blob/master/tim-hieu-ve-docker.md)
- Sau đó cài git và tải về `git clone https://github.com/TrongTan124/docker-keystone-federation.git`
- Chưa rõ tại sao khi cài keystone, biến `keystone_version: stable/newton` không được đọc trong Dockerfile nên tôi đã chỉ định luôn là `stable/newton` trong Dockerfile
- Chạy lệnh `docker-compose up` và chờ quá trình cài đặt hoàn thành.

Hệ thống này sử dụng toàn bộ là Container. lệnh kiểm tra các Container sau khi cài đặt hoàn thành `docker ps -a`
```sh
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS                                    NAMES
97d7c92365aa        sso_keystone            "/bin/sh -c '/bin/..."   About an hour ago   Up About an hour    0.0.0.0:5000->5000/tcp                   keystone
0224e3619b58        sso_idp                 "java -Didp.home=/..."   About an hour ago   Up About an hour    80/tcp, 443/tcp, 0.0.0.0:443->8443/tcp   idp
e55bdb26ab40        osixia/openldap:1.1.8   "/container/tool/run"    About an hour ago   Up About an hour    389/tcp, 636/tcp                         ldap
79cb3e71b481        mysql                   "docker-entrypoint..."   About an hour ago   Up About an hour    3306/tcp                                 database
b3c02316b7e0        osixia/phpldapadmin     "/container/tool/run"    About an hour ago   Up About an hour    80/tcp, 443/tcp                          phpldapadmin
```

## Recheck config

Sau khi cài đặt xong, tôi kiểm tra các cấu hình khai báo trên từng Container.

### Keystone

Login vào container keystone bằng lệnh `docker exec -it keystone /bin/bash`

Kiểm tra cấu hình keystone `root@keystone:/etc/keystone# cat keystone.conf |egrep -v "^#|^$"`
```sh
[DEFAULT]
use_syslog = False
use_stderr = True
[DEFAULT]
admin_token = eed0f656-f995-4773-8db4-43c045f5787b
max_token_size = 255
[database]
connection = mysql+pymysql://keystone:r00tme@database/keystone
[cache]
enabled=true
backend = dogpile.cache.memcached
backend_argument = url:localhost:11211
[token]
provider = fernet
expiration = 3600
[identity]
driver = sql
[revoke]
driver = sql
[catalog]
driver = sql
[saml2]
remote_id_attribute = 'Shib-Identity-Provider'
[federation]
sso_callback_template = /etc/keystone/sso_callback_template.html
[auth]
methods = external,password,token,saml2
```

Kiểm tra khai báo sso `# cat /etc/keystone/sso_callback_template.html`
```sh
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
  <head>
    <title>Keystone WebSSO redirect</title>
  </head>
  <body>
     <form id="sso" name="sso" action="$host" method="post">
       Please wait...
       <br/>
       <input type="hidden" name="token" id="token" value="$token"/>
       <noscript>
         <input type="submit" name="submit_no_javascript" id="submit_no_javascript"
            value="If your JavaScript is disabled, please click to continue"/>
       </noscript>
     </form>
     <script type="text/javascript">
       window.onload = function() {
         document.forms['sso'].submit();
       }
     </script>
  </body>
</html>
```

Kiểm tra khai báo Listen keystone `root@keystone:/etc/apache2/sites-available# cat keystone.conf`
```sh
Listen 5000
Listen 35357

<VirtualHost *:5000>

    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /usr/local/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    WSGIScriptAliasMatch ^(/v3/OS-FEDERATION/identity_providers/.*?/protocols/.*?/auth)$ /usr/local/bin/keystone-wsgi-public/$1

    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>

    ErrorLog /proc/self/fd/2
    CustomLog /proc/self/fd/1 combined

    <Directory /usr/local/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>

<Location /Shibboleth.sso>
    SetHandler shib
</Location>

<Location /v3/OS-FEDERATION/identity_providers/shibboleth/protocols/saml2/auth>
    ShibRequestSetting requireSession 1
    AuthType shibboleth
    ShibExportAssertion Off
    Require valid-user
    <IfVersion < 2.4>
        ShibRequireSession On
        ShibRequireAll On
   </IfVersion>
</Location>

</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /usr/local/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    LimitRequestBody 114688
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined

    <Directory /usr/local/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>
```

Kiểm tra attribute trong khai báo shibboleth `root@keystone:/etc/shibboleth# cat attribute-map.xml`
```sh
<Attributes xmlns="urn:mace:shibboleth:2.0:attribute-map" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

    <Attribute name="urn:mace:dir:attribute-def:cn" id="cn"/>
    <Attribute name="urn:mace:dir:attribute-def:sn" id="sn"/>
    <Attribute name="urn:mace:dir:attribute-def:mail" id="mail"/>
    <Attribute name="urn:mace:dir:attribute-def:description" id="description"/>
    <Attribute name="urn:mace:dir:attribute-def:l" id="l"/>
    <Attribute name="urn:mace:dir:attribute-def:o" id="o"/>
    <Attribute name="urn:mace:dir:attribute-def:ou" id="ou"/>

    <Attribute name="urn:oid:2.5.4.3" id="cn"/>
    <Attribute name="urn:oid:2.5.4.4" id="sn"/>
    <Attribute name="urn:oid:0.9.2342.19200300.100.1.3" id="mail"/>
    <Attribute name="urn:oid:2.5.4.13" id="description"/>
    <Attribute name="urn:oid:2.5.4.7" id="l"/>
    <Attribute name="urn:oid:2.5.4.10" id="o"/>
    <Attribute name="urn:oid:2.5.4.11" id="ou"/>

</Attributes>
```

kiểm tra khai báo shibbileth.xml `root@keystone:/etc/shibboleth# cat shibboleth2.xml`
```sh
<SPConfig xmlns="urn:mace:shibboleth:2.0:native:sp:config"
    xmlns:conf="urn:mace:shibboleth:2.0:native:sp:config"
    xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
    xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
    xmlns:md="urn:oasis:names:tc:SAML:2.0:metadata"
    clockSkew="180">

  <ApplicationDefaults entityID="http://keystone:5000/shibboleth">

      <Sessions lifetime="28800" timeout="3600" relayState="ss:mem"
                checkAddress="false" handlerSSL="false" cookieProps="http">

          <SSO entityID="http://idp/idp/shibboleth" ECP="true"> SAML2 SAML1 </SSO>
          <Logout>SAML2 Local</Logout>
          <Handler type="MetadataGenerator" Location="/Metadata" signing="false"/>
          <Handler type="Status" Location="/Status" acl="127.0.0.1 ::1"/>
          <Handler type="Session" Location="/Session" showAttributeValues="true"/>
          <Handler type="DiscoveryFeed" Location="/DiscoFeed"/>


      </Sessions>

      <Errors supportContact="root@localhost"
          helpLocation="/about.html"
          styleSheet="/shibboleth-sp/main.css"/>

      <MetadataProvider type="XML" uri="https://idp/idp/shibboleth"
         backingFilePath="/etc/shibboleth/idp.xml"
        reloadInterval="7200"/>

      <AttributeExtractor type="XML" validate="true" reloadChanges="false" path="attribute-map.xml"/>
      <AttributeResolver type="Query" subjectMatch="true"/>
      <AttributeFilter type="XML" validate="true" path="attribute-policy.xml"/>
      <CredentialResolver type="File" key="sp-key.pem" certificate="sp-cert.pem"/>

  </ApplicationDefaults>

  <SecurityPolicyProvider type="XML" validate="true" path="security-policy.xml"/>
  <ProtocolProvider type="XML" validate="true" reloadChanges="false" path="protocols.xml"/>

</SPConfig>
```

### IdP

Login vào container Idp để kiểm tra cấu hình `root@ELKServer2:~# docker exec -it idp /bin/bash`

Kiểm tra khai báo trong `# cat /etc/hosts`
```sh
172.22.0.2	ldap
172.22.0.4	idp
```

Kiểm tra khai báo tomcat server ``

# Tham khảo

- [https://xuctarine.blogspot.com/2016/02/keystone-service-provider-with.html](https://xuctarine.blogspot.com/2016/02/keystone-service-provider-with.html)

