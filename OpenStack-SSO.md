# Bắt đầu

Keystone v3 bắt đầu hỗ trợ xác thực federated vì vậy tôi thực hiện tìm hiểu cách cài đặt, cấu hình để sử dụng tính năng này. 

Việc cài đặt các thành phần: Keystone, glance, nova, neutron, horizone, cinder,.. vẫn như hướng dẫn tại [đây](https://github.com/congto/OpenStack-Mitaka-Scripts/blob/master/DOCS-OPS-Mitaka/Caidat-OpenStack-Mitaka.md)

Tôi thực hiện tách node database ra thành node riêng

Sau khi cài đặt hoàn thành OpenStack, thì việc xác thực vẫn chỉ sử dụng với user được tạo trên local bằng keystone.

Để sử dụng được Federated thì bắt đầu hiểu về mô hình xác thực SSO (Single Sign-On). Bạn có thể đọc tại phần cơ bản về SSO. Ở đây tôi sẽ hướng dẫn quá trình 
cài đặt và cấu hình

# Cài đặt

Bây giờ tôi bắt đầu dựng một hệ thống Identity Provider sử dụng shibboleth.

## Node database

Chạy Ubuntu server 14.04 64 bit

- Cài đặt database mysql
```sh
wget http://dev.mysql.com/get/mysql-apt-config_0.8.6-1_all.deb
sudo dpkg -i mysql-apt-config_0.8.6-1_all.deb
apt-get update
apt-get install mysql-server -y
==> passwd tan124
```

- Thực hiện cài đặt bảo mật, loại bổ một số thành phần không cần thiết
```sh
mysql_secure_installation
```

- tạo file `vi ~/.my.cnf` với nội dung dưới để sau này lúc login vào mysql không cần mật khẩu, chỉ cần gõ `mysql` là tự động đăng nhập
```sh
[client]
user = root
password = tan124
```

## Node Idp

Node Idp cài OS Ubuntu server 14.04 64 bit. 

Đặt hostname cho node Idp là: idp và chỉnh sửa file `/etc/host` thêm vào
```sh
172.16.68.21    database
172.16.68.22    idp
172.16.68.23    keystone
172.16.68.24    ldap
```

ở đây là hostname và IP của các node. trong đó node `keystone` là node đã cài OpenStack theo guide trên. 

Sau đó cài các gói sau.

Cài đặt java 8

- Cài đặt repo
```sh
add-apt-repository -y ppa:webupd8team/java
apt-get update -y
```

- Chạy 2 lệnh sau để quá trình cài java không hiện ra option hỏi
```sh
echo "oracle-java8-installer shared/accepted-oracle-license-v1-1 select true" | sudo debconf-set-selections
echo "oracle-java8-installer shared/accepted-oracle-license-v1-1 seen true" | debconf-set-selections
```

- Chạy lệnh sau cài Java
```sh
apt-get install oracle-java8-installer -y
```

Tải gói shibboleth để cài đặt

- Tạo thư mục lưu các gói cài đặt
```sh
mkdir -p /dists && cd /dists
```

- Thực hiện tải các gói sau
```sh
wget http://shibboleth.net/downloads/identity-provider/3.2.1/shibboleth-identity-provider-3.2.1.tar.gz
wget https://build.shibboleth.net/nexus/content/repositories/releases/net/shibboleth/idp/idp-jetty-base/9.3.0/idp-jetty-base-9.3.0.tar.gz
wget https://build.shibboleth.net/nexus/content/repositories/thirdparty/org/eclipse/jetty/jetty-distribution/9.3.9.v20160517/jetty-distribution-9.3.9.v20160517.tar.gz
```

- Giải nén các gói vừa download
```sh
tar xf shibboleth-identity-provider-3.2.1.tar.gz
tar xf idp-jetty-base-9.3.0.tar.gz
tar xf jetty-distribution-9.3.9.v20160517.tar.gz
```

- xóa thư mục jetty-base mặc định trong shibboleth
```sh
rm -rdf ./shibboleth-identity-provider-3.2.1/embedded/jetty-base
```

- Thay thư mục jetty-base download lúc đầu vào thay thế
```sh
mv ./jetty-base ./shibboleth-identity-provider-3.2.1/embedded/
```

- Đổi lại tên của thư mục chưa các gói cài
```sh
mv shibboleth-identity-provider-3.2.1 idp && mv ./jetty-distribution-9.3.9.v20160517 ./jetty
```

- export biến môi trường java
```sh
export JAVA_HOME=/usr/lib/jvm/java-8-oracle
export JRE_HOME=/usr/lib/jvm/java-8-oracle/jre
export JETTY_HOME=/dists/jetty
```

Tạo tập tin cấu hình cài đặt

- tạo tập tin `install.properties` với nội dung sau để thiết lập các biến trong quá trình cài đặt shibboleth
```sh
idp.merge.properties=/dists/merge.idp.properties
ldap.merge.properties=/dists/merge.ldap.properties
jetty.merge.properties=/dists/merge.jetty.properties

idp.src.dir=/dists/idp
idp.target.dir=/opt/shibboleth-idp
idp.scope=idp
idp.host.name=idp
idp.sealer.password=tan124
idp.keystore.password=tan124
idp.noprompt=true
idp.jetty.config=true
idp.uri.subject.alt.name=http://idp/idp/shibboleth
```

- Tạo thêm tập tin `merge.idp.properties` với nội dung sau:
```sh
idp.hostname=idp
idp.home=/opt/shibboleth-idp/

idp.entityID=http://idp/idp/shibboleth
idp.status.page.allowedips=0.0.0.0/0
idp.loglevel.idp = DEBUG
idp.encryption.optional = true
idp.scope=idp
idp.host.name=idp
idp.sealer.storePassword=tan124
idp.sealer.keyPassword=tan124
idp.authn.flows=Password|RemoteUserInternal
```

- Tạo thêm tập tin `merge.jetty.properties` với nội dung
```sh
jetty.host=0.0.0.0
jetty.https.port=443
jetty.backchannel.port=9443
jetty.backchannel.keystore.password=tan124
jetty.browser.keystore.password=tan124
jetty.nonhttps.host=0.0.0.0
jetty.nonhttps.port=80
```

- Tạo thêm tập tin `merge.ldap.properties` có nội dung
```sh
idp.authn.LDAP.authenticator                   = bindSearchAuthenticator
idp.authn.LDAP.ldapURL                         = ldap://ldap:389
idp.authn.LDAP.useStartTLS                     = false
idp.authn.LDAP.useSSL                          = false

idp.authn.LDAP.sslConfig                       = jvmTrust
idp.authn.LDAP.returnAttributes                 = cn,mail,description
idp.authn.LDAP.baseDN                           = ou=Users,dc=openstack,dc=com
idp.authn.LDAP.subtreeSearch                    = true
idp.authn.LDAP.userFilter                       = (cn={user})
idp.authn.LDAP.bindDN                           = cn=admin,dc=openstack,dc=com
idp.authn.LDAP.bindDNCredential                 = tan124
idp.authn.LDAP.dnFormat                         = cn=%s,ou=Users,dc=openstack,dc=com

idp.attribute.resolver.LDAP.ldapURL             = %{idp.authn.LDAP.ldapURL}
idp.attribute.resolver.LDAP.connectTimeout      = %{idp.authn.LDAP.connectTimeout:PT3S}
idp.attribute.resolver.LDAP.responseTimeout     = %{idp.authn.LDAP.responseTimeout:PT3S}
idp.attribute.resolver.LDAP.baseDN              = %{idp.authn.LDAP.baseDN:undefined}
idp.attribute.resolver.LDAP.bindDN              = %{idp.authn.LDAP.bindDN:undefined}
idp.attribute.resolver.LDAP.bindDNCredential    = %{idp.authn.LDAP.bindDNCredential:undefined}
idp.attribute.resolver.LDAP.useStartTLS         = false
idp.attribute.resolver.LDAP.trustCertificates   = %{idp.authn.LDAP.trustCertificates:undefined}
idp.attribute.resolver.LDAP.searchFilter        = (cn=$resolutionContext.principal)
idp.attribute.resolver.LDAP.returnAttributes    = cn,mail,description
```

Sau khi thực hiện tạo xong các tập tin cầu hình, thực hiện cài đặt shibboleth.

- Chạy lệnh sau để cài shibboleth
```sh
./idp/bin/install.sh -propertyfile install.properties
```

- Chạy lệnh sau để cài jetty
```sh
./idp/bin/ant.sh -f /opt/shibboleth-idp/bin/ant-jetty.xml -propertyfile install.properties
```

#### Cấu hình shibboleth

- backup lại tập tin `/opt/shibboleth-idp/conf/access-control.xml`
```sh
mv /opt/shibboleth-idp/conf/access-control.xml /opt/shibboleth-idp/conf/access-control.xml.orgi
```

- tạo tập tin `/opt/shibboleth-idp/conf/access-control.xml` mới với nội dung sau
```sh
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:util="http://www.springframework.org/schema/util"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:c="http://www.springframework.org/schema/c"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
                           http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd"

       default-init-method="initialize"
       default-destroy-method="destroy">

    <util:map id="shibboleth.AccessControlPolicies">

        <entry key="AccessByIPAddress">
            <bean parent="shibboleth.IPRangeAccessControl"
                p:allowedRanges="#{ {'0.0.0.0/0' } }" />
        </entry>

    </util:map>

</beans>
```

- Backup lại tập tin `/opt/shibboleth-idp/conf/attribute-filter.xml`
```sh
mv /opt/shibboleth-idp/conf/attribute-filter.xml /opt/shibboleth-idp/conf/attribute-filter.xml.orgi
```

- Tạo tập tin `/opt/shibboleth-idp/conf/attribute-filter.xml` mới có nội dung
```sh
<?xml version="1.0" encoding="UTF-8"?>

<AttributeFilterPolicyGroup id="ShibbolethFilterPolicy"
        xmlns="urn:mace:shibboleth:2.0:afp"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="urn:mace:shibboleth:2.0:afp http://shibboleth.net/schema/idp/shibboleth-afp.xsd">

    <AttributeFilterPolicy id="releaseToAnyone">

        <PolicyRequirementRule xsi:type="ANY" />

        <AttributeRule attributeID="cn">
            <PermitValueRule xsi:type="ANY" />
        </AttributeRule>

        <AttributeRule attributeID="mail">
            <PermitValueRule xsi:type="ANY" />
        </AttributeRule>

    </AttributeFilterPolicy>


</AttributeFilterPolicyGroup>
```

- Backup lại tập tin `/opt/shibboleth-idp/conf/attribute-resolver.xml`
```sh
mv /opt/shibboleth-idp/conf/attribute-resolver.xml /opt/shibboleth-idp/conf/attribute-resolver.xml.orgi
```

- Tạo tập tin `/opt/shibboleth-idp/conf/attribute-resolver.xml` mới với nội dung
```sh
<?xml version="1.0" encoding="UTF-8"?>

<AttributeResolver
         xmlns="urn:mace:shibboleth:2.0:resolver"
         xmlns:resolver="urn:mace:shibboleth:2.0:resolver"
         xmlns:pc="urn:mace:shibboleth:2.0:resolver:pc"
         xmlns:ad="urn:mace:shibboleth:2.0:resolver:ad"
         xmlns:dc="urn:mace:shibboleth:2.0:resolver:dc"
         xmlns:enc="urn:mace:shibboleth:2.0:attribute:encoder"
         xmlns:sec="urn:mace:shibboleth:2.0:security"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="urn:mace:shibboleth:2.0:resolver http://shibboleth.net/schema/idp/shibboleth-attribute-resolver.xsd
                             urn:mace:shibboleth:2.0:resolver:pc http://shibboleth.net/schema/idp/shibboleth-attribute-resolver-pc.xsd
                             urn:mace:shibboleth:2.0:resolver:ad http://shibboleth.net/schema/idp/shibboleth-attribute-resolver-ad.xsd
                             urn:mace:shibboleth:2.0:resolver:dc http://shibboleth.net/schema/idp/shibboleth-attribute-resolver-dc.xsd
                             urn:mace:shibboleth:2.0:attribute:encoder http://shibboleth.net/schema/idp/shibboleth-attribute-encoder.xsd
                             urn:mace:shibboleth:2.0:security http://shibboleth.net/schema/idp/shibboleth-security.xsd">

    <AttributeDefinition id="uid" xsi:type="ad:Simple" sourceAttributeID="uid">
        <Dependency ref="myLDAP" />
        <AttributeEncoder xsi:type="enc:SAML1String" name="urn:mace:dir:attribute-def:uid" encodeType="false" />
        <AttributeEncoder xsi:type="enc:SAML2String" name="urn:oid:0.9.2342.19200300.100.1.1" friendlyName="uid" encodeType="false" />
    </AttributeDefinition>

    <AttributeDefinition id="cn" xsi:type="ad:Simple" sourceAttributeID="cn">
        <Dependency ref="myLDAP" />
        <AttributeEncoder xsi:type="enc:SAML1String" name="urn:mace:dir:attribute-def:cn" encodeType="false" />
        <AttributeEncoder xsi:type="enc:SAML2String" name="urn:oid:2.5.4.3" friendlyName="cn" encodeType="false" />
    </AttributeDefinition>

    <AttributeDefinition id="description" xsi:type="ad:Simple" sourceAttributeID="description">
     <Dependency ref="myLDAP" />
       <AttributeEncoder xsi:type="enc:SAML1String" name="urn:mace:dir:attribute-def:description" encodeType="false" />
       <AttributeEncoder xsi:type="enc:SAML2String" name="urn:oid:2.5.4.13" friendlyName="description" encodeType="false" />
    </AttributeDefinition>

    <AttributeDefinition id="mail" xsi:type="ad:Simple" sourceAttributeID="mail">
        <Dependency ref="myLDAP" />
        <AttributeEncoder xsi:type="enc:SAML1String" name="urn:mace:dir:attribute-def:mail" encodeType="false" />
        <AttributeEncoder xsi:type="enc:SAML2String" name="urn:oid:0.9.2342.19200300.100.1.3" friendlyName="mail" encodeType="false" />
    </AttributeDefinition>

    <DataConnector id="myLDAP" xsi:type="dc:LDAPDirectory"
        ldapURL="%{idp.attribute.resolver.LDAP.ldapURL}"
        baseDN="%{idp.attribute.resolver.LDAP.baseDN}"
        principal="%{idp.attribute.resolver.LDAP.bindDN}"
        principalCredential="%{idp.attribute.resolver.LDAP.bindDNCredential}"
        useStartTLS="%{idp.attribute.resolver.LDAP.useStartTLS:true}">
        <dc:FilterTemplate>
            <![CDATA[
                %{idp.attribute.resolver.LDAP.searchFilter}
            ]]>
        </dc:FilterTemplate>

        <dc:ReturnAttributes>cn,mail</dc:ReturnAttributes>

	      <dc:ConnectionPool
            minPoolSize="%{idp.pool.LDAP.minSize:3}"
            maxPoolSize="%{idp.pool.LDAP.maxSize:10}"
            blockWaitTime="%{idp.pool.LDAP.blockWaitTime:PT3S}"
            validatePeriodically="%{idp.pool.LDAP.validatePeriodically:true}"
            validateTimerPeriod="%{idp.pool.LDAP.validatePeriod:PT5M}"
            expirationTime="%{idp.pool.LDAP.idleTime:PT10M}"
            failFastInitialize="%{idp.pool.LDAP.failFastInitialize:false}" />
    </DataConnector>

</AttributeResolver>
```

- Backup lại tập tin `/opt/shibboleth-idp/conf/relying-party.xml`
```sh
mv /opt/shibboleth-idp/conf/relying-party.xml /opt/shibboleth-idp/conf/relying-party.xml.orgi
```

- Tạo tập tin `/opt/shibboleth-idp/conf/relying-party.xml` có nội dung
```sh
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:util="http://www.springframework.org/schema/util"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:c="http://www.springframework.org/schema/c"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
                           http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd"

       default-init-method="initialize"
       default-destroy-method="destroy">

    <bean id="shibboleth.UnverifiedRelyingParty" parent="RelyingParty">
        <property name="profileConfigurations">
            <list>
              <bean parent="SAML2.SSO" p:encryptAssertions="false" />
              <ref bean="SAML2.ECP" />
            </list>
        </property>
    </bean>

    <bean id="shibboleth.DefaultRelyingParty" parent="RelyingParty">
        <property name="profileConfigurations">
            <list>
                <bean parent="Shibboleth.SSO" p:postAuthenticationFlows="attribute-release" />
                <ref bean="SAML1.AttributeQuery" />
                <ref bean="SAML1.ArtifactResolution" />
                <bean parent="SAML2.SSO" p:postAuthenticationFlows="attribute-release" />
                <ref bean="SAML2.ECP" />
                <ref bean="SAML2.Logout" />
                <ref bean="SAML2.AttributeQuery" />
                <ref bean="SAML2.ArtifactResolution" />
                <ref bean="Liberty.SSOS" />
            </list>
        </property>
    </bean>


    <util:list id="shibboleth.RelyingPartyOverrides">

    </util:list>

</beans>
```

Tiếp tục thay thế, chỉnh sửa tập tin cấu hình cho ECP SAML2

- Tạo tập tin `/opt/shibboleth-idp/webapp/WEB-INF/web.xml` có nội dung sau:
```sh
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd" version="3.0">

    <display-name>Shibboleth Identity Provider</display-name>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>${idp.home}/system/conf/global-system.xml,classpath*:/META-INF/net.shibboleth.idp/config.xml</param-value>
    </context-param>

    <context-param>
        <param-name>contextClass</param-name>
        <param-value>net.shibboleth.ext.spring.context.DelimiterAwareApplicationContext</param-value>
    </context-param>

    <context-param>
        <param-name>contextInitializerClasses</param-name>
        <param-value>net.shibboleth.idp.spring.IdPPropertiesApplicationContextInitializer</param-value>
    </context-param>

    <!-- Spring listener used to load up the configuration -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!-- Filters and filter mappings -->
    <!-- Try and force I18N, probably won't help much. -->
    <filter>
        <filter-name>CharacterEncodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
        <init-param>
            <param-name>forceEncoding</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <!-- Lets us lump repeated Set-Cookie headers into one, something containers rarely support. -->
    <filter>
        <filter-name>CookieBufferingFilter</filter-name>
        <filter-class>net.shibboleth.utilities.java.support.net.CookieBufferingFilter</filter-class>
    </filter>
    <!-- Automates TLS-based propagation of HttpServletRequest/Response into beans. -->
    <filter>
        <filter-name>RequestResponseContextFilter</filter-name>
        <filter-class>net.shibboleth.utilities.java.support.net.RequestResponseContextFilter</filter-class>
    </filter>
    <!-- Manages logging MDC. -->
    <filter>
        <filter-name>SL4JMDCServletFilter</filter-name>
        <filter-class>net.shibboleth.idp.log.SLF4JMDCServletFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>CookieBufferingFilter</filter-name>
        <url-pattern>/profile/Logout</url-pattern>
        <url-pattern>/profile/Shibboleth/SSO</url-pattern>
        <url-pattern>/profile/SAML2/Unsolicited/SSO</url-pattern>
        <url-pattern>/profile/SAML2/Redirect/SSO</url-pattern>
        <url-pattern>/profile/SAML2/POST/SSO</url-pattern>
        <url-pattern>/profile/SAML2/POST-SimpleSign/SSO</url-pattern>
        <url-pattern>/profile/SAML2/Redirect/SLO</url-pattern>
        <url-pattern>/profile/SAML2/POST/SLO</url-pattern>
        <url-pattern>/profile/SAML2/POST-SimpleSign/SLO</url-pattern>
        <url-pattern>/profile/cas/login</url-pattern>
    </filter-mapping>
    <filter-mapping>
        <filter-name>CharacterEncodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    <filter-mapping>
        <filter-name>RequestResponseContextFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    <filter-mapping>
        <filter-name>SL4JMDCServletFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>

    <!-- HTTP headers to every response in order to prevent response caching -->
    <!-- <filter> <filter-name>IdPNoCacheFilter</filter-name> <filter-class>edu.internet2.middleware.shibboleth.idp.util.NoCacheFilter</filter-class>
        </filter> <filter-mapping> <filter-name>IdPNoCacheFilter</filter-name> <url-pattern>/*</url-pattern> </filter-mapping> -->

    <!-- Servlets and servlet mappings -->
    <servlet>
        <servlet-name>idp</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>${idp.home}/system/conf/mvc-beans.xml, ${idp.home}/system/conf/webflow-config.xml</param-value>
        </init-param>
        <init-param>
            <param-name>contextClass</param-name>
            <param-value>net.shibboleth.ext.spring.context.DelimiterAwareApplicationContext</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>idp</servlet-name>
        <url-pattern>/status</url-pattern>
        <url-pattern>/profile/*</url-pattern>
    </servlet-mapping>

    <!-- Servlet protected by container used for RemoteUser authentication -->
    <servlet>
        <servlet-name>RemoteUserAuthHandler</servlet-name>
        <servlet-class>net.shibboleth.idp.authn.impl.RemoteUserAuthServlet</servlet-class>
        <load-on-startup>2</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>RemoteUserAuthHandler</servlet-name>
        <url-pattern>/Authn/RemoteUser</url-pattern>
    </servlet-mapping>

    <!-- Servlet protected by container used for X.509 authentication -->
    <servlet>
        <servlet-name>X509AuthHandler</servlet-name>
        <servlet-class>net.shibboleth.idp.authn.impl.X509AuthServlet</servlet-class>
        <load-on-startup>3</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>X509AuthHandler</servlet-name>
        <url-pattern>/Authn/X509</url-pattern>
    </servlet-mapping>

    <!-- Send request for the EntityID to the SAML metadata echoing JSP. -->
    <servlet>
        <servlet-name>shibboleth_jsp</servlet-name>
        <jsp-file>/WEB-INF/jsp/metadata.jsp</jsp-file>
    </servlet>
    <servlet-mapping>
        <servlet-name>shibboleth_jsp</servlet-name>
        <url-pattern>/shibboleth</url-pattern>
    </servlet-mapping>


<security-constraint>

    <web-resource-collection>
        <url-pattern>/Authn/RemoteUser</url-pattern>
        <url-pattern>/profile/SAML2/SOAP/ECP</url-pattern>
        <http-method>GET</http-method>
        <http-method>POST</http-method>
    </web-resource-collection>

    <auth-constraint>
        <role-name>**</role-name>
    </auth-constraint>

</security-constraint>

<login-config>
    <auth-method>BASIC</auth-method>
    <realm-name>ShibUserPassAuth</realm-name>
</login-config>

</web-app>
```

- Tạo tập tin `/opt/shibboleth-idp/jetty-base/etc/jetty-http.xml` có nội dung
```sh
<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure_9_3.dtd">

<Configure id="Server" class="org.eclipse.jetty.server.Server">

<Call name="addBean">
     <Arg>
       <New class="org.eclipse.jetty.jaas.JAASLoginService">
         <Set name="name">ShibUserPassAuth</Set>
         <Set name="LoginModuleName">ShibUserPassAuth</Set>
       </New>
     </Arg>
</Call>

<Call name="addConnector">
    <Arg>
      <New id="httpConnector" class="org.eclipse.jetty.server.ServerConnector">
        <Arg name="server"><Ref refid="Server" /></Arg>
        <Arg name="acceptors" type="int"><Property name="jetty.http.acceptors" deprecated="http.acceptors" default="-1"/></Arg>
        <Arg name="selectors" type="int"><Property name="jetty.http.selectors" deprecated="http.selectors" default="-1"/></Arg>
        <Arg name="factories">
          <Array type="org.eclipse.jetty.server.ConnectionFactory">
            <Item>
              <New class="org.eclipse.jetty.server.HttpConnectionFactory">
                <Arg name="config"><Ref refid="httpConfig" /></Arg>
                <Arg name="compliance"><Call class="org.eclipse.jetty.http.HttpCompliance" name="valueOf"><Arg><Property name="jetty.http.compliance" default="RFC7230"/></Arg></Call></Arg>
              </New>
            </Item>
          </Array>
        </Arg>
        <Set name="host"><Property name="jetty.nonhttps.host" deprecated="jetty.host" /></Set>
        <Set name="port"><Property name="jetty.nonhttps.port" deprecated="jetty.port" default="8080" /></Set>
        <Set name="idleTimeout"><Property name="jetty.http.idleTimeout" deprecated="http.timeout" default="30000"/></Set>
        <Set name="soLingerTime"><Property name="jetty.http.soLingerTime" deprecated="http.soLingerTime" default="-1"/></Set>
        <Set name="acceptorPriorityDelta"><Property name="jetty.http.acceptorPriorityDelta" deprecated="http.acceptorPriorityDelta" default="0"/></Set>
        <Set name="acceptQueueSize"><Property name="jetty.http.acceptQueueSize" deprecated="http.acceptQueueSize" default="0"/></Set>
      </New>
    </Arg>
</Call>

</Configure>
```

- Tạo thêm tập tin `/opt/shibboleth-idp/jetty-base/etc/login.conf`
```sh
ShibUserPassAuth {
    /*
	com.sun.security.auth.module.Krb5LoginModule required;
	*/

    org.ldaptive.jaas.LdapLoginModule required
      ldapUrl="ldap://ldap:389"
      bindDn="cn=admin,dc=openstack,dc=com"
      bindCredential="tan124"
      baseDn="ou=Users,dc=openstack,dc=com"
      userFilter="(cn={user})";

};
```

- Thêm module `jaas` vào tập tin `/opt/shibboleth-idp/jetty-base/start.ini`
```sh
echo "--module=jaas" >> /opt/shibboleth-idp/jetty-base/start.ini
```

- Chạy lệnh sau để thêm cấu hình vào tập tin `/opt/shibboleth-idp/jetty-base/webapps/idp.xml`
```sh
sed -i '/copyWebInf/a<Get name="securityHandler"><Set name="realmName">ShibUserPassAuth</Set></Get>' /opt/shibboleth-idp/jetty-base/webapps/idp.xml
```

- Enable debbug cho shibboleth
```sh
sed -i "s/\"idp.loglevel.idp\" value=\"INFO\"/\"idp.loglevel.idp\" value=\"DEBUG\"/g" /opt/shibboleth-idp/conf/logback.xml
```

Sau khi cài đặt và cấu hình shibboleth xong, để chạy shibboleth ta chạy lệnh sau
```sh
cd /opt/shibboleth-idp/jetty-base && java -Didp.home=/opt/shibboleth-idp -Djetty.base=/opt/shibboleth-idp/jetty-base -Djetty.logs=/opt/shibboleth-idp/jetty-base/logs -jar /dists/jetty/start.jar &
```

## Node Ldap

Thực hiện cài đặt ldap để làm backend cho shibboleth.

Node ldap chạy OS Ubuntu 14.04 64 bit. có hostname là `ldap`

- Chạy lệnh sau để cài
```sh
apt-get -y install slapd ldap-utils
```

trong quá trình cài đặt có hỏi password, bạn điền `tan124`

- Thực hiện cấu hình ldap
```sh
dpkg-reconfigure slapd
```

Khi chạy lệnh trên, sẽ hiển thị ra bảng để điền thông tin thì tương ứng với từng bảng ta chọn hoặc điền như sau:
```sh
No -> openstack.com -> Openstack -> tan124 -> HDB -> No -> Yes -> No
```

- Tạo tập tin `vi ~/domain.ldif` chứa các thông tin cấu hình:
```sh
dn: ou=Groups,dc=openstack,dc=com
changetype: add
objectclass: top
objectclass: organizationalUnit
ou: Groups

-

dn: ou=Users,dc=openstack,dc=com
changetype: add
objectclass: top
objectclass: organizationalUnit
ou: Users

-

dn: cn=admin,ou=Users,dc=openstack,dc=com
changetype: add
cn: admin
sn: admin
description: Openstack Admin
mail: admin@openstack.com
userPassword:: e01ENX1zS1dVQm9BMWJDNzNNRERnbjVFdlhnPT0=
objectClass: person
objectClass: inetOrgPerson
objectClass: top

-

dn: cn=admin,ou=Groups,dc=openstack,dc=com
changetype: add
cn: admin
member: cn=admin,ou=Users,dc=openstack,dc=com
objectClass: groupOfNames
objectClass: top
owner: cn=admin,ou=Users,dc=openstack,dc=com

-

dn: cn=dm,ou=Users,dc=openstack,dc=com
changetype: add
cn: dm
description: Openstack Principal Engineer
mail: dm@openstack.com
userPassword:: e01ENX1zS1dVQm9BMWJDNzNNRERnbjVFdlhnPT0=
sn: dm
objectClass: person
objectClass: inetOrgPerson
objectClass: top

-

dn: cn=kb,ou=Users,dc=openstack,dc=com
changetype: add
cn: kb
description: Openstack Principal Engineer
mail: kb@openstack.com
userPassword:: e01ENX1zS1dVQm9BMWJDNzNNRERnbjVFdlhnPT0=
sn: kb
objectClass: person
objectClass: inetOrgPerson
objectClass: top

-

dn: cn=enabled,ou=Groups,dc=openstack,dc=com
changetype: add
cn: enabled
member: cn=admin,ou=Users,dc=openstack,dc=com
member: cn=dm,ou=Users,dc=openstack,dc=com
objectClass: groupOfNames
objectClass: top
owner: cn=admin,ou=Users,dc=openstack,dc=com
```

- Chạy lệnh sau để thực hiện import thông tin từ tập tin `domain.ldif` vào DIT
```sh
ldapadd -c -x -w r00tme -h 'ldap' -D 'cn=admin,dc=openstack,dc=com' -f ~/domain.ldif
```

Tới bước này, ta đã thực hiện cài đặt thành công Identity Provider với shibboleth. Sau đây là bước cài đặt vào cấu hình để project keystone, horizone của 
OpenStack sử dụng Identity Provider này làm federated.

## Node keystone

Thực hiện cài đặt theo hướng dẫn tại [đây](https://github.com/congto/OpenStack-Mitaka-Scripts/blob/master/DOCS-OPS-Mitaka/Caidat-OpenStack-Mitaka.md)

Lưu ý việc chỉ định database thì trỏ tới node database, và hostname tôi đặt cho node là `keystone`

Có thể chỉ định về một repo tại VN để tăng tốc độ download gói. tạo tập tin `vi /etc/apt/apt.conf`
```sh
Acquire::http::Proxy "http://123.30.178.220:3142";
```

Thực hiện cài đặt bổ sung các thư viện cần thiết trên node keystone
```sh
apt-get install -y libxml2-dev libapache2-mod-proxy-html libapache2-mod-shib2
```

- Enable module cho apache
```sh
a2enmod proxy proxy_http ssl
```

- Chỉnh sửa lại tập tin `/etc/keystone/keystone.conf` sao cho có các nội dung sau
```sh
[DEFAULT]
admin_token = tan124
log_dir = /var/log/keystone
[assignment]
[auth]
methods = external,password,token,oauth1,saml2
[database]
connection = mysql+pymysql://keystone:tan124abc@database/keystone
[federation]
remote_id_attribute = 'Shib-Identity-Provider'
trusted_dashboard = http://172.16.68.23/horizon/auth/websso/
sso_callback_template = /etc/keystone/sso_callback_template.html
[token]
provider = fernet
[extra_headers]
Distribution = Ubuntu
```

- Tiếp tục chỉnh sửa tập tin `/etc/apache2/sites-available/wsgi-keystone.conf` sao cho có nội dung:
```sh
Listen 5000
Listen 35357

<VirtualHost *:5000>
	WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
	WSGIProcessGroup keystone-public
	WSGIScriptAlias / /usr/bin/keystone-wsgi-public
	WSGIApplicationGroup %{GLOBAL}
	WSGIPassAuthorization On
	WSGIScriptAliasMatch ^(/v3/OS-FEDERATION/identity_providers/.*?/protocols/.*?/auth)$ /usr/local/bin/keystone-wsgi-public/$1
	<IfVersion >= 2.4>
	  ErrorLogFormat "%{cu}t %M"
	</IfVersion>
	ErrorLog /var/log/apache2/keystone.log
	CustomLog /var/log/apache2/keystone_access.log combined

	<Directory /usr/bin>
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

<Location /v3/auth/OS-FEDERATION/websso/saml2>
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
	WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
	WSGIApplicationGroup %{GLOBAL}
	WSGIPassAuthorization On
	<IfVersion >= 2.4>
	  ErrorLogFormat "%{cu}t %M"
	</IfVersion>
	ErrorLog /var/log/apache2/keystone.log
	CustomLog /var/log/apache2/keystone_access.log combined

	<Directory /usr/bin>
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

- Chỉnh sửa tập tin `/etc/shibboleth/shibboleth2.xml` sao cho có nội dung sau
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

- Chỉnh sửa thêm tập tin `/etc/shibboleth/attribute-map.xml`
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

- Phân lại quyền cho thư mục
```sh
chown -R _shibd:_shibd  /etc/shibboleth/* && chown -R _shibd:_shibd  /etc/shibboleth/
```

- Gen key cho shibboleth module
```sh
shib-keygen -y 10
```

- Khởi động lại shibboleth và keystone
```sh
service shibd restart
service apache2 restart
```

- Enable module shibboleth
```sh
a2enmod shib2
```

Tiến hành khởi tạo groups, projects, mappings,... cho federate

- Tạo file `vi ~/admin-openrc` có nội dung
```sh
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=tan124
export OS_AUTH_URL=http://keystone:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

- import biến môi trường
```sh
unset `env | grep OS_ | awk -F '=' '{print $1}'| xargs`
source ~/admin-openrc
```

- Thiết lập
```sh
# -- group: federated_users
# -- project: federation
# -- assingment: role _member_ for federated_users in federation
```

```sh
openstack group create federated_users --domain Default
openstack group create federated_admins --domain Default

openstack project create --domain Default federation

# add admin and member roles to project federation
openstack role create _member_

# có thể role tên là admin đã được tạo trong quá trình cài đặt OpenStack nên chạy lệnh dưới bị lỗi cũng ko sao
openstack role create admin

openstack role add --project federation --group federated_users _member_
openstack role add --project federation --group federated_admins _member_
openstack role add --project federation --group federated_admins admin
```

- Tạo tập tin mapping có tên `default-federation-mapping.json` để ánh xạ người dùng của federate vào người dùng trên local
```sh
[
		{
			"remote": [
				{
					"type": "cn"
				},
				{
					"type": "mail"
				}
			],
			"local": [
				{
					"group": {
						"name": "federated_users",
						"domain": {
							"name": "Default"
						}
					}
				},

				{
					"user": {
						"name":  "{0}",
						"email": "{1}"
					}
				}

			]
		}

]
```

- Chạy tiếp các lệnh sau
```sh
openstack mapping create \
        --rules ~/default-federation-mapping.json \
        ldap-map

openstack identity provider create \
        --remote-id http://idp/idp/shibboleth shibboleth

openstack federation protocol create \
        --identity-provider shibboleth \
        --mapping ldap-map saml2
```


Chỉnh sửa lại tập tin horizon `/etc/openstack-dashboard/local_settings.py` để truy xuất federate qua giao diện web
```sh
WEBSSO_ENABLED = True
WEBSSO_INITIAL_CHOICE = "saml2"
WEBSSO_CHOICES = (
    ("credentials", _("Keystone Credentials")),
    ("saml2", _("Security Assertion Markup Language")),
)
```

- Khởi động lại apache
```sh
service apache2 restart
```

Tới đây, các bước cài đặt và cấu hình đã thực hiện thành công.

# Kiểm thử

- đăng nhập vào đường dẫn `http://172.16.68.23/horizon/` sẽ hiển thị ra màn hình sau

![sso1](/images/sso1.png)

- Chọn SAML2 và nhấn connect sẽ redirect sang màn hình đăng nhập của Idp

![sso2](/images/sso2.png)

- Nhập vào username/password là: admin/r00tme. Có thể sử dụng app `LdapAdmin` để đăng nhập vào node ldap, thiết lập lại mật khẩu

![sso3](/images/sso3.png)

- Sau khi điền thông tin và xác thực thành công, sẽ chuyển giao diện về horizon

![sso4](/images/sso4.png)

# Tham khảo

- [https://xuctarine.blogspot.com/2016/02/keystone-service-provider-with.html](https://xuctarine.blogspot.com/2016/02/keystone-service-provider-with.html)

